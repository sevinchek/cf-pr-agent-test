# cf-pr-agent-test — Central PR-Agent hub (PoC)

Reusable GitHub Actions workflow that runs [Qodo PR-Agent](https://github.com/qodo-ai/pr-agent)
(self-hosted action) as an AI reviewer for multiple CF projects from **one place**.

## Architecture (Option A: reusable workflow)

```
InsuranceBack  ─┐
InsuranceFront ─┤   uses: cf-pr-agent-test/.github/workflows/pr-review.yml@main
ResidentsFront ─┼──────────────────────────────────────────►  runs in CALLER's context
Web            ─┘                                              PR-Agent comments on caller's PR
```

- The workflow lives **here** (single source of truth).
- Each project has a tiny **caller** workflow that `uses:` this one.
- It executes in the **caller repo's context**, so the automatic `GITHUB_TOKEN` already
  has access to that repo's PR — **no cross-repo PAT needed**.

## Add a project (caller snippet)

In each project repo, `.github/workflows/pr-agent.yml`:

```yaml
name: PR Agent
on:
  pull_request:
    types: [opened, reopened, ready_for_review, review_requested]
  issue_comment:
concurrency:
  group: pr-agent-${{ github.event.pull_request.number || github.event.issue.number }}
  cancel-in-progress: true
jobs:
  review:
    uses: sevinchek/cf-pr-agent-test/.github/workflows/pr-review.yml@main
    with:
      project: insurance-back   # <- per project
    secrets: inherit            # <- passes OPENAI_KEY
```

## Required setup

1. **Secret `OPENAI_KEY`** in each caller repo (or an org secret shared to all repos).
2. **Reusable-workflow access** (private repos only): this hub must allow other repos of
   the owner to consume it:
   ```
   gh api -X PUT repos/sevinchek/cf-pr-agent-test/actions/permissions/access \
     -f access_level=user
   ```
   (Use `organization` if the hub lives in an org.)

## Per-project reviewer

Two layers, combined:
- **Central default** (here): `pr_reviewer.extra_instructions` is parameterized by `inputs.project`.
- **Per-repo override**: commit a `.pr_agent.toml` in each project (Ruby/Rails rules for
  InsuranceBack & Web, TS/Next for the fronts). The repo-local file wins.

## Triggers & re-review (design notes)

| Want | How |
|---|---|
| Auto review on open / ready / re-request | default `pr_actions` (includes `review_requested`) |
| Re-run on demand | comment `/review` (needs `issue_comment` trigger) |
| Auto review on push | add `"synchronize"` to `github_action_config.pr_actions` + concurrency debounce |
| Same comment thread on re-review | `pr_reviewer.persistent_comment: true` (default) |

> ⚠️ **Checks:** the open-source action posts **comments only**, not GitHub status checks.
> To gate merge on the review score, add a custom step using the Checks API, or use the
> paid Qodo Merge platform.
>
> ⚠️ **Label trigger** is not native (feature request
> [#1716](https://github.com/qodo-ai/pr-agent/issues/1716)). Prefer the `/review` comment.

Sources:
[GitHub events](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows)
· [PR-Agent review tool](https://github.com/qodo-ai/pr-agent/blob/main/docs/docs/tools/review.md)
· [PR-Agent GH Action (DeepWiki)](https://deepwiki.com/qodo-ai/pr-agent/5.2-cli-and-github-actions)
