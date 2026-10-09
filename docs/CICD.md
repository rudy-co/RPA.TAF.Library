# TAF.Library: CI/CD and automatic consumer updates (runbook)

This runbook explains how a change to **TAF.Library** reaches every UiPath process that depends on it,
and lists the one-time setup needed for the automation to work.

---

## 1. How it works

```
Developer merges PR into TAF.Library branch (Development | UAT | Production)
 │
 ├─ GitHub Actions: .github/workflows/CI_TAF_Library.yml
 │    validate → restore → (analyze) → test → pack → publish to Orchestrator library feed
 │    version = MAJOR.MINOR (project.json) . PATCH (run number + offset) [-dev | -rc]
 │
 └─ job notify-consumers (one per repo in .github/consumers.json subscribed to that branch)
      │  repository_dispatch "taf-library-published" {version, branch, ...}
      ▼
   Consumer repo: .github/workflows/update-taf-library.yml
      bump project.json  "TAF.Library": "[1.1.123]"
      restore + pack (compile check only, nothing is published)
      open/update PR  deps/taf-library-<branch>
      │
      ├─ Development → auto-merge once "pr-validate" passes
      └─ UAT / Production → waits for CODEOWNERS approval
      ▼
   Merge commit on consumer branch
      ▼
   Automation Ops pipeline (runMode ForEachCommit, one per process per environment)
      build → analyze → test → publish → (approval in Prod) → update Orchestrator process
```

### Design decisions (why it is built this way)

| Decision | Reason |
|---|---|
| Library CI only **notifies**; it never edits consumer repos directly | Each consumer team owns its repo. Every bump is a reviewable, revertable PR with an audit trail. |
| One PR branch per target branch (`deps/taf-library-<branch>`) | A newer library release updates the open PR instead of opening a second one, so PRs don't pile up. |
| A compile check runs before the PR is opened, and `pr-validate` is a required check | A breaking library change shows up as a red PR, not a failed robot job in Production. |
| Auto-merge only on Development | Fast feedback in Dev, with a human gate before UAT and Production. |
| Exact version pin `[x.y.z]` in `project.json` | Builds are reproducible: what was tested in UAT is exactly what ships to Production. |
| Pre-release versions (`-dev`, `-rc`) are blocked from Production | This is enforced twice: in `update-taf-library.yml` and in `pr-validate.yml`. |
| GitHub App token instead of a personal PAT | The token is short-lived (about 1 hour), scoped to selected repos and not tied to an employee. PRs it opens also trigger CI, which PRs opened with `GITHUB_TOKEN` don't. |
| Automation Ops pipeline for consumer deployment | It reuses the Git connection already set up in Automation Ops, runs on your own robots and updates the bound Orchestrator process natively. |

---

## 2. Versioning rules

| Library branch | Example version | Who receives it |
|---|---|---|
| `Development` | `1.1.123-dev` | Consumers' `Development` branch |
| `UAT` | `1.1.124-rc` | Consumers' `UAT` branch |
| `Production` | `1.1.125` | Consumers' `Production` branch |
| Pull request | `1.1.126-pr.42` | Nobody (it is packed but not published) |

- **PATCH is automatic:** `github.run_number + VERSION_PATCH_OFFSET`. The repo variable `VERSION_PATCH_OFFSET` defaults to `100`, which keeps new versions above the manually published `1.1.19`.
- **MINOR:** bump `projectVersion` in `project.json` (for example `1.2.0`) when you add new workflows.
- **MAJOR:** bump it when you change or remove arguments of an existing public workflow. Consumer PRs will then fail
  the compile check on purpose, so each team has to adapt its process before merging.
- Manual override: run the workflow with **Run workflow → version**.

---

## 3. One-time setup checklist

### 3.1 Git branches and GitHub rules (library repo and every consumer repo)
1. Create the branches `Development`, `UAT` and `Production`. Promotion is always by PR:
   feature → Development → UAT → Production.
2. **Settings → Rules → Rulesets** for those three branches:
   - Require a pull request before merging.
   - Required status checks: `Build` (library) or `pr-validate` (consumers).
   - Required approvals: 1 for UAT and 2 for Production, with *Require review from Code Owners* turned on.
   - Block force pushes and deletions.
3. **Settings → Environments:** create `uipath-dev`, `uipath-uat` and `uipath-prod`.
   - `uipath-prod`: add required reviewers and restrict deployment branches to `Production`.
4. Consumer repos: **Settings → General → Allow auto-merge** = on.

### 3.2 UiPath credentials (per environment)
1. **Automation Cloud → Admin → External Applications → Add:** create a *Confidential* application with
   the Orchestrator API scopes needed to publish and read packages (folders/execution scopes).
   Create one app per environment (Dev / UAT / Prod), or at least one per tenant.
2. In each GitHub environment (`uipath-dev`, `uipath-uat`, `uipath-prod`), in both the library repo and the consumer repos:
   - Secrets: `UIPATH_CLIENT_ID`, `UIPATH_CLIENT_SECRET`
   - Variables: `UIPATH_TENANT`, and `UIPATH_URL` (optional, default `https://cloud.uipath.com`)
3. Write the secret expiry dates here and rotate them before they expire:

   | Environment | App name | Expires |
   |---|---|---|
   | Dev | | |
   | UAT | | |
   | Prod | | |

> Enterprise recommendation: use **separate tenants** (or at least separate folders and feeds) for
> Dev / UAT / Prod, so a `-dev` library can never be resolved by a production robot.

### 3.3 GitHub App `taf-dependency-bot`
1. **Organization settings → Developer settings → GitHub Apps → New GitHub App**
   - Webhook: off
   - Repository permissions: **Contents: Read & write**, **Pull requests: Read & write**,
     **Actions: Read & write**, **Metadata: Read**
2. Generate a private key and install the app on the **library repo and all consumer repos**.
3. Add the organization (or repo) secrets `BOT_APP_ID` and `BOT_PRIVATE_KEY` (the full `.pem` content).

### 3.4 Library repo variables
| Variable | Default | Purpose |
|---|---|---|
| `VERSION_PATCH_OFFSET` | `100` | Added to the run number to get PATCH. |
| `ENABLE_ANALYZER` | `false` | Set to `true` to run Workflow Analyzer in CI once the rules pass. |

### 3.5 Onboard a consumer process
1. Copy from this repo into the consumer repo:
   - `ci-templates/consumer/.github/workflows/update-taf-library.yml` → `.github/workflows/`
   - `ci-templates/consumer/.github/workflows/pr-validate.yml` → `.github/workflows/`
   - `ci-templates/consumer/CODEOWNERS` → `.github/CODEOWNERS` (set the real team)
2. Make sure the consumer's `project.json` already depends on `TAF.Library`.
3. Add the repo to `.github/consumers.json` in this library repo with `"enabled": true`, through a PR.
   That PR is the change-control record for onboarding.
4. Create the Automation Ops pipelines (section 3.6).

### 3.6 Automation Ops pipelines (deployment of consumers)
Prerequisites:
- **Automation Ops → Pipelines:** the pipeline process is installed in a dedicated Orchestrator folder (for example `CI-CD`)
  with an **unattended robot** and a free unattended runtime. Pipelines run as Orchestrator jobs.
- The consumer **process already exists** in each environment's folder. Publish it once manually,
  because the pipeline updates an existing process.
- The CI/CD admins have the `Pipelines.View`, `Pipelines.Edit` and `Pipelines.Run` permissions.

Create **one pipeline per consumer per environment**, either in the UI or with the CLI:

```bash
uip tools install @uipath/aops-tool
uip login

uip aops connection list --output json                     # → connectionIdentifier
uip aops connection repos <CONNECTION_ID> --output json    # → RemoteId (NOT Identifier), CloneUrl, Name
uip aops repo project-files <REPO_IDENTIFIER> --output json  # → projectPath
uip aops pipeline processes --output json                  # → ProcessId of the target process

# fill ci-templates/aops/pipeline-consumer-<env>.json, save as pipeline-<process>-<env>.json
uip aops pipeline list --search "<PROCESS_NAME>-dev" --output json   # name must be free
uip aops pipeline create --file pipeline-<process>-dev.json --output json
```

Settings for each pipeline:
- `runMode`: `ForEachCommit`. Every merged commit, including bot bump PRs, builds and deploys.
- `branch`: `Development` / `UAT` / `Production`, paired with the matching environment's process.
- Stages: Build → Analyze → Run tests → Publish → **Update process**. Add an **approval** before Update for Production.

### 3.7 Monitoring
- Library run summary: shows the version, channel and every consumer that was notified.
- Consumer PRs: filter by the label `taf-library`.
- AOps runs: run `uip aops pipeline executions <pipeline-id> --output json` and
  `uip aops execution logs <execution-id> --follow`.

---

## 4. Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `Notify consumers` lists no repos | No enabled consumer subscribes to that branch in `consumers.json`. The job doesn't run at all for PRs. |
| `Create GitHub App token` fails | The app isn't installed on the consumer repo, or `BOT_APP_ID` / `BOT_PRIVATE_KEY` are missing. |
| Consumer bump job fails at restore | The consumer environment's credentials can't read the library feed, or the version was published to another tenant. |
| Consumer PR is red | The process doesn't compile against the new library (breaking change). Fix the process on the PR branch. |
| PR opened but CI didn't run | The PR was created with `GITHUB_TOKEN` instead of the App token. Check the `app-token` step. |
| AOps pipeline didn't start after merge | Check the pipeline's `runMode` (`ForEachCommit`) and branch, then run `uip aops connection sync <id> --wait`. |
| Publish fails with "version already exists" | A re-run used the same run number. Re-run with a manual `version`, or raise `VERSION_PATCH_OFFSET`. |
| Production bump rejected "pre-release" | This is expected. Only stable versions from the library's `Production` branch reach Production. |

---

## 5. Rollback
- **Consumer:** revert the bump PR on the consumer branch. The AOps pipeline redeploys the previous version.
  For an emergency, set the previous package version on the Orchestrator process directly.
- **Library:** never delete published versions. Fix forward, or revert the commit so that CI publishes a new, higher PATCH.
