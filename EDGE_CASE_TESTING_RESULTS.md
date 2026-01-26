# Edge Case Testing Results

Complete edge case testing performed on January 26, 2026 to validate OIDC trusted publishing robustness with both proto and standard Node.js setups.

## Tests Performed

### ✅ 1. Dual Workflow Race Condition

**Issue**: Both `release.yml` and `release-with-proto.yml` were triggering on push to main, causing duplicate PRs and race conditions.

**Fix**: Changed `release-with-proto.yml` to `workflow_dispatch` (manual trigger only)

**Result**: SUCCESS
- Only one workflow now runs automatically
- Proto workflow available for testing when needed
- No more duplicate version bump PRs
- Clear separation of concerns

**Commits**: 
- `8842af2` - "refactor: clean up workflows and make proto workflow manual-only"
- `8d5476c` - "refactor: improve proto workflow comments and remove verbose logging"

---

### ✅ 2. Minor Version Bump

**Test**: Create changeset for minor version bump and verify full publish flow

**Result**: SUCCESS
- Changeset created for minor bump
- Workflow created PR bumping 1.0.2 → 1.1.0
- Merged PR automatically triggered publish
- Published `garthdb@1.1.0` with full provenance attestation
- Workflow run: https://github.com/GarthDB/garthdb-npx/actions/runs/21342424833

**Package**: https://www.npmjs.com/package/garthdb/v/1.1.0
**Attestations**: https://registry.npmjs.org/-/npm/v1/attestations/garthdb@1.1.0

**Commit**: `474395d` - "test: add changeset for minor version bump"

---

### ✅ 3. Major Version Bump

**Test**: Create changeset for major version bump and verify breaking change handling

**Result**: SUCCESS
- Changeset created for major bump
- Workflow created PR bumping 1.1.0 → 2.0.0
- Merged PR automatically triggered publish
- Published `garthdb@2.0.0` with full provenance attestation
- Workflow run: https://github.com/GarthDB/garthdb-npx/actions/runs/21342474078

**Package**: https://www.npmjs.com/package/garthdb/v/2.0.0
**Attestations**: https://registry.npmjs.org/-/npm/v1/attestations/garthdb@2.0.0

**Commit**: `a8f4e35` - "test: add changeset for major version bump"

---

### ✅ 4. Error Handling - Missing `id-token` Permission

**Test**: Create workflow without `id-token: write` permission to verify error detection

**Result**: SUCCESS - Clear error detection
- OIDC environment variables correctly absent when permission missing:
  - `ACTIONS_ID_TOKEN_REQUEST_URL`: ❌ NOT SET
  - `ACTIONS_ID_TOKEN_REQUEST_TOKEN`: ❌ NOT SET
- npm publish fails with expected authentication error
- Clear error messages help diagnose misconfiguration
- Workflow run: https://github.com/GarthDB/garthdb-npx/actions/runs/21342507669

**Error Message**:
```
❌ ACTIONS_ID_TOKEN_REQUEST_URL is NOT SET
This means id-token: write permission is missing
```

**Commit**: `27f7e3b` - "test: add workflow to test OIDC error handling"

---

### ✅ 5. Performance Comparison: Proto vs Non-Proto

**Test**: Measure workflow execution time for both proto and non-proto workflows

**Results**:

#### Non-Proto Workflow (`release.yml`)
- Average duration: **23-24 seconds**
- Sample runs:
  - 23s, 23s, 23s, 24s, 27s

#### Proto Workflow (`release-with-proto.yml`)
- Average duration: **26-27 seconds**
- Sample runs:
  - 23s, 26s, 27s, 28s

**Performance Impact**: 
- Proto adds approximately **2-3 seconds** overhead
- Overhead is **minimal and acceptable** (~10% increase)
- Most overhead likely from proto toolchain initialization

**Conclusion**: Proto's performance impact is negligible. The benefits of version management and consistency outweigh the small time cost.

---

## Summary of All Published Versions

| Version | Type | Published | Attestation | Workflow |
|---------|------|-----------|-------------|----------|
| 1.0.2 | patch | 2026-01-26 | ✅ | [Run](https://github.com/GarthDB/garthdb-npx/actions/runs/21342058326) |
| 1.1.0 | minor | 2026-01-26 | ✅ | [Run](https://github.com/GarthDB/garthdb-npx/actions/runs/21342424833) |
| 2.0.0 | major | 2026-01-26 | ✅ | [Run](https://github.com/GarthDB/garthdb-npx/actions/runs/21342474078) |

All versions published with:
- ✅ OIDC authentication (no NPM_TOKEN)
- ✅ SLSA v1 provenance attestation
- ✅ Cryptographic signatures
- ✅ GitHub Actions workflow metadata

---

## Key Findings

### 1. Changesets Version Bumping Works Perfectly

- **Patch**: ✅ 1.0.1 → 1.0.2
- **Minor**: ✅ 1.0.2 → 1.1.0
- **Major**: ✅ 1.1.0 → 2.0.0

Changesets correctly:
- Detects changeset type
- Creates version bump PRs
- Updates CHANGELOG.md
- Publishes on PR merge

### 2. OIDC Works with All Version Types

All version bump types (patch, minor, major) successfully publish with OIDC authentication. No special configuration needed.

### 3. Error Detection is Clear

When OIDC is misconfigured:
- Environment variables are absent
- npm provides clear authentication errors
- Workflow logs show exactly what's missing
- Easy to diagnose and fix

### 4. Proto Performance is Acceptable

- Only 2-3 seconds additional overhead
- Worth the trade-off for version management
- No impact on OIDC functionality
- Both workflows work identically

### 5. Dual Workflow Solution

**Recommendation**: 
- Keep `release.yml` (non-proto) as primary automatic workflow
- Keep `release-with-proto.yml` as manual-trigger testing workflow
- Prevents race conditions
- Allows proto compatibility testing when needed

---

## Recommendations for Other Projects

### Using Changesets with OIDC

1. **Version Bumps**: All semantic version types (patch, minor, major) work perfectly with OIDC
2. **Workflow Structure**: Use changesets-action to automate PR creation and publishing
3. **Permissions**: Always include `id-token: write` in workflow permissions
4. **Error Handling**: Check for OIDC environment variables before attempting publish

### Using Proto with OIDC

1. **npm Version**: Ensure `.prototools` specifies npm 11.6.2+
2. **Command Format**: Use `proto run npm -- exec` NOT `proto run npx --`
3. **Action Fork**: Use `garthdb/changesets-action@v1.6.8` with `oidcAuth: true`
4. **Performance**: Expect 2-3 seconds additional overhead (acceptable)

### Workflow Design

1. **Single Automatic Workflow**: Only one workflow should auto-trigger on push
2. **Test Workflows**: Use `workflow_dispatch` for testing alternative configurations
3. **Clear Naming**: Name workflows clearly (e.g., "Release (Proto Test)")
4. **Comments**: Document critical configuration decisions inline

---

## Files Updated During Testing

- `.github/workflows/release.yml` - Cleaned up, removed debug steps
- `.github/workflows/release-with-proto.yml` - Made manual-only, improved comments
- `.github/workflows/test-oidc-error.yml` - Added for error scenario testing
- `.changeset/test-minor-version.md` - Test changeset (deleted after version bump)
- `.changeset/test-major-version.md` - Test changeset (deleted after version bump)

---

## Next Steps

✅ **All edge cases tested successfully**

Optional future tests:
- Test with scoped packages (e.g., `@username/package`)
- Test with monorepo (multiple packages)
- Test with pnpm/yarn package managers
- Test with GitHub Environments for staged releases

---

## Related Documentation

- [OIDC_SUCCESS_SUMMARY.md](./OIDC_SUCCESS_SUMMARY.md) - Overall solution summary
- [OIDC_ERROR_DETAILS.md](./OIDC_ERROR_DETAILS.md) - Debugging journey and all test runs
- [PROTO_RECOMMENDATIONS.md](./PROTO_RECOMMENDATIONS.md) - Suggestions for proto improvements
- [ADDITIONAL_TESTS.md](./ADDITIONAL_TESTS.md) - Test plan and validation checklist
