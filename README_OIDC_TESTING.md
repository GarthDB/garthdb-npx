# OIDC Trusted Publishing with Proto & Changesets - Complete Guide

This repository demonstrates a **fully working** implementation of npm OIDC trusted publishing with both standard Node.js and proto toolchain manager, including comprehensive edge case testing.

## 🎯 What Was Accomplished

### Core Achievement
✅ **Successfully published multiple npm packages** using OIDC authentication (no NPM_TOKEN required)  
✅ **All versions include full provenance attestation** and cryptographic signatures  
✅ **Both proto and non-proto workflows** tested and working  
✅ **Comprehensive edge case testing** completed

### Packages Published
- `garthdb@1.0.2` (patch) - [View on npm](https://www.npmjs.com/package/garthdb/v/1.0.2)
- `garthdb@1.1.0` (minor) - [View on npm](https://www.npmjs.com/package/garthdb/v/1.1.0)
- `garthdb@2.0.0` (major) - [View on npm](https://www.npmjs.com/package/garthdb/v/2.0.0)

All with SLSA v1 provenance attestation and signatures.

## 📚 Documentation

### Quick Start
- **[OIDC_SUCCESS_SUMMARY.md](./OIDC_SUCCESS_SUMMARY.md)** - Start here! Complete solution overview
- **[.github/workflows/release.yml](./.github/workflows/release.yml)** - Production-ready workflow (non-proto)
- **[.github/workflows/release-with-proto.yml](./.github/workflows/release-with-proto.yml)** - Proto testing workflow

### Deep Dives
- **[EDGE_CASE_TESTING_RESULTS.md](./EDGE_CASE_TESTING_RESULTS.md)** - All edge cases tested (NEW!)
- **[OIDC_ERROR_DETAILS.md](./OIDC_ERROR_DETAILS.md)** - Complete debugging journey (all 9 test runs)
- **[PROTO_RECOMMENDATIONS.md](./PROTO_RECOMMENDATIONS.md)** - Recommendations for proto improvements
- **[ADDITIONAL_TESTS.md](./ADDITIONAL_TESTS.md)** - Test plan and validation checklist
- **[TESTING.md](./TESTING.md)** - OIDC setup instructions

## 🔑 Key Issues Resolved

### Issue 1: Repository URL Case Sensitivity
**Problem**: npm provenance verification requires exact case matching  
**Solution**: Update package.json to match GitHub repository case exactly  
**Details**: [OIDC_ERROR_DETAILS.md#test-8](./OIDC_ERROR_DETAILS.md#test-8-repository-url-case-mismatch-discovery)

### Issue 2: Proto's npx Uses Wrong npm Version
**Problem**: `proto run npx` uses Node's bundled npm 10.x (no OIDC support) instead of proto's npm 11.x  
**Solution**: Use `proto run npm -- exec` instead of `proto run npx --`  
**Details**: [OIDC_ERROR_DETAILS.md#test-8](./OIDC_ERROR_DETAILS.md#test-8-repository-url-case-mismatch-discovery)

### Issue 3: Environment Variable Propagation
**Problem**: OIDC env vars don't reach deeply nested npm processes through proto  
**Solution**: Use `garthdb/changesets-action@v1.6.8` with `oidcAuth: true`  
**Details**: [OIDC_ERROR_DETAILS.md#test-6-7](./OIDC_ERROR_DETAILS.md#test-6-with-proto--forked-changesets-action-initial-attempt)

### Issue 4: Dual Workflow Race Condition
**Problem**: Both workflows running simultaneously caused conflicts  
**Solution**: Made proto workflow manual-trigger only  
**Details**: [EDGE_CASE_TESTING_RESULTS.md#1-dual-workflow-race-condition](./EDGE_CASE_TESTING_RESULTS.md#-1-dual-workflow-race-condition)

## 🧪 Edge Cases Tested

| Test | Status | Result |
|------|--------|--------|
| Minor version bump (1.0.2 → 1.1.0) | ✅ | Published successfully |
| Major version bump (1.1.0 → 2.0.0) | ✅ | Published successfully |
| Missing id-token permission | ✅ | Clear error detection |
| Proto vs non-proto performance | ✅ | 2-3s difference (acceptable) |
| Dual workflow race condition | ✅ | Fixed with manual triggers |

**Full Results**: [EDGE_CASE_TESTING_RESULTS.md](./EDGE_CASE_TESTING_RESULTS.md)

## ⚡ Performance

- **Non-proto workflow**: 23-24 seconds average
- **Proto workflow**: 26-27 seconds average
- **Overhead**: 2-3 seconds (~10% increase)
- **Verdict**: Negligible impact, proto benefits outweigh cost

## 🚀 Quick Setup Guide

### For Standard Node.js (Recommended)

```yaml
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

      - uses: actions/setup-node@v4
        with:
          node-version: '20.17.0'
          # Do NOT set registry-url

      - name: Install npm 11.6.2+
        run: npm install -g npm@11.6.2

      - name: Install dependencies
        run: npm install

      - name: Build
        run: npm run build

      - name: Release
        uses: garthdb/changesets-action@v1.6.8
        with:
          publish: npx changeset publish
          oidcAuth: true
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### For Proto Toolchain Manager

```yaml
name: Release (Proto)

on:
  workflow_dispatch:  # Manual trigger

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

### Required: `.prototools` (for proto)

```toml
node = "20.17.0"
npm = "11.6.2"  # Must be 11.5.1+ for OIDC
```

### Required: npm Trusted Publisher Configuration

1. Go to https://www.npmjs.com/package/YOUR_PACKAGE/access
2. Add Trusted Publisher:
   - Provider: GitHub Actions
   - Organization/User: YOUR_GITHUB_USER
   - Repository: YOUR_REPO_NAME
   - Workflow: `release.yml`
   - Environment: (leave blank or `production`)

## 📊 Test Coverage

### Functional Tests ✅
- [x] Patch version bump (1.0.1 → 1.0.2)
- [x] Minor version bump (1.0.2 → 1.1.0)
- [x] Major version bump (1.1.0 → 2.0.0)
- [x] OIDC authentication without NPM_TOKEN
- [x] Provenance attestation generation
- [x] Cryptographic signature creation
- [x] Proto toolchain integration
- [x] Changesets PR creation
- [x] Automatic publishing on PR merge

### Edge Cases ✅
- [x] Missing id-token permission (error handling)
- [x] Repository URL case mismatch
- [x] Proto npx vs npm exec behavior
- [x] Dual workflow race conditions
- [x] Performance comparison
- [x] Environment variable propagation

### Scenarios Tested ✅
- [x] First-time OIDC setup
- [x] Multiple consecutive releases
- [x] All semantic version types
- [x] Error detection and diagnosis
- [x] Workflow optimization

## 🎓 Key Learnings

### For Proto Users
1. **Always use `npm exec` not `npx`** when using proto
2. **Specify npm 11.6.2+** in `.prototools` for OIDC support
3. **Use forked changesets-action** with `oidcAuth: true`
4. **Performance overhead is minimal** (2-3 seconds)

### For Changesets Users
1. **Version bump PRs work perfectly** with OIDC
2. **All semver types supported** (patch, minor, major)
3. **Forked action required** for proto compatibility
4. **No changes needed** for standard Node.js setup

### For npm OIDC Users
1. **Case sensitivity matters** for repository URLs
2. **npm 11.5.1+ required** (10.x won't work)
3. **id-token: write required** in workflow permissions
4. **Error messages are clear** when misconfigured
5. **No NPM_TOKEN needed** (completely token-free)

## 🔗 Related Resources

- [npm OIDC Documentation](https://docs.npmjs.com/generating-provenance-statements)
- [GitHub Actions OIDC](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- [Changesets Action (Fork)](https://github.com/garthdb/changesets-action)
- [Proto Issue #930](https://github.com/moonrepo/proto/issues/930)
- [Proto Documentation](https://moonrepo.dev/proto)

## 📞 Questions or Issues?

This repository serves as a complete reference implementation. If you're implementing OIDC + proto:

1. **Read**: [OIDC_SUCCESS_SUMMARY.md](./OIDC_SUCCESS_SUMMARY.md)
2. **Copy**: Workflow files from `.github/workflows/`
3. **Reference**: [EDGE_CASE_TESTING_RESULTS.md](./EDGE_CASE_TESTING_RESULTS.md) for edge cases
4. **Debug**: [OIDC_ERROR_DETAILS.md](./OIDC_ERROR_DETAILS.md) for troubleshooting

## ✨ Status

**All testing complete!** ✅

This repository demonstrates a production-ready OIDC publishing setup with:
- ✅ Comprehensive documentation
- ✅ Edge case coverage
- ✅ Performance benchmarks
- ✅ Multiple working examples
- ✅ Clear recommendations for improvements

**Published packages**: 3 versions (1.0.2, 1.1.0, 2.0.0)  
**Test workflows**: 20+ successful runs  
**Documentation**: 7 comprehensive guides  
**Issues resolved**: 4 critical bugs fixed
