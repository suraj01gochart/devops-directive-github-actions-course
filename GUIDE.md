# GitHub Actions — Core Features Guide

> **Based on:** Module 03 of the DevOps Directive GitHub Actions Course  
> **Workflow files:** [`.github/workflows/03-core-features--*.yaml`](../.github/workflows/)  
> **Reference docs:** [GitHub Actions Documentation](https://docs.github.com/en/actions)

---

## Table of Contents

1. [Terminology & Mental Model](#1-terminology--mental-model)
2. [Hello World — The Minimal Workflow](#2-hello-world--the-minimal-workflow)
3. [Step Types](#3-step-types)
4. [Workflows, Jobs & Steps — Structure & DAGs](#4-workflows-jobs--steps--structure--dags)
5. [Triggers & Filters](#5-triggers--filters)
6. [Environment Variables & Scoping](#6-environment-variables--scoping)
7. [Passing Data Between Steps & Jobs](#7-passing-data-between-steps--jobs)
8. [Secrets & Variables](#8-secrets--variables)
9. [Contexts](#9-contexts)
10. [Quick Reference Cheat Sheet](#10-quick-reference-cheat-sheet)
11. [Common Mistakes to Avoid](#11-common-mistakes-to-avoid)

---

## 1. Terminology & Mental Model

Before writing any YAML, understand the four building blocks:

```
GitHub Event  →  triggers  →  Workflow
                               └── Job(s)
                                    └── Step(s)   (run on a Runner)
```

| Concept      | What it is                                                                    | Scope                                      |
|--------------|-------------------------------------------------------------------------------|--------------------------------------------|
| **Event**    | Something that happened on GitHub (push, PR, schedule, manual click…)         | External trigger                           |
| **Workflow** | A YAML file in `.github/workflows/`. The whole pipeline definition.           | File-level                                 |
| **Job**      | A group of steps that run on **one runner**. Jobs run in parallel by default. | Isolated compute environment per job       |
| **Step**     | A single command or action. Steps within a job share the same filesystem.     | Sequential within a job                    |
| **Runner**   | The virtual machine (GitHub-hosted or self-hosted) that executes a job.       | Spun up fresh for each job by default      |

> **Key insight:** Between steps → same filesystem/environment. Between jobs → completely separate environment. You must explicitly pass data across jobs.

---

## 2. Hello World — The Minimal Workflow

**File:** [`03-core-features--01-hello-world.yaml`](../.github/workflows/03-core-features--01-hello-world.yaml)

```yaml
name: Hello World

on:
  workflow_dispatch:       # Manual trigger — shows the "Run workflow" button in the UI

jobs:
  say-hello-inline-bash:
    runs-on: ubuntu-24.04  # GitHub-hosted Ubuntu runner
    steps:
      - run: echo "Hello from an inline bash script in a GitHub Action Workflow!"
```

### Anatomy of every field

| Field       | Purpose                                                               |
|-------------|-----------------------------------------------------------------------|
| `name`      | Display name in the GitHub Actions UI                                 |
| `on`        | Which event(s) trigger this workflow                                  |
| `jobs`      | Map of one or more named jobs                                         |
| `runs-on`   | Runner label — controls OS, pre-installed tools, CPU/RAM              |
| `steps`     | Ordered list of things to execute                                     |
| `run`       | Inline shell command (bash by default)                                |

### How to run it
1. Go to your repo → **Actions** tab → **Hello World** → **Run workflow** → **Run workflow**.
2. Click the run row → click the job → see the output.

---

## 3. Step Types

**File:** [`03-core-features--02-step-types.yaml`](../.github/workflows/03-core-features--02-step-types.yaml)

There are **three kinds of steps**:

### 3a. Inline Bash (default)

```yaml
steps:
  - run: echo "Hello from an inline bash script in a GitHub Action Workflow!"
```

Default shell on Ubuntu/Mac runners is **bash**. This is the most common step type.

### 3b. Inline Script in Another Language

```yaml
steps:
  - run: print("Hello from an inline python script in a GitHub Action Workflow!")
    shell: python
```

Swap `shell:` for any interpreter available on the runner (`python`, `pwsh`, `sh`, `cmd`, etc.).

### 3c. Marketplace Action (`uses:`)

```yaml
steps:
  - uses: actions/hello-world-javascript-action@081a6d193d1dcb38460df1e6927486d748730f9d # v1.1
    with:
      who-to-greet: "from an action in the GitHub Action marketplace! 👋"
```

- `uses: owner/repo@ref` — pulls in a reusable action from GitHub.
- `with:` — passes named inputs to the action.
- **Always pin to a commit hash** (not just `@v1`). A tag can be force-pushed; a commit hash cannot be changed. Add a comment with the tag for readability.

> **Why pin?** Malicious actors have wiped tags and re-pointed them to compromised commits. A SHA is immutable.

---

## 4. Workflows, Jobs & Steps — Structure & DAGs

**File:** [`03-core-features--03-workflows-jobs-steps.yaml`](../.github/workflows/03-core-features--03-workflows-jobs-steps.yaml)

```yaml
name: Workflows, Jobs, and Steps
on:
  workflow_dispatch:

jobs:
  job-1:
    runs-on: ubuntu-24.04
    steps:
      - run: echo "A job consists of"
      - run: echo "one or more steps"
      - run: echo "which run sequentially"
      - run: echo "within the same compute environment"

  job-2:
    runs-on: ubuntu-24.04
    steps:
      - run: echo "Multiple jobs can run in parallel"

  job-3:
    runs-on: ubuntu-24.04
    needs:             # job-3 waits for BOTH job-1 and job-2 to finish
      - job-1
      - job-2
    steps:
      - run: echo "They can also depend on one another..."

  job-4:
    runs-on: ubuntu-24.04
    needs:
      - job-2
      - job-3
    steps:
      - run: echo "...to form a directed acyclic graph (DAG)"
```

### DAG Execution Order

```
job-1 ──┐
         ├──► job-3 ──► job-4
job-2 ──┘
```

- Jobs with **no `needs`** run in parallel immediately.
- Jobs with `needs` start only after all listed jobs complete successfully.
- The graph must be **acyclic** — no circular dependencies allowed.

### Key Rules

| Rule                          | Detail                                             |
|-------------------------------|----------------------------------------------------|
| Steps within a job            | Always run **sequentially**, in order              |
| Jobs without `needs`          | Run **in parallel**                                |
| Jobs with `needs`             | Run only after listed jobs succeed                 |
| Filesystems across jobs       | Each job gets a **fresh runner** — not shared      |

---

## 5. Triggers & Filters

**File:** [`03-core-features--04-triggers-and-filters.yaml`](../.github/workflows/03-core-features--04-triggers-and-filters.yaml)

The `on:` key controls **what events** start the workflow and **which conditions** narrow those events down.

```yaml
on:
  # 1. Push to branches matching a glob pattern
  push:
    branches:
      - "example-branch/*"   # wildcard matches any sub-branch

  # 2. Pull request events with type and path filters
  pull_request:
    types:
      - opened        # PR was created
      - synchronize   # New commit pushed to the PR branch
      - reopened      # PR was re-opened
    paths:
      - "03-core-features/filters/*.md"    # trigger if these files changed
      - "!03-core-features/filters/*.txt"  # EXCLUDE .txt files (! = negate)

  # 3. Scheduled cron (UTC)
  # schedule:
  #   - cron: "0 0 * * *"   # midnight every day

  # 4. Manual trigger
  workflow_dispatch:
```

### The Four Most-Used Triggers

| Trigger              | When it fires                                              | Common use                        |
|----------------------|------------------------------------------------------------|-----------------------------------|
| `workflow_dispatch`  | You click "Run workflow" in the UI or call the GitHub API  | Manual runs, demos, tests         |
| `push`               | A commit is pushed to a matching branch                    | CI on merge to main               |
| `pull_request`       | A PR is opened, updated, or reopened                       | Pre-merge validation              |
| `schedule`           | Cron timer (UTC)                                           | Nightly jobs, dependency audits   |

### Filter Pattern Quick Reference

| Pattern       | Matches                                       |
|---------------|-----------------------------------------------|
| `main`        | Exact branch name                             |
| `release/*`   | Any branch starting with `release/`           |
| `**`          | Any path, any depth                           |
| `src/**`      | Any file under `src/`                         |
| `!docs/**`    | Exclude everything under `docs/`              |

> **Tip:** Path filters let a monorepo run only the relevant workflow when a specific service directory changes — avoiding wasteful runs across unrelated services.

### What the event payload looks like

The workflow dumps the full event JSON so you can explore it:

```yaml
steps:
  - name: Dump full event JSON
    run: |
      echo "=== $GITHUB_EVENT_NAME payload ==="
      cat "$GITHUB_EVENT_PATH"
```

`$GITHUB_EVENT_PATH` is a JSON file with everything about the triggering event: PR number, changed files, actor, branch, etc.

---

## 6. Environment Variables & Scoping

**File:** [`03-core-features--05-environment-variables.yaml`](../.github/workflows/03-core-features--05-environment-variables.yaml)

Environment variables can be set at **three levels**, each narrower than the previous.

```yaml
name: Environment Variables
on:
  workflow_dispatch:

env:                         # WORKFLOW scope: visible in every job and step
  WORKFLOW_VAR: I_AM_WORKFLOW_SCOPED

jobs:
  job-1:
    runs-on: ubuntu-24.04
    env:                     # JOB scope: visible only within job-1
      JOB_VAR: I_AM_JOB_1_SCOPED
    steps:
      - name: Inspect scopes job 1 step 1
        env:                 # STEP scope: visible only in this one step
          STEP_VAR: I_AM_STEP_SCOPED
        run: |
          echo "WORKFLOW_VAR: $WORKFLOW_VAR"   # ✅ visible
          echo "JOB_VAR:      $JOB_VAR"        # ✅ visible
          echo "STEP_VAR:     $STEP_VAR"       # ✅ visible only here

      - name: Inspect scopes job 1 step 2
        run: |
          echo "WORKFLOW_VAR: $WORKFLOW_VAR"             # ✅ visible
          echo "JOB_VAR:      $JOB_VAR"                 # ✅ visible
          echo "STEP_VAR:     ${STEP_VAR:-<UNSET>}"     # ❌ not set here

  job-2:
    runs-on: ubuntu-24.04
    steps:
      - name: Inspect scopes job 2 step 2
        run: |
          echo "WORKFLOW_VAR: $WORKFLOW_VAR"             # ✅ still visible
          echo "JOB_VAR:      ${JOB_VAR:-<UNSET>}"      # ❌ not set here
          echo "STEP_VAR:     ${STEP_VAR:-<UNSET>}"     # ❌ not set here
```

### Scope Summary

| Where defined    | Visible in                          |
|------------------|-------------------------------------|
| Workflow `env`   | All jobs, all steps                 |
| Job `env`        | All steps in that job only          |
| Step `env`       | That one step only                  |

---

## 7. Passing Data Between Steps & Jobs

**File:** [`03-core-features--06-passing-data.yaml`](../.github/workflows/03-core-features--06-passing-data.yaml)

Because each job runs in a separate VM, data doesn't flow automatically. There are two mechanisms:

### Method 1: `GITHUB_ENV` — share an env var within the same job

```yaml
- name: Generate and export foo
  id: generate-foo
  run: |
    foo=bar
    echo "FOO=${foo}" >> "$GITHUB_ENV"   # append KEY=VALUE to the special env file

- name: Inspect values inside producer
  run: echo "FOO (set via GITHUB_ENV): $FOO"   # ✅ picks it up within the same job
```

### Method 2: `GITHUB_OUTPUT` + job `outputs` — pass data across jobs

```yaml
jobs:
  producer:
    runs-on: ubuntu-24.04
    outputs:
      foo: ${{ steps.generate-foo.outputs.foo }}   # expose step output as job output
    steps:
      - name: Generate and export foo
        id: generate-foo                           # id required to reference outputs
        run: |
          foo=bar
          echo "foo=${foo}" >> "$GITHUB_OUTPUT"    # write key=value to output file

      - name: Inspect values inside producer
        run: |
          echo "FOO (set via GITHUB_ENV):   $FOO"
          echo "foo (step output):          ${{ steps.generate-foo.outputs.foo }}"

  consumer:
    runs-on: ubuntu-24.04
    needs: producer                                # must declare dependency
    steps:
      - name: Inspect values inside consumer (note FOO is unset)
        run: |
          echo "Value from producer: ${{ needs.producer.outputs.foo }}"   # ✅
          echo "FOO in consumer:     ${FOO:-<UNSET>}"                     # ❌ env vars don't carry over
```

### Step-by-step flow for cross-job data

```
Step     → echo "key=val" >> $GITHUB_OUTPUT
Job      → outputs: key: ${{ steps.<id>.outputs.key }}
Consumer → needs: <producer-job>
Consumer → ${{ needs.<producer-job>.outputs.key }}
```

### What does NOT persist across jobs

| Thing              | Persists across steps? | Persists across jobs? |
|--------------------|------------------------|-----------------------|
| `GITHUB_ENV` vars  | ✅ Yes (same job only) | ❌ No                 |
| Step outputs       | ✅ Yes (via `steps.*`) | Only via job `outputs`|
| Filesystem files   | ✅ Yes (same job only) | ❌ No (use artifacts) |

---

## 8. Secrets & Variables

**File:** [`03-core-features--07-secrets-and-variables.yaml`](../.github/workflows/03-core-features--07-secrets-and-variables.yaml)

### Where to store them

| Level              | How to set                                          | Scope                          |
|--------------------|-----------------------------------------------------|--------------------------------|
| **Organization**   | Org Settings → Secrets/Variables → Actions          | Shared across repos in org     |
| **Repository**     | Repo Settings → Secrets and variables → Actions     | All environments in that repo  |
| **Environment**    | Repo Settings → Environments → staging/production   | Only jobs using that env name  |

- **Secrets** → for sensitive values (passwords, API keys). Masked in logs. Write-only in the UI.
- **Variables** → for non-sensitive config. Readable and editable in the UI.

### How to use them in a workflow

```yaml
jobs:
  staging-environment:
    runs-on: ubuntu-24.04
    environment: staging          # activates environment-level secrets/vars

    env:
      # Repository-level items
      EXAMPLE_REPOSITORY_SECRET:   ${{ secrets.EXAMPLE_REPOSITORY_SECRET }}
      EXAMPLE_REPOSITORY_VARIABLE: ${{ vars.EXAMPLE_REPOSITORY_VARIABLE }}
      # Environment-level items (only available when environment: is set)
      EXAMPLE_ENVIRONMENT_SECRET:   ${{ secrets.EXAMPLE_ENVIRONMENT_SECRET }}
      EXAMPLE_ENVIRONMENT_VARIABLE: ${{ vars.EXAMPLE_ENVIRONMENT_VARIABLE }}

    steps:
      - name: Inspect values inside job
        run: |
          echo "Repo secret (masked):    $EXAMPLE_REPOSITORY_SECRET"
          echo "Repo variable:           $EXAMPLE_REPOSITORY_VARIABLE"
          echo "Env secret (masked):     $EXAMPLE_ENVIRONMENT_SECRET"
          echo "Env variable:            $EXAMPLE_ENVIRONMENT_VARIABLE"

  production-environment:
    runs-on: ubuntu-24.04
    environment: production       # same key names, different values
    env:
      EXAMPLE_REPOSITORY_SECRET:   ${{ secrets.EXAMPLE_REPOSITORY_SECRET }}
      EXAMPLE_REPOSITORY_VARIABLE: ${{ vars.EXAMPLE_REPOSITORY_VARIABLE }}
      EXAMPLE_ENVIRONMENT_SECRET:   ${{ secrets.EXAMPLE_ENVIRONMENT_SECRET }}
      EXAMPLE_ENVIRONMENT_VARIABLE: ${{ vars.EXAMPLE_ENVIRONMENT_VARIABLE }}
    steps:
      - name: Inspect values inside job
        run: |
          echo "Repo secret (masked):    $EXAMPLE_REPOSITORY_SECRET"
          echo "Repo variable:           $EXAMPLE_REPOSITORY_VARIABLE"
          echo "Env secret (masked):     $EXAMPLE_ENVIRONMENT_SECRET"
          echo "Env variable:            $EXAMPLE_ENVIRONMENT_VARIABLE"
```

### Key rules

- Use `${{ secrets.NAME }}` and `${{ vars.NAME }}` — the `${{ }}` expression syntax.
- Secrets are **automatically masked** in logs — GitHub replaces their value with `***`.
- Environment-level secrets are **only available** when the job declares `environment:`.
- A staging job and production job can use the **same secret name** but get **different values**.

---

## 9. Contexts

Contexts are read-only objects accessible via `${{ <context>.<property> }}` throughout your workflow.

| Context    | What's inside                                                        | Example usage                        |
|------------|----------------------------------------------------------------------|--------------------------------------|
| `github`   | Repo name, event name, actor, SHA, ref, run ID…                     | `${{ github.actor }}`                |
| `env`      | All environment variables set at workflow/job/step level             | `${{ env.MY_VAR }}`                  |
| `secrets`  | Repository and environment secrets                                   | `${{ secrets.API_KEY }}`             |
| `vars`     | Repository and environment variables (non-sensitive)                 | `${{ vars.ENV_NAME }}`               |
| `needs`    | Outputs from upstream jobs (when `needs:` is set)                   | `${{ needs.build.outputs.version }}` |
| `steps`    | Outputs from previous steps within the **same job**                 | `${{ steps.my-step.outputs.result }}`|
| `matrix`   | Current matrix values when using a matrix strategy                  | `${{ matrix.os }}`                   |
| `job`      | Current job's status and container info                              | `${{ job.status }}`                  |
| `runner`   | OS, architecture, temp directory of the runner                      | `${{ runner.os }}`                   |
| `inputs`   | Inputs provided to `workflow_dispatch` or a reusable workflow        | `${{ inputs.version }}`              |

---

## 10. Quick Reference Cheat Sheet

### Minimal Workflow Skeleton

```yaml
name: My Workflow

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  my-job:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@<commit-hash> # v4
      - run: echo "Hello!"
```

### Common `on:` Patterns

```yaml
on:
  push:
    branches: [main, "release/*"]
    paths: ["src/**"]

  pull_request:
    types: [opened, synchronize]
    paths: ["src/**", "!docs/**"]

  schedule:
    - cron: "0 2 * * 1"   # Every Monday at 2am UTC

  workflow_dispatch:
    inputs:
      environment:
        description: "Target environment"
        required: true
        default: "staging"
```

### Environment Variables — All Three Levels

```yaml
env:                         # Workflow scope
  GLOBAL: hello

jobs:
  example:
    env:                     # Job scope
      LOCAL: world
    steps:
      - env:                 # Step scope
          TEMP: temp-value
        run: echo "$GLOBAL $LOCAL $TEMP"
```

### Passing Data Within a Job

```yaml
- id: my-step
  run: echo "result=42" >> "$GITHUB_OUTPUT"

- run: echo "Got ${{ steps.my-step.outputs.result }}"
```

### Passing Data Across Jobs

```yaml
jobs:
  build:
    outputs:
      version: ${{ steps.set-ver.outputs.version }}
    steps:
      - id: set-ver
        run: echo "version=1.2.3" >> "$GITHUB_OUTPUT"

  deploy:
    needs: build
    steps:
      - run: echo "Deploying ${{ needs.build.outputs.version }}"
```

### Secrets & Variables

```yaml
env:
  API_KEY: ${{ secrets.API_KEY }}         # secret (masked in logs)
  APP_ENV: ${{ vars.APP_ENVIRONMENT }}    # variable (visible in logs)
```

### Using a Marketplace Action (safely)

```yaml
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
  with:
    fetch-depth: 0
```

---

## 11. Common Mistakes to Avoid

| ❌ Mistake | ✅ Fix |
|-----------|--------|
| Pinning to a mutable tag (`@v4`) | Pin to the commit SHA and add a `# v4` comment |
| Expecting env vars to persist across jobs | Use `outputs:` + `needs:` to pass data across jobs |
| Setting a step-scoped `env:` and expecting it in the next step | Move `env:` to job level, or use `GITHUB_ENV` |
| Forgetting `id:` on a step whose output you want to read | Always set `id:` on steps that write to `GITHUB_OUTPUT` |
| Using `GITHUB_OUTPUT` format incorrectly | Must be `key=value` appended with `>>`, no extra spaces |
| Hardcoding secrets in YAML | Always use `${{ secrets.NAME }}` |
| Accessing environment secrets without declaring `environment:` | Add `environment: <name>` to the job |
| Circular job dependencies | Jobs form a DAG — no loops allowed |

---

## What's Next

You've covered all 7 Core Feature workflows. The natural next step is **Module 04 — Advanced Features**, which builds on everything here:

- **Runner Types** — third-party and self-hosted runners
- **Artifacts** — persisting files across jobs
- **Caching** — avoiding repeated downloads
- **GitHub Permissions** — scoping `GITHUB_TOKEN`
- **Third-Party Auth** — OIDC vs. static credentials
- **Matrix & Conditionals** — fan-out and `if:` expressions
- **Workflow Commands** — `::error::`, `::notice::` and more

See: [`04-advanced-features/`](../04-advanced-features/)

---

*Guide created from the DevOps Directive GitHub Actions Course — Module 03*
