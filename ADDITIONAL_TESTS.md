# Additional Tests Needed

## Summary

While we've successfully resolved the OIDC authentication issue with proto, there are additional tests that should be run to ensure the solution is complete and robust.

## Tests Already Completed ✅

1. ✅ Verify OIDC variables are present in GitHub Actions environment
2. ✅ Test without proto (control test)
3. ✅ Test with proto using shims
4. ✅ Test with proto using `proto run`
5. ✅ Test with proto using direct binary paths
6. ✅ Test with forked changesets-action
7. ✅ Test with reduced process nesting

## Tests Still Needed

### 1. Verify Actual Package Publication

**Status**: ✅ COMPLETED (Test 9)

**Why**: The workflows succeeded, but we didn't verify if the package actually published to npm.

**How to Test**:
```bash
# Create a new changeset
npx changeset

# Example changeset content:
---
"garthdb": patch
---

Test OIDC publishing with proto - verify actual publish

# Commit and push
git add .changeset
git commit -m "test: add changeset to verify actual publish"
git push

# Check npm after workflow completes
npm view garthdb versions
```

**Result**: ✅ `garthdb@1.0.2` published successfully with full provenance attestation.

**Published**: January 26, 2026 at 00:13:30 UTC
- Package: https://www.npmjs.com/package/garthdb/v/1.0.2
- Attestations: https://registry.npmjs.org/-/npm/v1/attestations/garthdb@1.0.2
- SLSA provenance: `https://slsa.dev/provenance/v1`
- Cryptographic signatures: 1

### 2. Verify Provenance Attestation

**Status**: ✅ COMPLETED

**Why**: OIDC should automatically generate provenance attestation. We should verify this is working.

**How to Test**:
1. After successful publish, check package on npmjs.com
2. Look for provenance badge/information
3. Verify attestation details match the workflow

**Expected Result**: Package page should show:
- ✅ Provenance attestation badge
- Source: `github.com/GarthDB/garthdb-npx`
- Workflow: `release.yml`

**Check via CLI**:
```bash
npm view garthdb --json | jq '.dist.attestations'
```

### 3. Test with Real Version Bump

**Status**: ⏳ NOT TESTED

**Why**: We've only tested with version 1.0.1 which may already be published. Need to test full version bump flow.

**Steps**:
1. Create changeset for patch version
2. Merge version bump PR (if created by changesets-action)
3. Verify publish succeeds
4. Check new version on npm

### 4. Test Both Workflows Independently

**Status**: ⏳ PARTIAL - Both workflows ran, but only one should be active

**Why**: Currently both `release.yml` (without proto) and `release-with-proto.yml` trigger on push to main. This could cause conflicts.

**Recommendation**: 
- Keep `release.yml` as the main workflow (without proto is simpler and works)
- Rename `release-with-proto.yml` to `release-proto-test.yml` and change to manual trigger
- Or remove one workflow entirely

**Update workflow**:
```yaml
# release-proto-test.yml
on:
  workflow_dispatch:  # Manual trigger only
```

### 5. Test Error Handling

**Status**: ⏳ NOT TESTED

**Why**: Need to verify what happens when OIDC fails.

**Scenarios to Test**:
1. Missing `id-token: write` permission
2. Wrong workflow filename in trusted publisher config
3. Wrong repository name in trusted publisher config
4. Package not configured for trusted publishing

**Expected Results**: Should get clear error messages indicating the specific issue.

### 6. Test with Different Package Managers

**Status**: ⏳ NOT TESTED

**Why**: The repo uses npm, but proto supports pnpm and yarn. Should verify OIDC works with those too.

**How to Test**:
- Create test branch
- Switch to pnpm
- Configure trusted publisher for pnpm
- Test publish with `proto run pnpm -- publish`

### 7. Test Changeset Version PR Creation

**Status**: ⏳ NOT TESTED

**Why**: Changesets should create a version bump PR when there are changesets. We should verify this works.

**How to Test**:
1. Create a changeset
2. Push to main
3. Verify changesets-action creates a PR titled "chore: version packages"
4. Verify PR contains correct version bumps and changelog entries

**Expected Result**: 
- PR created automatically
- PR contains version bumps in package.json
- PR contains CHANGELOG.md updates

### 8. Test Package Access and Scope

**Status**: ⏳ NOT TESTED

**Why**: The package name is `garthdb` (not scoped). Should verify OIDC works for both scoped and unscoped packages.

**Current**: Unscoped package `garthdb`

**Additional Test**: If you have a scoped package (e.g., `@garthdb/something`), test OIDC with that too.

### 9. Performance and Timing

**Status**: ⏳ NOT TESTED

**Why**: Proto may add overhead. Should measure if there's a significant difference.

**How to Measure**:
- Compare workflow execution time: proto vs non-proto
- Document any performance differences

### 10. Test npm whoami with OIDC

**Status**: ⏳ ATTEMPTED BUT FAILED

**Why**: `npm whoami` failed with ENEEDAUTH even with OIDC. Need to understand if this is expected.

**Question**: Does npm OIDC support `npm whoami`, or does it only work for `npm publish`?

**Research Needed**: Check npm documentation to see if OIDC authentication is supposed to work with all npm commands or just publish.

## Test Priority

### High Priority (Should Test Soon)
1. ✅ Verify actual package publication (Test 1)
2. ✅ Verify provenance attestation (Test 2)
3. ✅ Test with real version bump (Test 3)

### Medium Priority (Should Test Eventually)
4. Test both workflows independently (Test 4)
5. Test changeset version PR creation (Test 7)

### Low Priority (Nice to Have)
6. Test error handling (Test 5)
7. Test with different package managers (Test 6)
8. Test package access and scope (Test 8)
9. Performance and timing (Test 9)
10. Test npm whoami with OIDC (Test 10)

## Next Steps

1. **Create a changeset** to trigger a real publish
2. **Monitor the workflow** to ensure actual publish succeeds
3. **Verify on npm** that the new version appears with provenance
4. **Choose one workflow** as primary (recommend non-proto for simplicity)
5. **Document the final solution** for other users

## Success Criteria

The solution is fully validated when:
- ✅ Workflow completes without errors (DONE)
- ⏳ Package actually publishes to npm
- ⏳ Provenance attestation is generated and visible
- ⏳ Version bump PR is created correctly
- ⏳ Documentation is complete and tested by others
