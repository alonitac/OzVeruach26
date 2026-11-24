# Continuous Delivery (CD) with GitHub Actions

**Continuous Delivery** means a merge to `main` deploys that commit to production environment. 

## Before you start

- The EC2 instance from the [AWS tutorial](aws_intro.md#exercises) is **running**, in `us-east-1`.
- `registry.service` is enabled. On the instance, `curl http://127.0.0.1:8000/health` returns `{"status":"ok"}`.
- You still have the `.pem` key from when you launched the instance.
- `main` is protected by the `protect-main` ruleset from the CI tutorial.


## Store the host and the key

A workflow file is committed in Git. But we can't commit a private key. GitHub keeps values like this in **Actions secrets**.

In your repo on GitHub: **Settings** → **Secrets and variables** → **Actions** → **New repository secret**. Create two:

| Name | Value |
| --- | --- |
| `EC2_HOST` | The instance's public IPv4 address |
| `EC2_SSH_KEY` | The whole `.pem` file, including the `BEGIN` and `END` lines |

## The deploy workflow

On a branch, create `.github/workflows/deploy-registry.yaml` in the **root** of the repo, next to `registry-service.yaml`:

```yaml
name: deploy-registry

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy registry-service
        uses: appleboy/ssh-action@v1.2.5
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ubuntu
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            set -e
            cd /home/ubuntu/ArgoPilot
            git checkout main
            git pull
            cd services/registry-service
            /home/ubuntu/.local/bin/uv sync --locked
            sudo systemctl restart registry.service
```

- **`on: push` to `main`**: the merge is the deploy. 
- **`appleboy/ssh-action`** opens SSH from the runner and runs `script` on the instance. `host` and `key` come from the secrets. This step checks the private key. It does not pin the server's host key.
- **`set -e`**: the script stops on the first command that fails, and the job goes red.
- **`git pull`**: the instance only moves forward to the commits already on GitHub. It will not invent a merge commit on the server.
- **`/home/ubuntu/.local/bin/uv`**: this SSH session is not an interactive login, so the `uv` on your PATH when you type on the instance is not on the PATH here. `uv sync --locked` installs the same lockfile CI installed.
- **`systemctl restart`**: the Ubuntu image lets the `ubuntu` user run `sudo` without a password. Restart sends `SIGTERM` and starts the unit again. `Restart=on-failure` in the unit file is for crashes later, not for this deploy.

## See a merge deploy

1. Commit the workflow on a branch and open a pull request into `main`.
2. Wait until **registry-service / test** is green. 
3. Merge.
4. Open the **Actions** tab. The push to `main` starts **deploy-registry / deploy**.
5. On the instance:

    ```bash
    git -C /home/ubuntu/ArgoPilot log -1 --oneline
    curl -fsS http://127.0.0.1:8000/health
    ```

    Make sure the commit matches the merge on GitHub and that health returns `{"status":"ok"}`.


# Exercises

### :pencil2: Give the instance a fixed address

When you stop the instance and start it again, the public IPv4 address can change. Then `EC2_HOST` is a dead address, the pipeline in GitHub Actions cannot SSH into the instance.

An **Elastic IP** is a public IPv4 address allocated to your AWS account. You associate it with the instance. Stop and start, and the address stays.

1. In the EC2 console, confirm the region is `us-east-1`.
2. **Elastic IPs** (under **Network & Security**) → **Allocate Elastic IP address** → **Allocate**. Leave the defaults. The address comes from Amazon's pool.
3. Select the new address → **Actions** → **Associate Elastic IP address**. Resource type **Instance**, choose your instance, **Associate**.
4. Put the Elastic IP in the `EC2_HOST` secret (edit the secret, or delete it and create it again).
5. On the **Actions** tab, open the last **deploy-registry** run and choose **Re-run all jobs**. A re-run reads the current secrets.

### :pencil2: Deploy this run's commit

You merge a pull request with a bug. The deploy runs, and production breaks. You want the previous version back with one click: open the last green **deploy-registry** run in the GitHub Actions page and choose **Re-run all jobs**.

With the current workflow, that click does not help. The script runs `git pull`, which moves the instance to the tip of `main`. The tip is the buggy commit, so the re-run deploys the bug again.

The fix: each run deploys its own commit, not the tip of `main`. GitHub Actions puts the commit that triggered the run in `${{ github.sha }}`. A re-run keeps the same value.

Replace the `script` of `deploy-registry.yaml` with:

```yaml
          script: |
            set -e
            cd /home/ubuntu/ArgoPilot
            git fetch origin main
            git reset --hard ${{ github.sha }}
            cd services/registry-service
            /home/ubuntu/.local/bin/uv sync --locked
            sudo systemctl restart registry.service
```

`git fetch` downloads the commits from GitHub. `git reset --hard` checks out this run's commit.

1. Create a new branch, commit the change, open a pull request, and merge. This run is your good deploy.
2. Merge a second pull request with a bug. Its run deploys the bug.
3. Open the run from step 1 and choose **Re-run all jobs**.
4. On the instance, `git log -1 --oneline` shows the commit from step 1. Production is back on the good version, while `main` on GitHub still has the bug.

### :pencil2: A pipeline that sets up the instance by itself

Look at what the pipeline does **not** do. You created `services/registry-service/.env` on the instance by hand. You wrote `/etc/systemd/system/registry.service` by hand, and ran `systemctl enable` by hand. You cloned the repo and installed `uv` by hand.

None of that is in git, and none of it is in the pipeline. If the instance dies, nobody remembers exactly what was on it. If you add a setting to `.env` on your laptop, production doesn't get it until someone SSHs in and types it.

Given a **new, clean EC2 instance**, a run of the pipeline should bring production up, with no manual step on the instance.

Modify your deploy workflow to write `.env` and install `registry.service` on every run.

Notes:

- The unit file is not a secret. Commit it to the repo, next to the service, and let the pipeline copy it to `/etc/systemd/system/`.

- `.env` must stay out of git. You should store secrets in **Settings** → **Secrets and variables** → **Actions**, then read them from the workflow as `${{ secrets.NAME }}` and write the `.env` file on the instance. We recommend that to write the whole `.env` on every run, from scratch. Don't append to it. 

- Every run has to work on a clean instance **and** on one that was already deployed 20 times. Before each step, ask: what if this already exists? Where does the repo come from on the first run? 

- `systemctl restart` returns before your app finishes starting. A service that crashes on startup (for example, a missing setting) still gives you a green job. It's a good practice to end the script with a health check on the instance, so a broken start turns the job red.

### :pencil2: Deploy the full stack

Production runs only `registry-service`.

Deploy the whole AgroPilot stack from [Task P2](../tasks/p2.md) on your instance, through the pipeline: `frontend`, `planner-service`, `weather-service`, `registry-service`, and PostgreSQL.

Notes:

- PostgreSQL: we prefer you run it in [Docker](https://docs.docker.com/engine/install/ubuntu/), with the official [`postgres`](https://hub.docker.com/_/postgres) image and a named volume, so the data survives a restart of the container. 
- Each service has its own `.env`. The URLs between services point at `127.0.0.1` on the instance. The database password is a secret.
- The grower only needs the frontend. Open port `3000` in the security group. Which other ports really need to be open to the internet, and which services can listen only on `127.0.0.1`?
- `t3.medium` is small. Run `npm run build` once per deploy and serve with `npm run start`, not `npm run dev`.
- The clean instance test from the previous exercise still holds, for the whole stack.

Open `http://<elastic-ip>:3000` on your phone, and see today's plan for North block.
