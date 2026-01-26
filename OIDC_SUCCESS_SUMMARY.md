# OIDC Trusted Publishing - Success Summary

## ✅ Achievement

Successfully configured and tested npm OIDC trusted publishing for the `garthdb` package with GitHub Actions, supporting both standard Node.js setup and `proto` toolchain manager workflows.

**Published**: [garthdb@1.0.2](https://www.npmjs.com/package/garthdb/v/1.0.2)  
**Attestations**: [View Provenance](https://registry.npmjs.org/-/npm/v1/attestations/garthdb@1.0.2)

## 🔧 Final Configuration

### Both Workflows Successfully Publish

1. **Without Proto** ([`release.yml`](https://github.com/GarthDB/garthdb-npx/blob/main/.github/workflows/release.yml))
   - Uses `actions/setup-node@v4` with Node 20.17.0
   - Installs npm 11.6.2 globally
   - No `registry-url` setting (avoids token-based auth conflict)

2. **With Proto** ([`release-with-proto.yml`](https://github.com/GarthDB/garthdb-npx/blob/main/.github/workflows/release-with-proto.yml))
   - Uses `moonrepo/setup-toolchain@v0`
   - Proto manages Node 20.17.0 + npm 11.6.2
   - Uses `proto run npm -- exec` instead of `proto run npx --`

Both workflows use `garthdb/changesets-action@v1.6.8` with `oidcAuth: true` for environment variable propagation.

## 🐛 Issues Discovered & Resolved

### Issue 1: Case-Sensitive Repository URL
**Problem**: npm's provenance verification requires exact case matching.

**Error**:
```
422 Unprocessable Entity - Error verifying sigstore provenance bundle:
Failed to validate repository information: 
package.json: "repository.url" is "git+https://github.com/garthdb/garthdb-npx.git",
expected to match "https://github.com/GarthDB/garthdb-npx" from provenance
```

**Solution**: Updated package.json repository URL from `garthdb` (lowercase) to `GarthDB` (correct case).

**Commit**: `f6b9e4b` - "fix: correct repository URL case for provenance verification"

### Issue 2: Proto's `npx` Uses Wrong npm Version
**Problem**: `proto run npx` executes using Node's bundled npm 10.8.2 instead of proto-managed npm 11.6.2.

**Why It Matters**: npm 10.8.2 doesn't support OIDC (requires 11.5.1+)

**Error**:
```
ENEEDAUTH This command requires you to be logged in
npm info using npm@10.8.2  # ❌ Wrong version!
```

**Solution**: Changed from `proto run npx -- changeset publish` to `proto run npm -- exec changeset publish`.

**Commit**: `124b9ed` - "fix: use proto npm exec instead of npx to ensure npm 11.6.2 is used"

### Issue 3: Proto Environment Variable Propagation
**Problem**: OIDC environment variables weren't reaching nested npm processes through proto's shims or standard invocation.

**Solution**: Used forked `garthdb/changesets-action@v1.6.8` which explicitly passes OIDC env vars to child processes.

## 📋 Requirements for OIDC Publishing

### npm Configuration
- ✅ npm version 11.5.1 or higher
- ✅ No `.npmrc` with token-based authentication
- ✅ No `registry-url` set in `actions/setup-node` (conflicts with OIDC)

### GitHub Actions Workflow
- ✅ Workflow file named exactly as configured in npm (e.g., `release.yml`)
- ✅ Permission: `id-token: write`
- ✅ Environment variables: `ACTIONS_ID_TOKEN_REQUEST_URL` and `ACTIONS_ID_TOKEN_REQUEST_TOKEN` (automatically set)

### npm Package Configuration  
- ✅ [Trusted publisher configured on npmjs.com](https://www.npmjs.com/package/garthdb/access)
- ✅ Repository URL in package.json must match exact case of GitHub repo
- ✅ Package must be public or have appropriate access

### Proto-Specific Requirements
- ✅ Use `proto run npm -- exec` instead of `proto run npx --`
- ✅ `.prototools` file specifies npm 11.6.2+
- ✅ Use action fork that handles environment variable propagation

## 🧪 Testing Results

| Test | Method | npm Version | OIDC Auth | Result |
|------|--------|-------------|-----------|--------|
| 1-4 | Various proto methods | 11.6.2 | ❌ | Failed - env var propagation |
| 5 | No proto (control) | 11.6.2 | ✅ | Success - OIDC works |
| 6-7 | Proto + forked action | 11.6.2 | ⚠️ | Partial - nested script issues |
| 8 | Verbose logging | Mixed | ⚠️ | Revealed both issues |
| 9 | **Final config** | **11.6.2** | ✅ | **Published successfully** |

## 📚 Documentation

- **Error Details**: [OIDC_ERROR_DETAILS.md](./OIDC_ERROR_DETAILS.md) - Complete test history
- **Proto Recommendations**: [PROTO_RECOMMENDATIONS.md](./PROTO_RECOMMENDATIONS.md) - Suggestions for proto improvements
- **Additional Tests**: [ADDITIONAL_TESTS.md](./ADDITIONAL_TESTS.md) - Future validation tasks
- **Testing Guide**: [TESTING.md](./TESTING.md) - OIDC setup and testing instructions

## 🔗 Successful Workflow Runs

### Production Publish (1.0.2)
- **Without Proto**: https://github.com/GarthDB/garthdb-npx/actions/runs/21342058326 ✅
- **With Proto**: https://github.com/GarthDB/garthdb-npx/actions/runs/21342073331 ✅

### Initial Success (Testing)
- **Without Proto**: https://github.com/GarthDB/garthdb-npx/actions/runs/21341669370 ✅
- **With Proto**: https://github.com/GarthDB/garthdb-npx/actions/runs/21341680402 ✅

## 🎯 Key Takeaways

1. **OIDC with Proto is Possible**: Requires careful attention to npm version and environment variable propagation
2. **Case Sensitivity Matters**: Repository URLs must match exact GitHub repository name case
3. **npx ≠ npm exec**: When using proto, `proto run npm -- exec` is required to use the correct npm version
4. **Forked Action Required**: Standard changesets-action doesn't handle OIDC env vars well with proto; use `garthdb/changesets-action@v1.6.8`
5. **No Tokens Needed**: OIDC eliminates the need for `NPM_TOKEN` secrets

## ✅ Verification

To verify the published package:

```bash
# Check version
npm view garthdb@1.0.2

# Check attestations
npm view garthdb@1.0.2 dist.attestations

# Install and test
npx garthdb
```

## 🚀 Next Steps

Completed:
- ✅ OIDC configuration
- ✅ Proto integration
- ✅ Package publication with provenance
- ✅ Documentation

Recommended:
- [ ] Test with actual version bump (not just patch)
- [ ] Verify provenance on npmjs.com package page
- [ ] Monitor for any npm/proto version updates
- [ ] Share findings with proto team ([moonrepo/proto#930](https://github.com/moonrepo/proto/issues/930))

## 📞 Related Resources

- [npm OIDC Documentation](https://docs.npmjs.com/generating-provenance-statements)
- [GitHub Actions OIDC](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- [Changesets Action Fork](https://github.com/garthdb/changesets-action)
- [Proto Issue #930](https://github.com/moonrepo/proto/issues/930)
