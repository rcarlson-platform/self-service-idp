# P0.3 live test of the workflow hardening (2026-10-24)

Dates: local date 2026-10-24. GitHub timestamps are UTC and show 2026-10-25 for the same evening.

## What this tests

The PR `ci: scope App tokens and permissions and remove shell injection in tenant workflows` passed CI but had never run. This test runs it end to end with a throwaway tenant, `push-test-2`, from `main`.

Changes exercised:

- workflow-level `permissions: contents: read` in `dispatch.yaml` and `decommission.yaml`
- `validate` App token with `permission-contents: read` (dispatch)
- `populate_tenants` App token with `repositories: self-service-idp,${{ inputs.service }}` and `permission-contents: write`
- `delete_tenant_items` App token with `permission-contents: write`
- `${{ }}` values moved into `env:` in both workflows (three `template-injection` fixes)

## Cluster condition

Vault unsealed, Argo CD settled, no tenant before the test. `make smoke` exit 0 before and after.

## Results

| #   | Step                                                       | Result                                                                | Label    |
| --- | ---------------------------------------------------------- | --------------------------------------------------------------------- | -------- |
| 1   | `make smoke` before                                        | exit 0                                                                | Verified |
| 2   | Form run for `push-test-2` from `main`                     | all three jobs passed                                                 | Verified |
| 3   | Actor of the tenant push (activity feed)                   | `rcarlson-platform-idp[bot]`, `push`, 2026-10-25T03:07:22Z, `a1912d8` | Verified |
| 4   | Control: form run again with `push-test-2` while it exists | `validate` failed with `rcarlson-platform/push-test-2 already exists` | Verified |
| 5   | `decommission.yaml` for `push-test-2`                      | all jobs passed                                                       | Verified |
| 6   | Actor of the decommission push (activity feed)             | `rcarlson-platform-idp[bot]`, `push`, 2026-10-25T03:11:29Z, `1a86620` | Verified |
| 7   | `make smoke` after                                         | exit 0                                                                | Verified |

The actor was read from the repository activity feed (`gh api 'repos/rcarlson-platform/self-service-idp/activity?ref=refs/heads/main&per_page=3'`), not from the commit author name, which is set by `git config` in the workflow and proves nothing.

## What this proves

- The narrowed tokens work: the read-only `validate` token, the `populate_tenants` token scoped to two repositories with `contents: write`, and the `delete_tenant_items` token with `contents: write` all ran without HTTP 403 or unexpected-input warnings.
- Both pushes to `main` were made as the GitHub App.
- The `template-injection` fixes did not break either workflow.
- Create and decommission still work end to end.

## What this does not prove

- **The duplicate-name control proves only that the `validate` check still fires.** Tenant repos are public (Observed), so `gh repo view` would probably succeed even with a weaker token (Assumed). The test does not show that the narrow token was required.
- `delete_github_repo` and the package deletion still use `PKG_DELETE_PAT` and an unscoped App token. They are unchanged by this PR.
- The `template-injection` fixes in `decommission.yaml` were written by the repository owner and were not reviewed line by line. The run shows they work; zizmor reports 0 findings for that audit.
- `permission-contents` values were exercised only for `read` (validate) and `write` (the other two). Nothing else was tried.
- The rules `main` is protected by are not in place yet. No ruleset exists, so the App bypass is untested.
- No capacity-ledger reading was taken: the tenant ran for a few minutes and was removed.

## Follow-ups

- Ruleset with the temporary App bypass (R16): a form create and a decommission must still succeed under it.
- Risk rows R29, R30 and the accepted-findings row on 2026-10-30.