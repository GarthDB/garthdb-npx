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
