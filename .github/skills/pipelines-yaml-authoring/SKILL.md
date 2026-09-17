---
name: pipelines-yaml-authoring
description: Write and review Azure Pipelines YAML (azure-pipelines.yml, templates, steps, variables, parameters, conditions, Cache@2, test publishing) while avoiding silent traps. Use when creating or changing pipeline YAML, or when a pipeline behaves differently than its YAML suggests.
---

# Azure Pipelines YAML authoring

Most of these traps fail **silently** — the pipeline stays green while doing something other than intended. After changing YAML, expand it with `previewRun` (see the `pipelines-run-and-validate` skill) and search the expanded YAML for the value or task you changed.

# Rules

## 1. Steps and shells

- `- script:` compiles to `CmdLine@2`, which runs **cmd.exe on Windows agents**. Use `- bash:` (`Bash@3`) for any shell syntax — arrays, `if`, pipes with bash semantics — and `- pwsh:` for PowerShell. Before adding shell syntax to an existing `script:` step, switch the step type.
- `$(name)` is replaced before the script runs **only if `name` is a defined variable**. Unmatched `$( … )` is left untouched, so bash command substitution such as `VERSION=$(node -p "…")` works. Do not rewrite it to backticks.
- A defined macro is substituted even inside single quotes. Never compare a secret against its own macro (`[ "$TOKEN" = '$(MY_TOKEN)' ]` always passes); validate its shape instead.
- `az extension add` re-queries the extension index even when installed (several seconds); guard it with `az extension show`.

## 2. Variables, secrets, and environment

- `System.AccessToken` and **secret** variables are not visible to scripts unless mapped into `env:`.
- Non-secret variables from a variable group are exposed to every step automatically. Map them explicitly anyway: marking one secret later silently turns it empty in every step that relied on auto-exposure.
- A variable whose name maps to the same environment variable as a secret (`.` becomes `_`) shadows the secret — the mapped value becomes empty.
- Secret values can never be read back once set, neither in the UI nor the CLI, so they cannot be copied between pipelines. Share secrets through a variable group instead of per-definition variables, which drift apart.
- Azure Pipelines does **not** set `CI`; it sets `TF_BUILD`. Gate CI-only behaviour on `CI || TF_BUILD`, or set `CI` yourself.
- `Build.SourceBranchName` is only the **last path segment** (`feature/login` becomes `login`). APIs that filter by branch need `Build.SourceBranch`.
- A `[skip ci]` in the **tip** commit's message suppresses CI for the whole push, including the commits below it.

## 3. Templates and parameters

- `${{ variables.x }}` inside a template renders **empty** for a variable declared in the root file, because templates are expanded before those variables exist. Pass the value as a template parameter.
- `type: boolean` parameters reach scripts inconsistently cased (`True`, `False`, `true`). Compare case-insensitively (`${VAR,,}` in bash) or compare at template level with `${{ if eq(parameters.x, true) }}`.
- Template expression `eq()` compares strings **case-insensitively**, so `Prod` matches `'prod'`.
- A parameter with `values:` and no `default:` is **not required**: the Run pipeline dialog pre-selects the first value, while the REST API rejects the missing value. If a value must be chosen deliberately, drop `values:` (a free-text parameter without default blocks the dialog) and add a guard stage that fails on unexpected values — a misspelling otherwise makes `${{ if }}` blocks emit nothing and the run passes green having done nothing.

## 4. Conditions

- An explicit `condition:` **replaces** the implicit `succeeded()`; it does not add to it. Include `succeeded()` (or `failed()`, `always()`) deliberately, for example `condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))`.

## 5. Tasks

- Pin built-in tasks to the **major** version only (`PublishTestResults@2`). A full version pin silently runs whatever build of that major the agent has installed. Renovate's `azure-pipelines` manager expands majors to full versions by default; keep them major-only with an `extractVersion` rule if you use it.

## 6. Caching with Cache@2

- Each `|`-separated key segment is interpreted by shape: a segment containing `/` is treated as a **file path to hash** and fails the task if the file does not exist. Quote literal segments: `"packages/app/node_modules"`, `"$(Agent.OS)"`.
- The key decides what is **restored**, never what is **saved**: the post-job step saves whatever is on disk. On agents that are reused between builds, a cached directory that survives on disk grows with every build.
- On an **exact** key hit the save is skipped. A key that only hashes a lockfile therefore freezes the cache content. For caches that should evolve, end the key with `$(Build.SourceVersion)` and use `restoreKeys` with the stable prefix.
- The checkout step's clean (`git clean -ffdx`) deletes untracked directories inside the sources directory. Put caches that must survive checkout under `$(Pipeline.Workspace)`, outside `$(Build.SourcesDirectory)`.
- On self-hosted scale-set pools, agents are recycled or reused unpredictably. Compare the **Agent name** in *Initialize job* between runs before concluding that a cache change worked or failed.

## 7. Publishing test results

- `PublishTestResults@2` follows symlinks when matching files. In workspaces that symlink packages into `node_modules` (pnpm, npm workspaces), one result file can be published many times, inflating test counts. Narrow the search folder or pattern, or set `mergeTestResults: true` with a distinct `testRunTitle`.
- `No Result Found to Publish` for a project without tests is expected. A regression looks like a **drop** in the `TestResults To Publish <n>` count.
- If publishing is slow, `publishRunAttachments: false` skips uploading the run attachments.

# Examples

- "Add a cache for the pnpm store to our pipeline"
- "Why does this step behave differently on the Windows agent?"
- "Make the deploy stage run only on main"
