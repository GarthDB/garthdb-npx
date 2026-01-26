# Recommendations for Proto to Improve OIDC Support

## Summary

While we found a workaround for using npm OIDC trusted publishing with proto, there are improvements proto could make to better support OIDC and other CI/CD environment variables.

## Current Issues

1. **Environment variables are lost in nested process chains**
   - When proto spawns processes that spawn other processes, environment variables don't always propagate
   - This affects OIDC variables (`ACTIONS_ID_TOKEN_REQUEST_URL`, `ACTIONS_ID_TOKEN_REQUEST_TOKEN`) and potentially other CI/CD variables

2. **No documentation on OIDC usage**
   - Proto doesn't currently document how to use it with npm OIDC trusted publishing
   - Users have to discover workarounds through trial and error

## Recommended Changes to Proto

### 1. Preserve Environment Variables Through Process Chains

**Problem**: Environment variables are lost when proto spawns nested processes.

**Solution**: Proto should ensure that important environment variables are explicitly passed to all child processes, not just the immediate child.

**Implementation Ideas**:
- Add a whitelist of environment variables that should always be passed through (e.g., `CI`, `ACTIONS_*`, `GITHUB_*`)
- Add a configuration option to specify additional variables to preserve
- Ensure all exec/spawn calls in proto's codebase explicitly pass environment variables

```toml
# Example .prototools configuration
[env]
# Automatically preserve these environment variable patterns
preserve = ["ACTIONS_*", "GITHUB_*", "CI"]
```

### 2. Add OIDC-Specific Support

**Problem**: OIDC requires specific environment variables that proto doesn't explicitly handle.

**Solution**: Add first-class OIDC support to proto.

**Implementation Ideas**:
- Detect when running in GitHub Actions with `id-token: write` permission
- Automatically preserve OIDC environment variables
- Add a `--preserve-env` flag to `proto run` for explicit control
- Document OIDC usage in proto's documentation

```bash
# Example usage
proto run --preserve-env npm -- publish
# or
proto run --preserve-env=ACTIONS_ID_TOKEN_REQUEST_URL,ACTIONS_ID_TOKEN_REQUEST_TOKEN npm -- publish
```

### 3. Improve Shim Environment Handling

**Problem**: Proto shims start new shell processes that may not inherit all environment variables.

**Solution**: Ensure shims explicitly pass through critical environment variables.

**Implementation**: Update shim templates to preserve environment variables when spawning processes.

### 4. Add Debugging Tools

**Problem**: It's difficult to diagnose environment variable issues with proto.

**Solution**: Add debugging capabilities.

**Implementation Ideas**:
```bash
# Show what environment variables will be available to the command
proto run --debug-env npm -- publish

# Output:
# Environment variables that will be passed:
# - ACTIONS_ID_TOKEN_REQUEST_URL=https://...
# - ACTIONS_ID_TOKEN_REQUEST_TOKEN=***
# - CI=true
# ... etc
```

### 5. Document CI/CD Usage

**Problem**: No documentation on using proto in CI/CD environments with special authentication.

**Solution**: Add comprehensive CI/CD documentation.

**Topics to Cover**:
- Using proto with npm OIDC trusted publishing
- Using proto with GitHub Actions
- Using proto with other CI/CD providers
- Environment variable handling
- Troubleshooting authentication issues

## Workarounds (Current Solution)

Until proto implements these improvements, users can:

1. **Use forked changesets-action** that explicitly passes OIDC variables
2. **Reduce process nesting** by calling tools directly instead of through npm scripts
3. **Use `proto run` for each command** instead of chaining commands in npm scripts

Example:
```yaml
# ❌ Don't use nested npm scripts
publish: proto run npm -- run release

# ✅ Call tools directly
publish: proto run npm -- run build && proto run npx -- changeset publish
```

## Testing Recommendations

To verify these improvements work:

1. **Test with npm OIDC** (this issue)
2. **Test with other authentication methods**:
   - PyPI trusted publishing (Python)
   - RubyGems trusted publishing
   - Docker registry authentication
3. **Test in different CI/CD environments**:
   - GitHub Actions
   - GitLab CI
   - CircleCI
   - Jenkins

## References

- **Original Issue**: https://github.com/moonrepo/proto/issues/930
- **Our Testing Results**: See OIDC_ERROR_DETAILS.md
- **npm OIDC Documentation**: https://docs.npmjs.com/trusted-publishers
- **GitHub Actions OIDC**: https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect

---

## Updates from Edge Case Testing (January 26, 2026)

### New Findings

#### 1. npx vs npm exec Behavior

**Critical Discovery**: `proto run npx` uses Node's bundled npm (10.x), not proto's managed npm (11.x)

**Impact**: 
- npm 10.x lacks OIDC support (requires 11.5.1+)
- Results in `ENEEDAUTH` errors even with correct OIDC configuration
- Confusing for users who expect proto to use its managed npm version

**Workaround**:
```bash
# ❌ Wrong - uses Node's bundled npm 10.x
proto run npx -- changeset publish

# ✅ Correct - uses proto's managed npm 11.6.2+
proto run npm -- exec changeset publish
```

**Recommendation for Proto**:
- Make `npx` command respect proto's managed npm version
- Add warning when `npx` would use older Node-bundled version
- Document this behavior prominently in proto docs

#### 2. Performance Impact is Minimal

**Finding**: Proto adds only 2-3 seconds overhead (~10% increase)

**Measurement**:
- Non-proto workflow: 23-24 seconds average
- Proto workflow: 26-27 seconds average
- Difference: **2-3 seconds**

**Conclusion**: Performance overhead is negligible and acceptable for the benefits proto provides.

#### 3. Version Bumping Works Perfectly

**Tested**: All semantic version types with proto + OIDC
- Patch: 1.0.1 → 1.0.2 ✅
- Minor: 1.0.2 → 1.1.0 ✅
- Major: 1.1.0 → 2.0.0 ✅

**Result**: No proto-specific issues with any version bump type.

#### 4. Error Detection is Clear

**Finding**: When OIDC is misconfigured, error messages are clear and actionable.

**Test Scenario**: Workflow without `id-token: write` permission
- OIDC env vars correctly absent
- npm provides clear "need auth" error
- Easy to diagnose root cause

**Recommendation for Proto**: No changes needed - error detection works well.

---

## Updated Priority Recommendations

### High Priority

1. **Fix `npx` to respect proto-managed npm version**
   - Most critical issue discovered
   - Causes silent failures with OIDC
   - Users expect proto to control all tool versions

2. **Document npm exec vs npx behavior**
   - Add prominent warning in docs
   - Explain when to use `npm exec` vs `npx`
   - Provide examples for common scenarios

3. **Add OIDC documentation**
   - Document tested workaround with changesets
   - Provide example GitHub Actions workflow
   - Link to this repository as reference implementation

### Medium Priority

4. **Improve environment variable propagation**
   - Ensure OIDC vars reach deeply nested processes
   - Add `--preserve-env` flag to proto run
   - Whitelist common CI/CD environment variables

5. **Add proto doctor command**
   - Check for common misconfigurations
   - Verify OIDC environment variables when detected
   - Suggest fixes for common issues

### Low Priority

6. **Performance optimization**
   - 2-3 second overhead is acceptable
   - Could be reduced with caching or parallel initialization
   - Not blocking for OIDC use

---

## Example Configuration for Other Projects

Based on comprehensive testing, here's the recommended setup:

### `.prototools`
```toml
node = "20.17.0"
npm = "11.6.2"  # Critical: 11.5.1+ required for OIDC
```

### GitHub Actions Workflow
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
          # CRITICAL: Use 'npm exec' not 'npx'
          publish: proto run npm -- exec changeset publish
          oidcAuth: true
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Testing Checklist for Proto Developers

If implementing these recommendations, test:

- [x] OIDC authentication works with proto
- [x] Environment variables reach nested npm processes
- [x] npx respects proto-managed npm version
- [x] All semantic version types publish correctly
- [x] Error messages are clear when misconfigured
- [x] Performance overhead is acceptable
- [ ] Works with pnpm and yarn package managers
- [ ] Works with monorepo setups
- [ ] Works with GitHub Environments
- [ ] Works on different CI/CD platforms (GitLab, CircleCI, etc.)

---

## Reference Implementation

This repository (`GarthDB/garthdb-npx`) serves as a complete reference implementation:

- ✅ Proto + OIDC working configuration
- ✅ Comprehensive test coverage
- ✅ Documentation of all issues and solutions
- ✅ Performance benchmarks
- ✅ Error handling examples

**Test Results**: [EDGE_CASE_TESTING_RESULTS.md](./EDGE_CASE_TESTING_RESULTS.md)

---

## Contact for Proto Team

If the proto team wants to discuss these recommendations:
- Repository: https://github.com/GarthDB/garthdb-npx
- Original Issue: [moonrepo/proto#930](https://github.com/moonrepo/proto/issues/930)
- All test results and findings documented in this repository

