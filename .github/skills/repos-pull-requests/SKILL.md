---
name: repos-pull-requests
description: Create, update, retarget, label, auto-complete, and read Azure DevOps pull requests without the silent failures of the PR tools. Use for any change to a PR's title, description, target branch, labels, merge commit message, auto-complete, or comment threads, for reading PR vote history, and before pushing directly to a policy-protected branch.
---

# Pull requests

Several PR write operations **report success while changing nothing**. The rule for every write in this skill: **read the PR back** with `repo_pull_request` (action `get`) and compare the fields you meant to change. The update response is not proof — it can mix new and stale values.

# Tools

Use Azure DevOps MCP Server tools for all interactions with Azure DevOps.

- `repo_pull_request` (action: `get`, `list`): Read a PR (`includeLabels`, `includeWorkItemRefs`) or list PRs.
- `repo_pull_request_write` (action: `create`, `update`, `update_reviewers`, `vote`): Change a PR.
- `repo_pull_request_thread` (action: `list`, `list_comments`): Read comment threads.
- `repo_pull_request_thread_write` (action: `create`, `reply`, `update`, `update_status`): Write comment threads.

Prefer these tools over the `az repos pr` CLI for PR **text**: `az repos pr show` corrupts non-ASCII characters in descriptions (for example, em-dashes come back as U+FFFD), so never write a description read through `az` back to the PR; and `az repos pr update --merge-commit-message` silently truncates the message at its first newline.

# Rules

## 1. Title and description

- The description is limited to **4000 characters**. Longer values reject the whole call — nothing is truncated. Keep the description to what a reviewer needs on screen and put long rationale in commit messages.
- **Never combine `targetRefName` with `title` or `description` in one `update` call.** The retarget is applied, the text changes are dropped, and the response echoes the old text. Retarget in one call, then change title/description in a second call, then read back.
- Retargeting has no CLI path (`az repos pr update` has no target option and `az devops invoke` rejects `PATCH` on pull requests), so use `repo_pull_request_write`.

## 2. Labels

- `labels` on `update` **replaces the whole label set**. To add or remove one label, read the current labels with `repo_pull_request` (action `get`, `includeLabels: true`), modify that list, and pass the complete result — an empty array removes all labels.

## 3. Merge commit message and auto-complete

- The squash/merge commit message (`completionOptions.mergeCommitMessage`) is stored **separately** from the description. Rewriting the description does not update it, and it is what lands on the target branch. When a PR's scope changes, update the merge message too, and keep its first line in sync with the title (it becomes the commit subject).
- `mergeCommitMessage` is only persisted as part of the auto-complete bundle:
  - `mergeCommitMessage` alone is rejected ("At least one field … must be provided").
  - With `autoComplete: false` the call succeeds and **persists nothing**.
  - With `autoComplete: true` it is stored — but `mergeStrategy`, `deleteSourceBranch` (default `false`) and `transitionWorkItems` (default `true`) fall back to their defaults unless passed. Always re-send the PR's intended values, or a squash PR is silently downgraded.
- The whole completion options payload is capped at **4000 encoded characters** (merge message title and body together).
- After arming, read the PR back: `autoCompleteSetBy` **not null** is the only reliable "armed" signal. `completionOptions.triggeredByAutoComplete` can be `true` on a PR that is not armed.
- If you cannot arm auto-complete (for example, the PR must not merge yet), there is no tool path to fix a stale merge message — tell the user to edit it in the **Complete pull request** dialog.

## 4. Votes and approvals

- The reviewers list shows only the **current** vote. Vote history lives in the PR's threads as system comments such as `"<name> voted 10"` (10 approve, 5 approve with suggestions, -5 waiting for author, -10 reject) and `"Vote of <name> was reset: Changes pushed to source branch"`. Read threads to answer "was this ever approved?".
- When the target branch's reviewer policy has **reset votes on push** enabled, any push to the source branch wipes existing approvals. If a PR is approved and only its build is expired, **re-queue the policy build instead of pushing** (see the `pipelines-pr-validation` skill).
- Do not vote, approve, or complete a PR unless the user explicitly asks. With "allow requestors to approve their own changes" enabled and auto-complete armed, the author's own approval can merge the PR immediately.

## 5. Stacked pull requests

When PRs are stacked (a PR targets another PR's source branch) and complete as **squash** merges:

- After the lower PR completes, retarget the child to the base branch **and merge the base branch into the child's source branch**. Retargeting alone leaves a stale merge base and the child re-shows the parent's changes as new.
- Prefer merging the base branch up the stack over rebasing: a rebase force-pushes every branch above it, discards comment anchoring, and resets votes.
- A stacked PR usually gets no validation build (see the `pipelines-pr-validation` skill).

## 6. Pushing directly to a protected branch

- Any **enabled** branch policy makes a direct push fail with `TF402455` ("you must use a pull request to update this branch"). Force-pushing does not help, and `git push --atomic` is not supported by Azure Repos.
- Prefer the per-user **Bypass policies when pushing** permission over disabling policies. Disabling a blocking policy releases every PR with auto-complete armed against that branch — list armed PRs (`autoCompleteSetBy` not null) and cancel their auto-complete first.
- Push the branch first, then tags, so a rejected push never leaves a tag on a commit that is not on the branch.

# Examples

- "Retarget PR 123 to main and update its description"
- "Remove the needs-review label from PR 456"
- "Set auto-complete with squash on PR 789 and fix the merge message"
- "Was PR 321 ever approved?"
