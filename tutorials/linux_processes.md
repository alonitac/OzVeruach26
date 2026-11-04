# Linux Processes

A process is a program that is running. The program is a file of instructions. When you start it, Linux copies those instructions into memory and gives the process room for its data. That running copy is the process.

The registry service you started in the [Python environment tutorial](python_env.md) is one process. The shell you typed the command in is another.

`top` lists running processes and refreshes the list:

```console
myuser@hostname:~$ top
...
```

Press `q` to leave `top`.

## Process states

Many users can run many processes on one machine. A single CPU core runs one process at a time, and switches among them. A process is always in one of these states:

![][linux_process_state]

1. **Running**. The process is executing on a CPU core.
2. **Waiting**. The process is paused until something else finishes, such as a disk read, a network response, or keyboard input.
3. **Ready**. The process can run, and it is waiting for a free CPU core. The scheduler moves it to Running when a core is free. How long it waits depends on the scheduler and on how busy the machine is.
4. **Initialized**. The process has been created and does not have a process id (PID) yet.
5. **Terminated**. The process has finished and has exited.
6. **Zombie**. The process has exited, and its parent has not yet read the exit status. The process stays in the table until the parent does.

## The process tree

A new process starts when an existing process copies itself. The original is the parent. The copy is the child. The result is a tree. `pstree` prints it:

```console
myuser@hostname:~$ pstree
systemd─┬─ModemManager───2*[{ModemManager}]
        ├─NetworkManager───2*[{NetworkManager}]
        ├─accounts-daemon───2*[{accounts-daemon}]
        ├─acpid
        ├─atd
.
.
.
```

`systemd` is the first process. Its PID is 1. Every other process is under it.

The child inherits the parent's environment. It gets its own PID. When the child exits, it returns an **exit code** to the parent. `0` means success. Any other code means failure.

`ps` prints the processes and those details:

```console
myuser@hostname:~$ ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.1 172752  9440 ?        Ss   13 Feb  7:21 /sbin/init splash
root         2  0.0  0.0      0     0 ?        S    13 Feb  0:00 [kthreadd]
```

`PID` is the process id. `STAT` is the state. `%CPU` and `%MEM` are usage. `USER` owns the process. `COMMAND` is what started it.

While the registry service is running, find it:

```console
myuser@hostname:~$ ps aux | grep uvicorn
myuser   18421  0.4  0.8  ...  uvicorn main:app --host 127.0.0.1 --port 8000
```

`18421` is an example. Your PID will differ.

## Signals

A process receives **signals** from other processes and from the kernel. `kill` sends a signal. The name is old: most signals do not kill the process.

| Signal  | Number | Meaning                                                         |
|---------|--------|-----------------------------------------------------------------|
| SIGHUP  | 1      | The controlling terminal closed, or the parent session ended.   |
| SIGINT  | 2      | Interrupt. The process may catch this and shut down in its own way. |
| SIGKILL | 9      | Kill at once. The process cannot catch this signal.            |
| SIGTERM | 15     | Ask the process to exit. The process may catch this and clean up first. |

`man 7 signal` lists the rest.

Try it with a process that only waits:

```console
myuser@hostname:~$ sleep 600
```

`sleep 600` does nothing for 600 seconds, then exits. Leave it running.

In a **second terminal**, find the PID and stop it:

```console
myuser@hostname:~$ ps -a
PID TTY          TIME CMD
46214 pts/1    00:00:00 sleep
46445 pts/3    00:00:00 ps
...
myuser@hostname:~$ kill -9 46214
```

The first terminal reports that `sleep` was killed. `kill -9` sends `SIGKILL`.

The same keys work on the process in the foreground of your terminal:

| Shortcut | Signal  | Meaning                                              |
|----------|---------|------------------------------------------------------|
| Ctrl+C   | SIGINT  | Interrupt the foreground process.                    |
| Ctrl+Z   | SIGTSTP | Suspend the foreground process. `fg` resumes it.    |

You already use the first one. In the Python tutorial, Ctrl+C stops the registry service.

### Graceful shutdown

**Graceful shutdown** means the process finishes current work, releases what it holds, and then exits. A database connection that closes cleanly is the usual case. `SIGTERM` is the signal that asks for this. `SIGKILL` skips it: the process is gone, and cleanup does not run.

Uvicorn, which runs the registry service, already handles `SIGINT` and `SIGTERM`. Start the service as in the [Python environment tutorial](python_env.md):

```bash
cd services/registry-service
uv run uvicorn main:app --host 127.0.0.1 --port 8000
```

Ctrl+C in that terminal sends `SIGINT`. Uvicorn stops accepting requests, then exits. You can see the shutdown lines in the terminal.

To see the other path, start the service again. In a second terminal, list the matching processes:

```console
myuser@hostname:~$ ps aux | grep uvicorn
```

You will see two lines. One command is `uv run`. The other is `uvicorn` itself. Use the PID from the `uvicorn` line:

```console
myuser@hostname:~$ kill -15 <PID>
```

The service terminal prints a shutdown and the process exits. `curl http://127.0.0.1:8000/health` then fails, because nothing is listening.

Send `SIGKILL` once, on a fresh start, and compare:

```console
myuser@hostname:~$ kill -9 <PID>
```

The process disappears. Uvicorn does not run its shutdown. Use `SIGTERM` (or Ctrl+C) when you stop the registry service yourself.

## Services

A **service** is a process that stays running and does one job for the system or for other programs. `ssh` and `cron` are services.

**systemd** manages services. It is PID 1. A **unit file** is a text file that tells systemd how to start, stop, and restart one service. Your own unit files go in `/etc/systemd/system/`.

`systemctl` is the command you use:

```console
myuser@hostname:~$ sudo systemctl status ufw
myuser@hostname:~$ sudo systemctl start ufw
myuser@hostname:~$ sudo systemctl stop ufw
myuser@hostname:~$ sudo systemctl restart ufw
myuser@hostname:~$ sudo systemctl enable ufw
```

`enable` starts the service on every boot. `start` starts it now.

List the services systemd knows about:

```console
myuser@hostname:~$ systemctl list-units --type=service
```

# Exercises

### :pencil2: Run the registry service as a Linux service

Register AgroPilot's `registry-service` with systemd. You will start it without a terminal, read its logs, and stop it with `SIGTERM`.

1. Create `/etc/systemd/system/registry.service`:

```text
[Unit]
Description=AgroPilot registry-service
After=network.target

[Service]
WorkingDirectory=/home/myuser/ArgoPilot/services/registry-service
ExecStart=/home/myuser/ArgoPilot/services/registry-service/.venv/bin/uvicorn main:app --host 127.0.0.1 --port 8000
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Replace both paths with your own clone of ArgoPilot. `WorkingDirectory` is the folder that contains `main.py`. `ExecStart` is the `uvicorn` installed in that service's virtual environment.

`Restart=on-failure` tells systemd to start the process again if it crashes.

2. Tell systemd to read the new file:

```bash
sudo systemctl daemon-reload
```

3. Start it: `sudo systemctl start registry.service`
4. Check it: `sudo systemctl status registry.service`
5. Call the health route from another terminal: `curl http://127.0.0.1:8000/health`
6. Follow the log: `journalctl -fu registry.service`
7. Stop it: `sudo systemctl stop registry.service`

`stop` sends `SIGTERM`. The log shows uvicorn shutting down. The health URL stops answering.

Remove the service before you go back to running it by hand. A service that is still running keeps port 8000, and your next `uv run` will fail to bind.

```bash
sudo systemctl stop registry.service
sudo systemctl disable registry.service
sudo rm /etc/systemd/system/registry.service
sudo systemctl daemon-reload
```

### :pencil2: Foreground and background

A process you start from the terminal is in the **foreground** until it exits or you move it. The terminal waits, and it accepts no new command. A **background** process keeps running, and the terminal accepts the next command.

1. Run `sleep 600`. The terminal waits.
2. Press Ctrl+Z. The process suspends.
3. Run `fg`. The process is in the foreground again.
4. Press Ctrl+C to stop it.
5. Run `sleep 600 &`. The `&` starts it in the background. The shell prints a PID and gives you a prompt.
6. Run `fg` to bring it back.
7. Press Ctrl+C to stop it.

The registry service behaves the same way. `uv run uvicorn ...` holds the terminal. Ctrl+C stops it. You do not need to background it during the course.


[linux_process_state]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/linux_process_state.png
