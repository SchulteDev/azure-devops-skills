---
name: pipelines-run-and-validate
description: Queue an Azure DevOps pipeline run on a specific branch or ref, pass template parameters, cancel or retry runs, and validate pipeline YAML without running it (previewRun). Use when running a pipeline, testing pipeline changes on a branch, or checking how YAML templates expand.
---

# Run pipelines and validate YAML

Queuing a pipeline is a write operation. Only queue a run when the user asks for it, and confirm what will run (definition, ref, parameters) first. Validating YAML with `previewRun` queues nothing.

# Tools

Use Azure DevOps MCP Server tools for all interactions with Azure DevOps.

- `pipelines_definition` (action: `list`): Find the definition ID.
- `pipelines_write` (action: `run_pipeline`): Queue a run, or expand YAML with `previewRun: true`.
- `pipelines_write` (action: `update_build_stage`): Cancel, retry, or run a stage of an in-progress build.
- `pipelines_build` (action: `get_status`): Check a queued run.

# Rules

## 1. Choose the ref explicitly

- `run_pipeline` has no branch argument. Pass the ref through resources:
  `resources: { repositories: { self: { refName: "refs/heads/<branch>" } } }`
  Without it, the run uses the definition's **default branch** — including that branch's YAML, not the YAML you changed.
- For a PR's merge commit use `refName: "refs/pull/<prId>/merge"` (see the `pipelines-pr-validation` skill for why that ref can be stale).
- After queuing, read `resources.repositories.self.refName` and `version` in the response and confirm they are what you intended before reporting the run.

## 2. Template parameters

- `templateParameters` values must be **strings**, also for `type: boolean` parameters: `{ "fullBuild": "true" }`, not `true`.
- Parameters are validated against the parameters declared by the pipeline definition's YAML on that ref.
- Queuing again from a build's results page in the web UI (**Run new**) inherits that run's parameter values. Queue through the tool when the parameters must be defaults.

## 3. Validate YAML without running it

- `run_pipeline` with `previewRun: true` returns the fully expanded YAML (`finalYaml`) and creates **no run**. It resolves `${{ }}` expressions, template includes (also from other repositories), conditions, and `dependsOn`. It is the only pre-merge check pipeline YAML has.
- Pass the branch through `resources` as in rule 1, so templates resolve from that branch.
- `yamlOverride` (only valid with `previewRun`) replaces the **root** YAML file only; templates still resolve from `refName`. Use it when the branch renamed the file the definition points at, or to expand a minimal root that includes just the template under test.
- A missing required parameter fails the preview with `A value for the '<name>' parameter must be provided.` That proves the **API** path only — the web UI's Run pipeline dialog may still pre-select a value (see the `pipelines-yaml-authoring` skill).
- Expansions of real pipelines are very large (tens of thousands of characters). Search the result for the job, step, or task you are verifying instead of reading it whole. With the Azure CLI you can filter it yourself: `az devops invoke --area pipelines --resource runs --route-parameters project=<project> pipelineId=<id> --http-method POST --api-version 7.1-preview --in-file <body.json>` with `{"previewRun": true, "resources": {"repositories": {"self": {"refName": "refs/heads/<branch>"}}}}`, then read `.finalYaml`. `--api-version 7.1-preview.1` fails to parse, and `--in-file` needs a real file path.
- `previewRun` validates expansion only. Runtime behaviour — for example `Cache@2` key resolution, task inputs, or secrets — is not checked.

## 4. Cancel and retry

- To cancel or retry a stage, use `pipelines_write` (action `update_build_stage`).
- To cancel a whole build with the CLI: `az pipelines runs update` does not exist; use `az devops invoke --area build --resource builds --route-parameters project=<project> buildId=<id> --http-method PATCH --api-version 7.1 --in-file <file>` with `{"status": "cancelling"}`.
- Retrying a stage of a build that already published artifacts can fail with `Artifact … already exists for build <id>`. Queue a new run instead.

# Examples

- "Run the CI pipeline on my branch feature/login"
- "Preview how pipeline 42 expands on branch ci/cache-change"
- "Cancel build 12345"
