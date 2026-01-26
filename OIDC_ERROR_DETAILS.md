# OIDC Authentication Failure with Proto

## Test Run Details

### Test 1: With Proto Shims
- **Workflow Run**: https://github.com/GarthDB/garthdb-npx/actions/runs/21340297101
- **Commit**: `8e40a8e` - "debug: add steps to capture OIDC error details"
- **Date**: January 25, 2026
- **Method**: Using proto shims (`npm` via PATH → `/home/runner/.proto/shims/npm`)

### Test 2: With Proto Run (Bypassing Shims)
- **Workflow Run**: https://github.com/GarthDB/garthdb-npx/actions/runs/21341477050
- **Commit**: `9d5dd3f` - "fix: use correct proto run syntax with -- separator"
- **Date**: January 25, 2026
- **Method**: Using `proto run npm --` (bypassing shims, but still using proto runtime)
- **Result**: ❌ FAILED - Still cannot authenticate

### Test 3: With Direct Binary Path (Found Shim Instead)
- **Workflow Run**: https://github.com/GarthDB/garthdb-npx/actions/runs/21341523579
- **Commit**: `91559f1` - "test: use direct npm binary paths from proto installation"
- **Date**: January 25, 2026
- **Method**: Using direct path to npm binary from `~/.proto/tools/npm/`
- **Issue**: Found `/home/runner/.proto/tools/npm/11.6.2/shims/npm` (shim) instead of `/home/runner/.proto/tools/npm/11.6.2/bin/npm` (actual binary)
- **Result**: ❌ FAILED - Still cannot authenticate (was using shim)

### Test 4: With Direct Binary Path (Found Actual Binary)
- **Workflow Run**: https://github.com/GarthDB/garthdb-npx/actions/runs/21341546336
- **Commit**: `844292e` - "docs: update error details with Test 3 findings"
- **Date**: January 25, 2026
- **Method**: Using direct path to actual npm binary `/home/runner/.proto/tools/npm/11.6.2/bin/npm`
- **Result**: ❌ FAILED - Binary cannot run standalone (requires proto environment)
- **Error**: `Cannot find module '/home/runner/.proto/tools/node/20.17.0/bin/node_modules/npm/bin/npm-prefix.js'`
- **Finding**: npm binaries installed by proto have dependencies on proto's environment setup and cannot be called directly without proto's runtime context

### Test 5: Without Proto (Control Test)
- **Workflow Run**: https://github.com/GarthDB/garthdb-npx/actions/runs/21341669370
- **Commit**: `293cd9d` - "fix: use correct version tag v1.6.8 for changesets-action fork"
- **Date**: January 25, 2026
- **Method**: Using `actions/setup-node@v4` directly (no proto)
- **Changesets Action**: `garthdb/changesets-action@v1.6.8` with `oidcAuth: true`
- **Result**: ✅ SUCCESS - OIDC authentication working correctly
- **Finding**: Confirmed OIDC works when proto is not involved

### Test 6: With Proto + Forked Changesets Action (Initial Attempt)
- **Workflow Run**: https://github.com/GarthDB/garthdb-npx/actions/runs/21341669373
- **Commit**: `293cd9d` - "fix: use correct version tag v1.6.8 for changesets-action fork"
- **Date**: January 25, 2026
- **Method**: Using `proto run npm -- run release` with `garthdb/changesets-action@v1.6.8`
- **Result**: ❌ FAILED - Still `ENEEDAUTH` error
- **Finding**: Even with the fork that explicitly passes OIDC env vars, nested npm scripts lose the variables

### Test 7: With Proto + Direct Changeset Publish Call (SOLUTION)
- **Workflow Run**: https://github.com/GarthDB/garthdb-npx/actions/runs/21341680402
- **Commit**: `af2c01b` - "test: call changeset publish directly with proto to reduce nesting"
- **Date**: January 25, 2026
- **Method**: Using `proto run npm -- run build && proto run npx -- changeset publish` with `garthdb/changesets-action@v1.6.8`
- **Result**: ✅ SUCCESS - OIDC authentication working correctly with proto!
- **Solution**: Calling `changeset publish` directly instead of through `npm run release` reduces process nesting and allows OIDC variables to propagate

## Environment Configuration

### OIDC Environment Variables Status
✅ **Variables are present in GitHub Actions environment:**
- `ACTIONS_ID_TOKEN_REQUEST_URL`: Set and accessible
- `ACTIONS_ID_TOKEN_REQUEST_TOKEN`: Present (length: 3616 characters)
- Both variables are visible in the shell environment

### npm Configuration
- **npm version**: 11.6.2 (supports OIDC - requires 11.5.1+)
- **npm path**: `/home/runner/.proto/shims/npm` (using proto shims)
- **Node.js version**: 20.17.0
- **Proto version**: 0.54.2

## Error Details

### Authentication Failure
```
npm error code ENEEDAUTH
npm error need auth This command requires you to be logged in to https://registry.npmjs.org
npm error need auth You need to authorize this machine using `npm adduser`
```

### Commands That Failed
1. **`npm whoami`** - Failed with `ENEEDAUTH`
2. **`npm publish`** - Failed with `ENEEDAUTH` when called through changesets

### Full Error Output
```
🦋  error an error occurred while publishing garthdb: ENEEDAUTH This command requires you to be logged in to https://registry.npmjs.org 
🦋  error You need to authorize this machine using `npm adduser`
npm error code ENEEDAUTH
npm error need auth This command requires you to be logged in to https://registry.npmjs.org
npm error need auth You need to authorize this machine using `npm adduser`
```

## Root Cause Analysis

### The Problem
Even though OIDC environment variables (`ACTIONS_ID_TOKEN_REQUEST_URL` and `ACTIONS_ID_TOKEN_REQUEST_TOKEN`) are present in the GitHub Actions environment, npm cannot authenticate when called through proto, regardless of whether shims or `proto run` are used.

### Evidence

#### Test 1: With Proto Shims
1. ✅ OIDC environment variables are present and accessible in the shell
2. ✅ npm 11.6.2 is installed (supports OIDC)
3. ✅ Workflow has `id-token: write` permission
4. ✅ Trusted publisher is configured on npmjs.com
5. ❌ npm authentication fails when called via `/home/runner/.proto/shims/npm`
6. ❌ Error occurs even though environment variables are present

#### Test 2: With Proto Run (Bypassing Shims)
1. ✅ OIDC environment variables are present and accessible in the shell
2. ✅ npm 11.6.2 is installed (supports OIDC)
3. ✅ Using `proto run npm --` bypasses shims
4. ❌ **npm authentication STILL fails** with `ENEEDAUTH` error
5. ❌ `proto run npm -- whoami` fails even though environment variables are present

### Critical Discovery
**The issue is NOT limited to proto shims.** Even when using `proto run npm --` (which bypasses shims), npm still cannot authenticate. This indicates:

1. **Proto's `run` command is also not preserving OIDC environment variables**
2. **The problem is deeper than just the shim mechanism**
3. **Environment variables are lost somewhere in proto's execution chain**, regardless of whether shims or `proto run` are used

### Conclusion (Initial Analysis)
**Proto was blocking OIDC environment variables from reaching npm** in multiple ways:
- ❌ Via shims: `npm` → shim → `proto run` → binary (fails)
- ❌ Via `proto run`: `proto run npm --` → binary (also fails)
- ❌ Via nested npm scripts: `proto run npm -- run release` → `npm run build && changeset publish` → `npm publish` (fails)

The issue was that nested process chains (especially through npm scripts) were losing OIDC environment variables.

### Solution
**✅ RESOLVED** - The solution was to:
1. Use the forked `garthdb/changesets-action@v1.6.8` which explicitly passes OIDC environment variables
2. Reduce process nesting by calling `changeset publish` directly instead of through npm scripts:
   - ❌ `proto run npm -- run release` (nested: npm → npm script → changeset → npm)
   - ✅ `proto run npm -- run build && proto run npx -- changeset publish` (direct: proto → changeset → npm)

This approach ensures OIDC environment variables propagate through proto's process chain successfully.

## Debug Output

### OIDC Environment Variables Check
```
=== OIDC Environment Variables ===
ACTIONS_ID_TOKEN_REQUEST_URL: https://run-actions-2-azure-eastus.actions.githubusercontent.com/144//idtoken/...
ACTIONS_ID_TOKEN_REQUEST_TOKEN present: YES
ACTIONS_ID_TOKEN_REQUEST_TOKEN length: 3616

=== npm Configuration ===
npm version: 11.6.2
npm path: /home/runner/.proto/shims/npm
```

### npm whoami Output
```
=== Checking npm authentication status ===
npm error code ENEEDAUTH
npm error need auth This command requires you to be logged in.
npm error need auth You need to authorize this machine using `npm adduser`
Not authenticated (exit code: 1)
```

### npm publish (dry-run) Output
```
npm warn This command requires you to be logged in to https://registry.npmjs.org/ (dry-run)
```

## Related Issues and Resources

- **Proto Issue**: https://github.com/moonrepo/proto/issues/930
- **Forked Changesets Action**: https://github.com/GarthDB/changesets-action
- **Fork Release (v1.6.8)**: https://github.com/GarthDB/changesets-action/releases/tag/v1.6.8
- **npm Trusted Publishing Docs**: https://docs.npmjs.com/trusted-publishers

## Key Learnings

1. **OIDC environment variables must be explicitly passed** through nested process chains
2. **Proto shims and `proto run` both require explicit environment variable passing** when used with nested commands
3. **Reducing process nesting** (calling `changeset publish` directly instead of through npm scripts) significantly improves OIDC variable propagation
4. **The forked changesets-action** (`garthdb/changesets-action@v1.6.8`) solves this by explicitly passing OIDC variables to the exec() call
5. **Both workflows (with and without proto) now work** with the correct configuration

## Solution Implementation

### Required Changes

1. **Use forked changesets-action**: `garthdb/changesets-action@v1.6.8`
   - This fork explicitly passes OIDC environment variables to the publish command
   - Release notes: https://github.com/GarthDB/changesets-action/releases/tag/v1.6.8

2. **Enable OIDC in workflow**:
   ```yaml
   uses: garthdb/changesets-action@v1.6.8
   with:
     oidcAuth: true
   ```

3. **Reduce process nesting** (for proto workflows):
   ```yaml
   # ❌ Don't use nested npm scripts
   publish: proto run npm -- run release
   
   # ✅ Call changeset publish directly
   publish: proto run npm -- run build && proto run npx -- changeset publish
   ```

### Working Configuration

**Without Proto** (`release.yml`):
```yaml
- name: Create Release Pull Request or Publish
  uses: garthdb/changesets-action@v1.6.8
  with:
    publish: npm run build && npx changeset publish
    oidcAuth: true
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**With Proto** (`release-with-proto.yml`):
```yaml
- name: Create Release Pull Request or Publish
  uses: garthdb/changesets-action@v1.6.8
  with:
    publish: proto run npm -- run build && proto run npx -- changeset publish
    oidcAuth: true
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Next Steps
1. ✅ Test with proto but without shims (using `proto run npm --`) - **FAILED initially, then SUCCEEDED with solution**
2. ✅ Test with direct binary paths (`~/.proto/bin/npm` or `~/.proto/tools/npm/...`) - **COMPLETED**
3. ✅ Compare results with workflow that bypasses proto completely - **COMPLETED - Both work!**
4. ⏳ Document findings in the proto issue (https://github.com/moonrepo/proto/issues/930)

## Test Results Summary

| Method | OIDC Variables Present | Authentication Result | Notes |
|--------|------------------------|----------------------|-------|
| Proto Shims (`npm` via PATH) | ✅ Yes | ❌ FAILED | ENEEDAUTH error |
| Proto Run (`proto run npm --`) | ✅ Yes | ❌ FAILED | ENEEDAUTH error - **Critical finding** |
| Direct Binary Path (found shim) | ✅ Yes | ❌ FAILED | Found shim instead of binary - **Issue with binary discovery** |
| Direct Binary (`~/.proto/tools/npm/*/bin/npm`) | ✅ Yes | ❌ FAILED | Binary found but cannot run standalone - **Requires proto environment** |
| Without Proto (`actions/setup-node`) | ✅ Yes | ✅ SUCCESS | OIDC works correctly - **Confirmed working** |
| Proto + Fork (`proto run npm -- run release`) | ✅ Yes | ❌ FAILED | Nested npm scripts still lose env vars |
| Proto + Fork + Direct Call | ✅ Yes | ✅ SUCCESS | **SOLUTION: Direct changeset publish call** |

## Successful Workflow Runs

### Without Proto
- **Run**: https://github.com/GarthDB/garthdb-npx/actions/runs/21341669370
- **Status**: ✅ Success
- **Action**: `garthdb/changesets-action@v1.6.8` with `oidcAuth: true`

### With Proto (Solution)
- **Run**: https://github.com/GarthDB/garthdb-npx/actions/runs/21341680402
- **Status**: ✅ Success
- **Action**: `garthdb/changesets-action@v1.6.8` with `oidcAuth: true`
- **Command**: `proto run npm -- run build && proto run npx -- changeset publish`

---

## Final Resolution - Actual Package Publication

### Test 8: Repository URL Case Mismatch Discovery
- **Workflow Runs**: 
  - Without Proto: https://github.com/GarthDB/garthdb-npx/actions/runs/21342025707
  - With Proto: https://github.com/GarthDB/garthdb-npx/actions/runs/21342025703
- **Commit**: `b7ba488` - "test: add verbose logging to changeset publish"
- **Date**: January 26, 2026
- **Result**: ❌ FAILED (but revealed critical issues)
- **Error (non-proto)**: `422 Unprocessable Entity - Error verifying sigstore provenance bundle: Failed to validate repository information: package.json: "repository.url" is "git+https://github.com/garthdb/garthdb-npx.git", expected to match "https://github.com/GarthDB/garthdb-npx" from provenance`
- **Error (proto)**: `ENEEDAUTH` - npm 10.8.2 (Node-bundled) was being used instead of npm 11.6.2
- **Key Findings**:
  1. ✅ OIDC authentication **worked** for non-proto workflow! (Token exchange succeeded, provenance statement published)
  2. ❌ Publish failed due to repository URL case mismatch (lowercase `garthdb` vs `GarthDB`)
  3. ❌ Proto workflow was using `npm@10.8.2` (Node-bundled) via `npx`, not `npm@11.6.2` (proto-managed)
  4. npm 10.8.2 does not support OIDC (requires npm 11.5.1+)

### Test 9: Complete Success - Fixed Repository URL + npm Version
- **Workflow Runs**: 
  - Without Proto: https://github.com/GarthDB/garthdb-npx/actions/runs/21342058326 ✅
  - With Proto: https://github.com/GarthDB/garthdb-npx/actions/runs/21342073331 ✅
- **Commits**: 
  - `f6b9e4b` - "fix: correct repository URL case for provenance verification"
  - `124b9ed` - "fix: use proto npm exec instead of npx to ensure npm 11.6.2 is used"
- **Date**: January 26, 2026
- **Result**: ✅ **COMPLETE SUCCESS - PACKAGE PUBLISHED**
- **Fixes Applied**:
  1. **Repository URL Case**: Updated package.json repository URL from `https://github.com/garthdb/garthdb-npx.git` to `https://github.com/GarthDB/garthdb-npx.git` to match exact GitHub repository name
  2. **Proto npm Version**: Changed proto workflow from `proto run npx -- changeset publish` to `proto run npm -- exec changeset publish` to ensure npm 11.6.2 (proto-managed) is used instead of npm 10.8.2 (Node-bundled)
- **Published**: `garthdb@1.0.2` with full provenance attestation and cryptographic signatures
- **Verification**: 
  - Package: https://www.npmjs.com/package/garthdb/v/1.0.2
  - Attestations: https://registry.npmjs.org/-/npm/v1/attestations/garthdb@1.0.2
  - Provenance: SLSA provenance predicate with GitHub Actions workflow information

## Critical Learnings

### Issue 1: Case-Sensitive Repository URL Matching
**Problem**: npm's provenance verification requires exact case matching between package.json `repository.url` and the actual GitHub repository name.

**Error**: `422 Unprocessable Entity - Error verifying sigstore provenance bundle`

**Solution**: Ensure package.json repository URL matches the exact case of the GitHub repository (e.g., `GarthDB` not `garthdb`).

### Issue 2: Proto's npx Uses Node-Bundled npm
**Problem**: When using `proto run npx`, it executes using Node's bundled npm (10.8.2 with Node 20.17.0) instead of proto's managed npm (11.6.2). npm 10.8.2 does not support OIDC trusted publishing.

**Error**: `ENEEDAUTH This command requires you to be logged in`

**Solution**: Use `proto run npm -- exec <command>` instead of `proto run npx -- <command>` to ensure proto's npm version is used.

### Working Configuration

Both workflows now successfully publish with OIDC:

**Without Proto** (`release.yml`):
```yaml
- name: Setup Node.js
  uses: actions/setup-node@v4
  with:
    node-version: '20.17.0'
    # Do NOT set registry-url - it enables token-based auth

- name: Install npm 11.6.2
  run: npm install -g npm@11.6.2

- name: Create Release Pull Request or Publish
  uses: garthdb/changesets-action@v1.6.8
  with:
    publish: npx changeset publish --verbose
    oidcAuth: true
```

**With Proto** (`release-with-proto.yml`):
```yaml
- name: Setup proto toolchain
  uses: moonrepo/setup-toolchain@v0
  with:
    auto-install: true

- name: Create Release Pull Request or Publish
  uses: garthdb/changesets-action@v1.6.8
  with:
    # Use 'npm exec' instead of 'npx' to ensure proto's npm is used
    publish: proto run npm -- exec changeset publish --verbose
    oidcAuth: true
```

## Final Status

✅ **OIDC trusted publishing fully operational**
✅ **Package published with provenance attestation**
✅ **Both proto and non-proto workflows working**
✅ **All security features enabled (signatures, attestations, SLSA provenance)**

