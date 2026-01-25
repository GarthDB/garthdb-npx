# OIDC Authentication Failure with Proto Shims

## Test Run Details
- **Workflow Run**: https://github.com/GarthDB/garthdb-npx/actions/runs/21340297101
- **Commit**: `8e40a8e` - "debug: add steps to capture OIDC error details"
- **Date**: January 25, 2026

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
Even though OIDC environment variables (`ACTIONS_ID_TOKEN_REQUEST_URL` and `ACTIONS_ID_TOKEN_REQUEST_TOKEN`) are present in the GitHub Actions environment, npm cannot authenticate when called through proto shims.

### Evidence
1. ✅ OIDC environment variables are present and accessible in the shell
2. ✅ npm 11.6.2 is installed (supports OIDC)
3. ✅ Workflow has `id-token: write` permission
4. ✅ Trusted publisher is configured on npmjs.com
5. ❌ npm authentication fails when called via `/home/runner/.proto/shims/npm`
6. ❌ Error occurs even though environment variables are present

### Conclusion
**Proto shims are blocking OIDC environment variables from reaching npm.** The shim → `proto run` → actual binary process chain loses the OIDC variables along the way, even though other environment variables seem to make it through.

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
1. Test with proto but without shims (using `proto run npm` directly)
2. Compare results with workflow that bypasses proto completely
3. Document findings in the proto issue
