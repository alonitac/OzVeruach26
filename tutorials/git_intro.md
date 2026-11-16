# SDLC with Git

A coding agent can change 20 files in a minute. If something breaks, you want to know what changed, and you want a way back.

That's what **git** gives you. It also lets a team work on the same project without stepping on each other. Together with **GitHub**, it's how a change goes from an idea to a reviewed and released version. This process is called the **SDLC** (Software Development Life Cycle).


## Every commit is a snapshot

A **commit** is a snapshot of the whole project at a point in time. It also saves who made it, when, a message, and a unique ID (a **hash**, like `a1b2c3d`).

Your project history is a chain of these snapshots. You can go back to any of them, or compare two of them.

![][git_versions]

To go back to a commit, you can use the hash.

```bash 
git log --oneline
```

This will list all commits, with their hash and message.

```bash 
git checkout <hash>
```


Git doesn't copy files that didn't change, it just links to the previous copy (the dashed files above). So commits are cheap. Commit often.

Everything is stored in a hidden `.git` folder in the project. That folder **is** the repository.

> [!TIP]
> On your first commit, git may ask who you are. Set it once:
>
> ```bash
> git config --global user.name "Your Name"
> git config --global user.email "you@example.com"
> ```

## The three areas

![][git_areas]

- **Working tree**: the files you (and the agent) edit.
- **Staging area**: the changes you picked for the next commit.
- **Repository** (`.git`): the saved commits.

The flow is always: edit → **stage** (`git add`) → **commit** (`git commit`).

Why stage? Because you don't always want to commit everything. Say the agent fixed the plot validation bug *and* reformatted three other files. You can stage only the fix.

### File status

| Status | Meaning | In VS Code |
| --- | --- | --- |
| **Untracked** | A new file git doesn't know yet. | `U` |
| **Modified** | Changed since the last commit, not staged yet. | `M` |
| **Staged** | Will go into the next commit. | under **Staged Changes** (`A` for a new file) |
| **Unmodified** | Same as in the last commit. | not shown |
| **Deleted** | Removed. | `D` |

### Let's try it

Open a terminal in the root of your `ArgoPilot` repo:

```console
$ git status
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
```

> [!NOTE]
> If you see changes from Task P1 that you haven't committed yet, commit them first, so you start from a clean tree.

Now change the first line of `README.md` to `# AgroPilot - irrigation planning for growers`, and create a new file `CHANGELOG.md`:

```markdown
# Changelog

## v1.0.0
- registry-service: farms, plots, and the crop, soil and emitter catalogues (Task P1).
```

```console
$ git status
On branch main
Changes not staged for commit:
        modified:   README.md

Untracked files:
        CHANGELOG.md
```

> [!TIP]
> **In VS Code:** open the **Source Control** tab (`Ctrl+Shift+G`). The **Changes** list shows the same files, with their status letter.

`git diff` shows what changed, line by line:

```console
$ git diff
-# AgroPilot
+# AgroPilot - irrigation planning for growers
```

> [!TIP]
> **In VS Code:** click a file in the **Changes** list to see its diff side by side.

Stage both files:

```console
$ git add README.md CHANGELOG.md
$ git status
On branch main
Changes to be committed:
        new file:   CHANGELOG.md
        modified:   README.md
```

> [!TIP]
> **In VS Code:** hover a file and click **+** to stage it. The file moves to **Staged Changes**.

> [!NOTE]
> `git add` stages the file **as it is right now**. If you edit it again, you need to `git add` it again.

Commit:

```console
$ git commit -m "Add changelog and a tagline to the README"
[main 4e1f2a9] Add changelog and a tagline to the README
 2 files changed, 5 insertions(+), 1 deletion(-)
 create mode 100644 CHANGELOG.md
```

> [!TIP]
> **In VS Code:** write the message in the box at the top of the Source Control tab and click **Commit**. If your editor has an AI (sparkle) button next to the box, it can write the message for you.
>
> A good commit message says **why**. "Reject plots with a planting date in the future" is much better than "fix".

To undo changes you **haven't committed yet**:


```bash
git restore <file>   # Certain file 
git restore .        # All files
```

> [!TIP]
> **In VS Code:** hover a file and click **−** to unstage it, or the curved arrow to discard its changes.

## .gitignore

Some files should never be committed: secrets (`.env`), the virtual environment (`.venv/`, it's huge and `uv sync` recreates it), and cache files.

List them in a `.gitignore` file, and git ignores them. Here's part of the one in `ArgoPilot`:

```text
# Python
__pycache__/
*.py[cod]
.venv/

# Secrets and local env
.env
.env.*
!.env.example

# Tools and test output
.pytest_cache/
.ruff_cache/
.coverage
htmlcov/
```

`!.env.example` means "except this one". `.env.example` holds safe example values, so it **is** committed.

Ignored files don't show up in `git status`, and VS Code shows them greyed out. 

`.gitignore` itself **is** committed, so the whole team ignores the same files.

## Branches

A **branch** is a separate line of work. You commit on it as much as you want, and `main` doesn't change until you merge.

In AgroPilot there are no long-lived branches besides `main`. **`main` is production**: whatever is on `main` can be deployed to growers at any moment, so `main` must always work.

Every change, big or small, gets its own short-lived **feature branch** from `main`, like `feat/farm-summary` or `fix/plot-planting-date`. When it's done, it goes back into `main` through a pull request, and the branch is deleted.

```text
main  ──●───────────────────────●──────────────●───   (production)
         \                     /                \
          ●───●───●───●───────●  feat/farm-summary ...
                       PR: review + tests
```

Branches are also great for agent experiments: don't like the result? Delete the branch.

When you checkout branches, git changes your files to match that branch. So commit (or discard) your changes before you checkout.

> [!TIP]
> **In VS Code:** the current branch is shown at the bottom-left corner. Click it to checkout, or to create a new branch.

## Tags

A **tag** is a name for one commit, usually a release, like `v1.0.0`. Unlike a branch, a tag never moves.

Release versions follow the same **semantic versioning** you met in the [Python environment tutorial](python_env.md): `MAJOR.MINOR.PATCH`.

- **PATCH** (`v1.0.1`): a bug fix.
- **MINOR** (`v1.1.0`): a new feature, nothing breaks. For example a new endpoint.
- **MAJOR** (`v2.0.0`): something breaks for the callers, for example the planner or the frontend. An endpoint was removed, or a field was renamed.

The registry API you finished in P1 is your first release. Tag it:

```bash
git tag -a v1.0.0 -m "Registry API (Task P1)"
git tag            # list tags
git show v1.0.0    # see the tag and its commit
```

`-a` saves who created the tag, when, and a message. Use it for releases.

## GitHub

So far everything was only on your computer. **GitHub** keeps a copy of the repository (a **remote**) that the team shares. When you cloned, git named it `origin`.

Your computer and GitHub don't sync by themselves. Your commits stay on your computer until you push them.

### push and pull

```bash
git push origin main     # send your commits to GitHub (origin is the default name for the remote)
git pull origin main     # get new commits from GitHub
git push origin v1.0.0   # tags are not pushed with git push, push them like this
```

The first push of a new branch is a bit different, since GitHub doesn't have it yet:

```bash
git push -u origin feat/farm-summary
```

After that, `git push` and `git pull` just work on this branch.

> [!TIP]
> **In VS Code:** the button at the top of the Source Control tab says **Publish Branch** for a new branch, and **Sync Changes** (pull + push) for an existing one.

> [!TIP]
> Pull `main` before you create a branch, so you start from the latest production code.

### Pull requests

Since `main` is production, nobody pushes straight to it. You push your branch and open a **pull request** (PR): *"please review my branch and merge it into `main`"*.

In a PR:

- **Review**: teammates (and AI reviewers) read your changes in the **Files changed** tab and comment on specific lines.
- **Checks**: tests, lint and other scans run automatically on your branch. You'll set them up in the [CI tutorial](github_actions_intro.md).
- **Merge**: when it's approved and the checks pass, click **Merge pull request**. Now your commits are in `main`, which means they are in production.



# Exercises

### :pencil2: Stop tracking `registry.db`

In P1 you probably committed `registry.db`, the SQLite file that `seed_registry.sh` creates. It shouldn't be in git:

- It's a binary file. `git diff` can't show what changed in it.
- It changes every time you run the service or the tests, so it shows up as modified all the time.
- Anyone can recreate it with `bash seed_registry.sh`.

Check whether git tracks it:

```bash
git ls-files services/registry-service/registry.db
```

If this prints nothing, you're done. Otherwise:

1. Add `*.db` to `.gitignore`. Run `git status`. `registry.db` still shows up as modified when it changes, as git keeps tracking a file it already knows.
2. Remove it from git, but keep it on your disk:

    ```bash
    git rm --cached services/registry-service/registry.db
    ```

3. Run `git status`. The file is staged as **deleted**, and it's no longer listed as untracked (it's ignored now). Check that it's still on your disk with `ls services/registry-service`.
4. Commit the `.gitignore` change and the removal together, with a message that says why.

> [!NOTE]
> The file is still in the **old** commits. Anyone can check out an old commit and get it. 

### :pencil2: Your first pull request

Now let's build a feature, the right way. A grower wants to see, for each farm, how many plots it has and their total area:

```text
GET /v1/farms/{id}/summary
```

```json
{ "farm_id": "b7e2c0a0-3c1d-4f0a-9a11-0c5d2e6f1001", "plot_count": 2, "total_area_m2": 1800 }
```

An unknown farm returns `404` with `code`: `NOT_FOUND`, like every other route.

1. Start from the latest `main`, and create a branch:

    ```bash
    git checkout main
    git pull
    git checkout -c feat/farm-summary
    ```

2. In your coding agent: *"In `services/registry-service`, add `GET /v1/farms/{id}/summary` as described: [paste the spec above]. Add tests for a farm with plots, a farm with no plots, and an unknown farm. Run the tests."*
3. Check it yourself: run `bash seed_registry.sh`, start the service, and call the endpoint for **Demo cooperative**. The seed has two plots for it, 1000 m² and 800 m², so you should get `1800`.
4. Commit the changes and push the branch.
5. On GitHub, click **Compare & pull request**. Make sure the base is `main`. Write a title and a short description (or ask the agent to write it).
6. **Review.** Open a new chat with a **different model** than the one that wrote the code, and ask:

    > Review the changes on this branch compared to `main`. If you find something worth fixing, fix it, commit, and push.

7. Watch the PR update.
8. Merge the PR on GitHub, and delete the branch (GitHub shows a button for it).
9. Back in VS Code, update your `main` and delete the local branch:

    ```bash
    git checkout main
    git pull
    git branch -d feat/farm-summary
    ```

10. `main` now runs in production with a new endpoint. A new feature that breaks nothing is a MINOR release. Add a `## v1.1.0` entry to `CHANGELOG.md` on a branch, merge it the same way, then tag:

    ```bash
    git tag -a v1.1.0 -m "Farm summary endpoint"
    git push origin v1.1.0
    ```

[git_versions]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/git_versions.png
[git_areas]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/git_areas.png
