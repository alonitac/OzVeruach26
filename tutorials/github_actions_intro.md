# Continuous Integration (CI) with GitHub Actions

In the [Git tutorial](git_intro.md) we agreed on one rule: **`main` is production**. Every change goes through a short-lived feature branch and a pull request, and only then into `main`:

```text
main ──► feature branch ──► pull request + automated checks + human or agent review ──► merge into main (production)
```

The pull request is the main gate for production.

**Continuous Integration** means that developers merge their changes into `main` often, and every change is verified by **automated checks** (tests, lint, scans) before it gets in. A broken change is caught on its branch, minutes after it was pushed, not by a grower in the field.

To run those checks we need an **automation platform**: a server that runs our checks on every pull request, shows the results in a nice UI, and blocks the merge when they fail. We'll use the one built into GitHub: **GitHub Actions**.

## Your first workflow

In your `ArgoPilot` repo, create `.github/workflows/registry-service.yaml` (in the **root** of the repo, not inside the service):

```yaml
name: registry-service

on:
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: services/registry-service

    steps:
      - uses: actions/checkout@v7

      - name: Install uv
        uses: astral-sh/setup-uv@v10

      - name: Install dependencies
        run: uv sync --locked

      - name: Run tests
        run: uv run pytest
```

A **workflow** is an automated process, defined in a [YAML](https://learnxinyminutes.com/docs/yaml/) file under `.github/workflows/`. GitHub finds every `.yml` or `.yaml` file in that folder, so the name of the file is up to you.

Let's read it from the top:

- **`on`**: the events that start the workflow. Here: every pull request into `main`. 
- **`jobs`**: a workflow has one or more jobs. This one has a single job called `test`.
- **`runs-on`**: every job runs on a fresh virtual machine, a **runner**. `ubuntu-latest` is a clean Ubuntu machine. Nothing from your laptop is there: no `.venv`, no `.env`, no `registry.db`. Only what's committed to git.
- **`defaults.run.working-directory`**: every `run` step runs inside `services/registry-service` dir.
- **`steps`**: run one after another. If a step fails, the job stops and is marked as failed.
    - **`uses`** runs a ready-made **action** that someone published. `actions/checkout` clones your repo into the runner. `astral-sh/setup-uv` installs uv.
    - **`run`** runs a shell command, exactly like in your terminal.
- **`uv sync --locked`** installs the exact versions from `uv.lock`, and **fails** if `uv.lock` doesn't match `pyproject.toml`. CI tests the same set of packages production installs (remember the [Python environment tutorial](python_env.md)). uv also reads `.python-version` from the repo root, so CI runs the same Python as you.


## The flow, end to end

Let's see the whole flow with a small feature. The registry stores plot area in square metres. Growers around here talk in dunams. Add the conversion:

```text
GET /v1/plots/{id}/area
```

```json
{ "plot_id": "c8f3d1b0-4d2e-5a1b-8b22-1d6e3f702002", "area_m2": 1000, "area_dunam": 1.0 }
```

- `area_dunam` = `area_m2 / 1000`. A dunam is 1000 m².
- Unknown plot: `404`, `code`: `NOT_FOUND`.

For **North block** in the seed: 1000 / 1000 = 1.0. **East block** is 800 / 1000 = 0.8.

#### Step 1 - Start from `main`, create a branch

```bash
git checkout main
git pull
git checkout -c feat/plot-area
```

Commit the workflow file on this branch too, if you haven't pushed it yet.

#### Step 2 - Implement, with tests

Ask your coding agent to implement the endpoint as described above, with tests.

Read the tests before you trust them. Do they check the actual numbers, or only that the status is `200`?

Run them locally:

```bash
cd services/registry-service
uv run pytest
```

#### Step 3 - Push and open a PR

1. Commit and push the branch.
2. Open a pull request from `feat/plot-area` into `main`.
3. Scroll down in the PR. You'll see the check **registry-service / test** running. Click **Details** to watch the log of every step, live.
4. If it fails, read the log, fix on your branch, and push again. The checks run again on every push.
5. When it's green and reviewed, merge.

After the merge, open the **Actions** tab of your repo. You'll see one more run: the `push` to `main`. That's production being verified.

## Make the checks required

Right now a red check is only a warning: the **Merge** button still works. Let's make GitHub enforce our flow on `main`.

In your repo on GitHub: **Settings** → **Rules** → **Rulesets** → **New ruleset** → **New branch ruleset**.

1. Name it `protect-main`, and set **Enforcement status** to **Active**.
2. **Target branches** → **Add target** → **Include default branch**.
3. Check **Restrict deletions** and **Block force pushes**.
4. Check **Require a pull request before merging**.
5. Check **Require status checks to pass**, click **Add checks**, and add `test`. (GitHub only suggests checks that already ran at least once in the repo.)
6. **Create**.

Now `main` only changes through a pull request whose checks passed. Try it:

```bash
git checkout main
echo "# direct push" >> README.md
git commit -m "Direct push to main"
git push
```

GitHub rejects the push. Undo the local commit with `git reset --hard origin/main`.



# Exercises

### :pencil2: Full coverage for registry-service


**Code coverage** is the number of lines of your code that ran while the tests ran.

**Full coverage** means every line ran, and every branch (if condition) went true and false.

> [!IMPORTANT]
> 100% coverage doesn't prove the code is right.

We'll require (near) 100% coverage for `registry-service`.


1. Install the coverage plugin for pytest, as a dev dependency:

    ```bash
    cd services/registry-service
    uv add --dev pytest-cov
    ```

2. Configure coverage in `pyproject.toml`:

    ```toml
    [tool.coverage.run]
    branch = true
    source = ["."]
    omit = ["tests/*"]

    [tool.coverage.report]
    show_missing = true
    ```

    `branch = true` turns on branch coverage. `omit` leaves the tests themselves out of the number.

3. Measure:

    ```bash
    uv run pytest --cov
    ```

    Read the `Missing` column. Then get a visual report:

    ```bash
    uv run pytest --cov --cov-report=html
    ```

    Open `htmlcov/index.html` in your browser and click a file. Red lines never ran. Yellow lines are branches that went only one way. (`htmlcov/` and `.coverage` are already in `.gitignore`.)

4. Reach as close to 100% as possible. For every red or yellow line, write a test that runs it **and asserts the result**. You should use a technique called **mocking** to simulate some situation that can't happen in a test - for example `503` when the database is unreachable. You can't unplug the database during the test, so simulate it: use pytest's `monkeypatch` to replace the function that opens the database connection with one that raises an error.

5. Make the rule a gate. Add this to `[tool.coverage.report]`:

    ```toml
    fail_under = 100
    ```

    Now `uv run pytest --cov` exits with an error when coverage drops below 100%, even if every test passed.

6. In the workflow, change the test step to:

    ```yaml
          - name: Run tests with coverage
            run: uv run pytest --cov --cov-report=term --cov-report=html

          - name: Upload coverage report
            if: always()
            uses: actions/upload-artifact@v7
            with:
              name: coverage-report
              path: services/registry-service/htmlcov/
    ```

    `if: always()` runs the upload even when the previous step failed. That's exactly when you need the report. After the run, you'll find `coverage-report` at the bottom of the run's **Summary** page.

7. Open a PR, see it pass, merge.

### :pencil2: Scan packages for known vulnerabilities

Tests check your code. `uv audit` checks the **packages** in `uv.lock`. It looks each version up in a public list of known problems ([OSV](https://osv.dev/)). If one of your packages is on that list, the command fails and tells you which newer version fixes it.

> [!IMPORTANT]
> A passing audit doesn't prove the packages are safe. It only means none of them is on the list today.

1. From the service, run the scan:

    ```bash
    cd services/registry-service
    uv audit --locked --preview-features audit-command
    ```

    `--locked` checks `uv.lock` as it is in git: the same versions CI and production install. The extra flag turns off a warning. This command is still new.

2. If a package is reported, move it to a newer version that `pyproject.toml` already allows, then scan again:

    ```bash
    uv lock --upgrade-package PACKAGE
    uv sync
    uv audit --locked --preview-features audit-command
    ```

    If the fix is a new major version, you choose whether to allow it. That choice is the one from the [Python environment tutorial](python_env.md). If there is no fix yet, skip that problem until one comes out. Put the id from the report in place of `GHSA-....`:

    ```toml
    [tool.uv.audit]
    ignore-until-fixed = ["GHSA-...."]
    ```

3. Add a second job to `.github/workflows/registry-service.yaml`:

    ```yaml
      audit:
        runs-on: ubuntu-latest
        defaults:
          run:
            working-directory: services/registry-service

        steps:
          - uses: actions/checkout@v7

          - name: Install uv
            uses: astral-sh/setup-uv@v10

          - name: Audit dependencies
            run: uv audit --locked --preview-features audit-command
    ```

    Jobs in one workflow run at the same time. This job reads the lockfile and checks the list.

4. In the `protect-main` ruleset, add the `audit` check next to `test`.

5. Open a PR, watch **registry-service / audit** go green, and merge.


