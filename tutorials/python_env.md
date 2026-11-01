# Python development environment

On the first day you set up the machine you will code on, then run the project. In this course the project is AgroPilot. The service you run today is `registry-service`, a small FastAPI application. Its only route is `GET /health`.

## Get the code

1. Open the course repository: https://github.com/alonitac/ArgoPilot
2. Create your own copy. Use **Use this template** when that button is there. Otherwise click **Fork**.
3. Name the copy (for example `ArgoPilot`) and create it.

Clone your copy (Windows users - do it in WSL terminal):

```bash
git clone https://github.com/YOUR-USERNAME/ArgoPilot.git
cd ArgoPilot
```

Open the folder in VS Code: **File** → **Open Folder**, then select the cloned project.

The registry service lives here:

```text
services/registry-service/
  pyproject.toml          dependencies for this service
  uv.lock                 exact versions uv will install
  src/main.py    the FastAPI application
```

## What a Python environment is

A Python environment is the place your code runs. It includes:

- the Python interpreter, the program that executes your code
- the packages installed for that interpreter
- environment variables

`which python` prints the interpreter your shell will use. In VS Code, open a terminal with **Terminal** → **New Terminal** and run that command.

Each project needs its own environment. A shared interpreter collects packages from every project, and those packages eventually conflict. An isolated environment, a virtual environment, holds one project's packages in a `.venv` folder.

## pip

**pip** is Python's package installer. It downloads libraries from the Python Package Index (PyPI). Many projects still install packages with pip, but we will use more modern tool called `uv`.

## uv

AgroPilot uses **uv** for `registry-service`. uv creates the virtual environment and installs the packages listed for the service.

Install Python 3.12 and uv, then check both:

```bash
python3 --version
uv --version
```

The uv install instructions are at https://docs.astral.sh/uv/getting-started/installation/.

From the AgroPilot folder:

```bash
cd services/registry-service
uv sync
```

`uv sync` reads `pyproject.toml` and `uv.lock`, creates `services/registry-service/.venv`, and installs FastAPI and uvicorn at the locked versions.

VS Code may offer to use this interpreter. Accept it. If it does not, press `Ctrl+Shift+P` (`Cmd+Shift+P` on macOS), run **Python: Select Interpreter**, and choose `services/registry-service/.venv`.

Start the service:

```bash
uv run uvicorn main:app --host 127.0.0.1 --port 8000
```

Open http://127.0.0.1:8000/health. A running service returns:

```json
{"status": "ok"}
```

http://127.0.0.1:8000/docs lists all service endpoint in what called a Swagger UI. Stop the server with `Ctrl+C`.

To add a library later, stay in `services/registry-service` and run `uv add <package>`. uv updates `pyproject.toml` and `uv.lock`.

### The line in `pyproject.toml` is a minimum

A dependency looks like this:

```toml
dependencies = [
    "fastapi>=0.115.0",
]
```

`>=0.115.0` means 0.115.0 or newer. 0.115.0 is the oldest FastAPI this service is allowed to run. That floor is a project decision: this code relies on behavior that exists from that release onward.

The three numbers are a semantic version, `MAJOR.MINOR.PATCH` (for example `2.4.1`):

- **PATCH** (`2.4.1` → `2.4.2`) fixes bugs. Calls that worked still work.
- **MINOR** (`2.4.1` → `2.5.0`) adds features. Existing calls still work.
- **MAJOR** (`2.4.1` → `3.0.0`) can break existing calls.

`>=` accepts every newer release, including the next major. A later `uv lock --upgrade` is allowed to select that major. Production then installs it from the lock, and the service can break.

`~=` written with the major and the minor stops at the next major. Drop the patch:

```toml
"fastapi~=0.115"    # 0.115.0 or newer, and still 0.x. Rejects 1.0.0.
```

`~=0.115` allows a newer patch (`0.115.4`) and a newer minor (`0.116.0`). It rejects `1.0.0`. Moving to that major is a manual edit of this line. `~=0.115.0` (three numbers) is tighter than you want here: it also rejects the next minor.

One person runs `uv lock --upgrade`, tests the service, and commits `uv.lock`. Everyone else, and production, run `uv sync` and get that set. The upgrade can move patch and minor versions. It cannot cross into the next major until someone edits `pyproject.toml` first.


### `uv.lock` is the tested set

`uv lock` solves every constraint and writes in `uv.lock` one exact version for each direct package and for each package those packages need. `uv sync` installs that list.

`pyproject.toml` states what the service is allowed to use. `uv.lock` states what it will use. Every developer and production install from the lock, so they run the same set of packages.


## Debug the service in VS Code

The debugger pauses the program on a line you choose, so you can inspect variables and step forward.

1. Open `services/registry-service/src/main.py`.
2. Click in the gutter next to `return {"status": "ok"}`. A red dot marks the breakpoint.
3. Open **Run and Debug** (`Ctrl+Shift+D`).
4. Choose **registry-service** and start it.

That launch configuration runs uvicorn with the service folder as the working directory. Open http://127.0.0.1:8000/health. Execution stops on the return line. Use the debug toolbar to continue.

# Exercises

### :pencil2: Stop the next major

This exercise shows how uv handles minor and major version changes.

Work in `services/registry-service`.

1. Open `pyproject.toml`. Pin both packages to these exact versions.

```toml
dependencies = [
    "fastapi==0.115.0",
    "uvicorn==0.32.0",
]
```

2. Lock that set and install it:

```bash
uv lock
uv sync
```

As seen, `==` left no room for a newer release.

3. Change both lines to `~=` and drop the patch number.

```toml
dependencies = [
    "fastapi~=0.115",
    "uvicorn~=0.32",
]
```

4. Upgrade inside that range and install again:

```bash
uv lock --upgrade
uv sync
```

`uv lock --upgrade` prints an `Updated` line for each package that moved.
Now both versions are newer than the pins, and both still start with `0`. 

FastAPI and uvicorn have no `1.0` release, so this step stays inside `0.x`.

5. Now let's install a package that has a major version.
  Add `rich`, pinned to an old major:

```toml
dependencies = [
    "fastapi~=0.115",
    "uvicorn~=0.32",
    "rich==13.0.0",
]
```

6. Lock and install:

```bash
uv lock
uv sync
```

7. Change that line to `rich~=13.0`. Save, then upgrade and check again:

```bash
uv lock --upgrade
uv sync
```

`rich` moves from `13.0.0` to a newer `13` release. The first number stays `13`. The command does not select `15.0.0`.

8. Allow the new major. Change the line to `rich~=15.0`, save, and run the same two commands. `uv lock --upgrade` crossed that major only after you edited `pyproject.toml`.

9. Remove the `rich` line. Run `uv lock` and `uv sync`. Leave FastAPI and uvicorn on `~=`.
