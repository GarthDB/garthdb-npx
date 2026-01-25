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

### Conclusion
**Proto is blocking OIDC environment variables from reaching npm** in multiple ways:
- ❌ Via shims: `npm` → shim → `proto run` → binary (fails)
- ❌ Via `proto run`: `proto run npm --` → binary (also fails)

Both execution paths lose the OIDC variables, suggesting a fundamental issue with proto's environment variable handling, not just the shims.

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

## Related Issue
https://github.com/moonrepo/proto/issues/930

## Next Steps
1. ✅ Test with proto but without shims (using `proto run npm --`) - **FAILED**
2. ⏳ Test with direct binary paths (`~/.proto/bin/npm` or `~/.proto/tools/npm/...`) - **IN PROGRESS**
3. ⏳ Compare results with workflow that bypasses proto completely (`release-no-proto.yml`)
4. ⏳ Document findings in the proto issue

## Test Results Summary

| Method | OIDC Variables Present | Authentication Result | Notes |
|--------|------------------------|----------------------|-------|
| Proto Shims (`npm` via PATH) | ✅ Yes | ❌ FAILED | ENEEDAUTH error |
| Proto Run (`proto run npm --`) | ✅ Yes | ❌ FAILED | ENEEDAUTH error - **Critical finding** |
| Direct Binary Path (found shim) | ✅ Yes | ❌ FAILED | Found shim instead of binary - **Issue with binary discovery** |
| Direct Binary (`~/.proto/tools/npm/*/bin/npm`) | ⏳ Testing | ⏳ Pending | Updated to explicitly prefer bin/ over shims/ |
| Without Proto (`actions/setup-node`) | ⏳ Testing | ⏳ Pending | Expected to work |
