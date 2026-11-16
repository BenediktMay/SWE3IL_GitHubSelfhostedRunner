# GitHub Self-Hosted Runner & CI/CD Migration Documentation

## 1. Overview & Objective
The goal of this assignment is to migrate an existing GitLab CI/CD pipeline (`.gitlab-ci.yml`) to GitHub Actions running on a local self-hosted runner.

- **Repository**: [https://github.com/BenediktMay/SWE3IL_GitHubSelfhostedRunner](https://github.com/BenediktMay/SWE3IL_GitHubSelfhostedRunner)
- **Successful CI/CD Run**: [Run #43923955133](https://github.com/BenediktMay/SWE3IL_GitHubSelfhostedRunner/actions/runs/43923955133)
- **Official References**:
  - [GitHub Docs: Migrate from GitLab CI/CD to GitHub Actions](https://docs.github.com/en/actions/tutorials/migrate-to-github-actions/manual-migrations/migrate-from-gitlab-cicd)
  - [GitHub Docs: Manage self-hosted runners](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners)

---

## 2. Concept Mapping: GitLab CI vs. GitHub Actions

Following the [GitHub migration guide](https://docs.github.com/en/actions/tutorials/migrate-to-github-actions/manual-migrations/migrate-from-gitlab-cicd):

| GitLab CI/CD (`.gitlab-ci.yml`) | GitHub Actions (`.github/workflows/ci.yml`) | Description / Behavior |
| :--- | :--- | :--- |
| `stages: [lint, typchecking, test]` | `jobs.<job_id>` + `needs: [...]` | GitLab executes stages sequentially by default. GitHub Actions runs jobs concurrently unless dependencies are declared via `needs: [lint, typecheck]`. |
| `default.image: ghcr.io/astral-sh/uv` | `astral-sh/setup-uv@v5` + local host | In GitLab, jobs execute inside containers. On self-hosted Windows, tools run directly on host shell with `astral-sh/setup-uv`. |
| `variables:` | `env:` | Environment variables (`UV_CACHE_DIR`, `UV_COMPILE_BYTECODE`, `UV_LINK_MODE`) mapped globally and per-step. |
| `cache:` | `astral-sh/setup-uv` (`enable-cache: true`) | Caching dependencies based on `uv.lock`. |
| `allow_failure: true` | `continue-on-error: true` | Applied to `ruff-format` so formatting issues do not block downstream stages. |
| `before_script: [uv sync --locked]` | Workflow job step: `uv sync --locked` | Replicated as an explicit step across all jobs prior to executing checks. |
| GitLab Runner | Self-Hosted Runner (`runs-on: self-hosted`) | Executes workflow steps on the user's local machine. |

---

## 3. Step-by-Step Implementation

### Step 1: Archiving GitLab CI
- Renamed `.gitlab-ci.yml` $\rightarrow$ `.gitlab-ci.yml.disabled` to keep historical record while disabling GitLab CI execution.
- Committed and pushed to GitHub (`chore: archive gitlab ci configuration as disabled`).

### Step 2: Translating Pipeline to GitHub Actions
- Created `.github/workflows/ci.yml` containing 4 jobs targeting `runs-on: self-hosted`:
  - `lint`: `uv run ruff check`
  - `lint-format`: `uv run ruff format --check` (`continue-on-error: true`)
  - `typecheck`: `uv run ty check`
  - `test`: `uv run pytest` (depends on `lint` and `typecheck`)
- Committed and pushed to GitHub (`ci: add github actions workflow for self-hosted runner`).

### Step 3: Self-Hosted Runner Installation & Configuration
1. **Acquiring Runner Software**:
   - Provisioned directory at `C:\actions-runner`.
   - Downloaded latest supported GitHub Actions Runner release (`v2.338.2`).
2. **Registration**:
   - Generated a registration token using GitHub CLI API (`gh api repos/:owner/:repo/actions/runners/registration-token`).
   - Configured runner against repository:
     ```powershell
     .\config.cmd --url https://github.com/BenediktMay/SWE3IL_GitHubSelfhostedRunner --token <TOKEN> --name "local-runner" --work "_work" --unattended
     ```
3. **Execution**:
   - Started the listener loop via `.\run.cmd`.
   - Connected successfully: `√ Connected to GitHub` $\rightarrow$ `Listening for Jobs`.

### Step 4: Verification & Successful Execution
- Runner picked up workflow run `#43923955133`.
- Jobs `Type Checking (ty)`, `Ruff Lint`, and `Pytest` completed with **Succeeded** (`exit code 0`).
- Job `Ruff Format Check` failed on style formatting as expected, but workflow continued cleanly due to `continue-on-error: true`.
- Entire workflow run concluded with **Success**.

---

## 4. Challenges Faced & Solutions

1. **GitHub Runner Minimum Version Enforcement**:
   - *Challenge*: Attempting to register runner `v2.322.0` failed with HTTP 404 (`Runner version is out of date and can no longer register with GitHub. Minimum required: 2.329.0`).
   - *Solution*: Queried GitHub releases API dynamically to fetch the latest supported runner release (`v2.338.2`), allowing registration to proceed.
2. **Windows PowerShell Execution Policy**:
   - *Challenge*: Runner script execution threw `PSSecurityException: UnauthorizedAccess` (`running scripts is disabled on this system`).
   - *Solution*: Updated PowerShell execution policy for current user: `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser -Force`.
3. **Container vs. Native Host Tooling**:
   - *Challenge*: The GitLab pipeline used a Debian container with Linux path conventions.
   - *Solution*: Replaced container references with native Windows execution, leveraging `astral-sh/setup-uv@v5` and `${{ github.workspace }}` path expressions.

---

## 5. Screenshots

Below are the required screenshots demonstrating the runner setup and pipeline execution.

**Screenshot 1 – GitHub Runner Settings (self‑hosted tab)**
[View on GitHub](https://github.com/BenediktMay/SWE3IL_GitHubSelfhostedRunner/actions/runners?tab=self-hosted)
![Screenshot 1](file:///C:/Users/Benedikt/.gemini/antigravity/brain/f6f60f3b-243f-4cb0-bf7d-de21b3ca1963/.user_uploaded/media_1794852795262_b6790a4a.png)

**Screenshot 2 – CI workflow file view on GitHub**
[View workflow file](https://github.com/BenediktMay/SWE3IL_GitHubSelfhostedRunner/actions/workflows/ci.yml)
![Screenshot 2](file:///C:/Users/Benedikt/.gemini/antigravity/brain/f6f60f3b-243f-4cb0-bf7d-de21b3ca1963/.user_uploaded/media_1794852807708_de10a49e.png)

**Screenshot 3 – Latest workflow run after formatting fix**
[View run](https://github.com/BenediktMay/SWE3IL_GitHubSelfhostedRunner/actions/runs/43927723275)
![Screenshot 3](file:///C:/Users/Benedikt/.gemini/antigravity/brain/f6f60f3b-243f-4cb0-bf7d-de21b3ca1963/.user_uploaded/media_1794852827400_f9f39201.png)

**Screenshot 4 – Runner console output showing job execution**
![Screenshot 4](file:///C:/Users/Benedikt/.gemini/antigravity/brain/f6f60f3b-243f-4cb0-bf7d-de21b3ca1963/.user_uploaded/media_1794853354888_0b9e7194.png)

---

## 6. AI Assistance Disclosure
*Note: This documentation, runner configuration, and CI/CD pipeline setup were created with assistance from AI.*

