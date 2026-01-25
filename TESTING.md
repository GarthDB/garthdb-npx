# Testing OIDC Publishing with Proto

This repository is set up to test the issue where proto shims prevent npm OIDC environment variables from reaching `npm publish`.

## Issue Reference

https://github.com/moonrepo/proto/issues/930

## Setup

1. **Proto Configuration** (`.prototools`)
   - Node.js: 20.17.0
   - npm: 11.6.2
   - pnpm: 10.17.1

2. **Trusted Publisher** (configured on npmjs.com)
   - Repository: `garthdb/garthdb-npx`
   - Workflow: `release.yml`
   - Environment: (production or leave blank)

3. **GitHub Secrets Required**
   - `NPM_TOKEN`: Not needed for OIDC, but kept for fallback

## Workflows

### `release.yml` - With Proto
- Uses `moonrepo/setup-toolchain@v0`
- Should fail with OIDC due to environment variable issue
- Triggers on push to `main`

### `release-no-proto.yml` - Without Proto
- Uses `actions/setup-node@v4` directly
- Should work with OIDC
- Manual trigger only (`workflow_dispatch`)

## Testing Process

1. Create a changeset:
   ```bash
   npx changeset
   ```

2. Commit and push to trigger `release.yml`:
   ```bash
   git add .
   git commit -m "chore: add changeset for testing"
   git push
   ```

3. Observe the workflow:
   - Watch for OIDC authentication behavior
   - Check if environment variables are passed through proto shims

4. To test without proto, manually trigger `release-no-proto.yml`:
   - Go to Actions → Release (No Proto) → Run workflow

## Expected Results

### With Proto (release.yml)
- ❌ Should fail with `ENEEDAUTH` error
- OIDC environment variables blocked by proto shims

### Without Proto (release-no-proto.yml)
- ✅ Should succeed with OIDC authentication
- Environment variables reach npm directly

## Debugging

To verify environment variables are present in the workflow, add this step:

```yaml
- name: Debug OIDC Environment
  run: |
    echo "ACTIONS_ID_TOKEN_REQUEST_URL: ${{ env.ACTIONS_ID_TOKEN_REQUEST_URL }}"
    echo "ACTIONS_ID_TOKEN_REQUEST_TOKEN present: ${{ env.ACTIONS_ID_TOKEN_REQUEST_TOKEN != '' }}"
```
