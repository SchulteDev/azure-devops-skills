---
name: pipelines-pr-validation
description: Find and interpret the validation build of an Azure DevOps pull request, and diagnose a PR that seems to get no build. Use when checking whether a push was validated, which build belongs to a PR, why no build was queued, or which branch policies gate a branch.
---

# Pull request validation builds

The most common wrong conclusion here is **"the trigger is broken"**. An empty build listing has several innocent causes. Work through the checks below before stating that — and never write it into a PR description or comment without proof.

# Tools

Use Azure DevOps MCP Server tools where they exist; the MCP server has **no branch policy tool**, so policy checks use the Azure CLI.

- `repo_pull_request` (action: `get`): Read the PR, including `mergeStatus`, `lastMergeSourceCommit`, `lastMergeTargetCommit`, and `targetRefName`.
- `pipelines_build` (action: `get_status`): Read one build.
- `az repos pr policy list --id <prId>`: Policy evaluations for a PR, including the build policy's `context.buildId`.
- `az repos policy list --repository-id <repoId> --branch <branch>`: Policies configured on a branch.
- `az pipelines runs list`: Filtered build listings (see the `pipelines-build-summary` skill for why listing builds through the MCP server can be expensive).

# Rules

## 1. PR builds run on the merge ref

- A PR's policy build has `sourceBranch = refs/pull/<prId>/merge` — the server-computed merge commit — **never** `refs/heads/<source-branch>`. Filtering by the source branch returns nothing.
  `az pipelines runs list --pipeline-ids <id> --branch refs/pull/<prId>/merge --query "[].{id:id,status:status,result:result,queued:queueTime,version:sourceVersion}"`
- `refs/pull/<prId>/merge` is recomputed when the **PR** updates, not when the target branch moves. A manually queued run against it can test a stale base; compare the run's `sourceVersion` with the PR's current merge commit.

## 2. Find the build for a specific push

- Use `az repos pr policy list --id <prId>` and read the **Build** policy's `context.buildId`. It shows the build as soon as it is queued, and whether it is required.
- Build listings (`az pipelines runs list`, `pipelines_build` list) can lag the queue by **seconds to minutes**. Never conclude from one empty listing that no build was queued.
- To confirm a specific push was validated, compare the build's `sourceVersion` (and `queueTime`) with the pushed commit. A build queued before a follow-up push validated the older tree.
- `az pipelines runs list --top 1` can return a newer run from an unrelated branch — always filter by branch, or watch a known run with `az pipelines runs show --id <runId>`.

## 3. When a PR gets no build at all

Check, in this order:

1. **Merge conflicts.** If `mergeStatus` is `conflicts` (numeric `2`; `3` is succeeded), Azure DevOps cannot compute the merge ref and queues **no** build on push — no error, no red build. Resolve the conflict and push.
2. **No build policy on the target branch.** PR validation normally comes from a **build validation policy on the target branch**, not from the pipeline's `pr:` trigger. A PR targeting a branch without that policy — for example, a stacked PR targeting a feature branch — gets no build. Check with `az repos policy list --repository-id <repoId> --branch <target>`. Say plainly in such a PR that it is not validated until retargeted.
3. **Listing lag or wrong ref** — see rules 1 and 2.

## 4. Branch policies

- Check policies **per branch**. Never infer that a branch is gated because a sibling branch is; the pipeline YAML's `trigger:` list says nothing about policies.
- `az repos policy list --query "[].{name:type.displayName, enabled:isEnabled, blocking:isBlocking}" -o json` — `-o table` silently drops the name column.
- Use the listing to check whether a *specific* policy is enabled. Two listings taken at different times can differ without any policy having changed, so do not diff them to claim a policy was added or removed.
- Reading policy configurations through `az devops invoke --area policy --resource configurations` requires `--api-version 7.1-preview` (plain `7.1` is rejected).
- To refresh an expired build on an approved PR, **re-queue the policy build** from the PR instead of pushing — a push may reset approvals (see the `repos-pull-requests` skill).

# Examples

- "Did my last push to PR 123 get a build?"
- "Why does PR 456 not have a validation build?"
- "Which policies protect the release branch?"
