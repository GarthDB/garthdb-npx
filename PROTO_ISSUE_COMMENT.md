# Comment for Proto Issue #930

Hi! I've done extensive testing and can confirm this issue, but I also found a working solution. Here's what I discovered:

## TL;DR - Workaround

**Problem**: `proto run npx` uses Node's bundled npm (10.8.2) instead of proto's managed npm (11.6.2), which breaks OIDC support.

**Solution**: Use `proto run npm -- exec` instead of `proto run npx --`

```bash
# ❌ Doesn't work - uses Node's bundled npm 10.x (no OIDC support)
proto run npx -- changeset publish

# ✅ Works - uses proto's managed npm 11.6.2+ (has OIDC support)
proto run npm -- exec changeset publish
```

## Key Findings

Through comprehensive testing, I identified three main issues:

1. **`npx` respects Node's bundled npm, not proto's managed version**
   - When using `proto run npx`, it executes with npm 10.8.2 (bundled with Node 20.17.0)
   - npm 10.8.2 doesn't support OIDC (requires 11.5.1+)
   - This causes `ENEEDAUTH` errors even with correct OIDC configuration

2. **Environment variable propagation through nested processes**
   - OIDC env vars (`ACTIONS_ID_TOKEN_REQUEST_URL`, `ACTIONS_ID_TOKEN_REQUEST_TOKEN`) don't always reach deeply nested npm processes
   - Workaround: Use `garthdb/changesets-action@v1.6.8` which explicitly passes OIDC env vars

3. **Performance impact is minimal**
   - Proto adds only 2-3 seconds overhead (~10% increase)
   - Acceptable trade-off for version management benefits

## Working Reference Implementation

I've created a complete reference implementation with:
- ✅ OIDC publishing working with proto
- ✅ Multiple successful package publishes (patch, minor, major versions)
- ✅ Comprehensive edge case testing
- ✅ Full documentation

**Repository**: https://github.com/GarthDB/garthdb-npx

### Documentation

- **[OIDC_SUCCESS_SUMMARY.md](https://github.com/GarthDB/garthdb-npx/blob/main/OIDC_SUCCESS_SUMMARY.md)** - Complete solution overview
- **[EDGE_CASE_TESTING_RESULTS.md](https://github.com/GarthDB/garthdb-npx/blob/main/EDGE_CASE_TESTING_RESULTS.md)** - All test results and findings
- **[PROTO_RECOMMENDATIONS.md](https://github.com/GarthDB/garthdb-npx/blob/main/PROTO_RECOMMENDATIONS.md)** - Detailed recommendations for proto improvements

### Published Packages

All with OIDC authentication and full provenance attestation:
- `garthdb@1.0.2` - https://www.npmjs.com/package/garthdb/v/1.0.2
- `garthdb@1.1.0` - https://www.npmjs.com/package/garthdb/v/1.1.0
- `garthdb@2.0.0` - https://www.npmjs.com/package/garthdb/v/2.0.0

## Recommendations for Proto

Based on testing, I recommend:

1. **High Priority**: Make `npx` respect proto's managed npm version (not Node's bundled version)
2. **Medium Priority**: Improve OIDC environment variable propagation through nested processes
3. **Documentation**: Add OIDC trusted publishing examples to proto docs

Full recommendations and implementation details are in the [PROTO_RECOMMENDATIONS.md](https://github.com/GarthDB/garthdb-npx/blob/main/PROTO_RECOMMENDATIONS.md) file.

## Example Working Configuration

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches: [main]

permissions:
  contents: write
  pull-requests: write
  id-token: write  # Required for OIDC

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup proto
        uses: moonrepo/setup-toolchain@v0
        with:
          auto-install: true

      - name: Install dependencies
        run: proto run npm -- install

      - name: Build
        run: proto run npm -- run build

      - name: Release
        uses: garthdb/changesets-action@v1.6.8
        with:
          # CRITICAL: Use 'npm exec' NOT 'npx'
          publish: proto run npm -- exec changeset publish
          oidcAuth: true
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## .prototools Configuration

```toml
node = "20.17.0"
npm = "11.6.2"  # Must be 11.5.1+ for OIDC support
```

Hope this helps! Let me know if you need any clarification on the findings or implementation.
