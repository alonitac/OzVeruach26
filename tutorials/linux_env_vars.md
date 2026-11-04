# Linux Environment Variables

An **environment variable** is a name and a value that a process can read. A child process inherits a copy of its parent's environment. The shell uses this to pass settings such as your home directory, your language, and the list of folders it searches for commands.

`HOME` is one you have already used. `cd $HOME` moves to your home directory. The shell replaces `$HOME` with the value before it runs `cd`.

`printenv` prints the environment. 

```console
myuser@hostname:~$ printenv HOME
/home/myuser
```

## The `PATH` variable

You run `ls` by name. You rarely type `/bin/ls`. `which` shows the file the shell actually runs:

```console
myuser@hostname:~$ which ls
/bin/ls
myuser@hostname:~$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

`PATH` is a list of directories, separated by colons. When you type a command, the shell looks in each directory, in order, for an executable file with that name. `/bin` is on `PATH`, so `ls` is enough.


## `export`

A name you set in the shell is a shell variable. It is visible in that shell. A child process does not receive it.

`export` copies the variable into the environment. Every process you start from that shell inherits it.

```bash
export PORT=8000
cd services/registry-service
uv run uvicorn main:app --host 127.0.0.1 --port "$PORT"
```

The shell expands `$PORT` to `8000` before it starts `uv`. The `uv` process, and the uvicorn process under it, also inherit `PORT`.

Set the variable for one command by writing it on the same line. That form does not change the shell afterwards:

```bash
PORT=8000 uv run uvicorn main:app --host 127.0.0.1 --port "$PORT"
```

A later `echo $PORT` in that shell prints nothing. The value existed only for that one command.

Check what a child can see:

```console
myuser@hostname:~$ PORT=8000
myuser@hostname:~$ printenv PORT
myuser@hostname:~$ export PORT=8000
myuser@hostname:~$ printenv PORT
8000
myuser@hostname:~$ bash -c 'printenv PORT'
8000
```

`bash -c` starts a child shell. After `export`, the child prints `8000`. Before `export`, `printenv PORT` prints a blank line.

## `source`

`source` runs a file in the **current** shell. Variables and `PATH` changes in that file remain after it finishes. Running the same file with `bash` starts a child process. The child exits, and the current shell is unchanged.

Activating the registry virtual environment is the case you will meet:

```bash
cd services/registry-service
source .venv/bin/activate
```

The activate script puts `.venv/bin` at the front of `PATH` in this shell. `which python` and `which uvicorn` then point inside `.venv`. The prompt gains a `(.venv)` prefix.

`$$` is the PID of the current shell. It stays the same across `source`, because you never left the shell:

```console
myuser@hostname:~/ArgoPilot/services/registry-service$ echo $$
44132
myuser@hostname:~/ArgoPilot/services/registry-service$ bash .venv/bin/activate
myuser@hostname:~/ArgoPilot/services/registry-service$ echo $$
44132
myuser@hostname:~/ArgoPilot/services/registry-service$ source .venv/bin/activate
(.venv) myuser@hostname:~/ArgoPilot/services/registry-service$ echo $$
44132
(.venv) myuser@hostname:~/ArgoPilot/services/registry-service$ which uvicorn
/home/myuser/ArgoPilot/services/registry-service/.venv/bin/uvicorn
```

The first `bash .venv/bin/activate` runs in a child. The child exits at once, and `uvicorn` is still not on your `PATH`. `source` changes this shell, so `which uvicorn` finds the one in `.venv`.

`deactivate` undoes that `PATH` change. You can also close the terminal. The course commands use `uv run`, so you can leave the environment inactive and keep using `uv run uvicorn`.

# Exercises

### :pencil2: Add a command of your own

You will write a small script, place it in a folder, and add that folder to `PATH`. The shell will then run it by name, the same way it runs `ls`.

Run these steps in order.

1. In your home directory, create a directory called `scripts`.
2. Create a file called `myscript` with no extension:

```bash
#!/bin/bash
echo my script is running...
```

3. Run it with `bash myscript`.
4. Make it executable: `chmod +x myscript`.
5. Move it to `~/scripts`.
6. Add `~/scripts` to `PATH`. Keep the directories already there. Put the new directory at the front:

```bash
export PATH="$HOME/scripts:$PATH"
```

7. Run `myscript` with no path. The shell finds it through `PATH`.
8. Open a new terminal and run `myscript` again.

The new terminal is a new process. It inherited the environment from the program that opened it, not from the shell where you ran `export`. The assignment was only in that first shell.

To keep the folder on `PATH` for every new terminal, add the `export` line to `~/.bashrc`, then open a new terminal and try `myscript` again.
