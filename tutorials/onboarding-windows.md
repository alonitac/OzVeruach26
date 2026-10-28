# Environment Setup (Windows)

Read the below instructions carefully and follow them step by step to set up your local working environment.

## Step I: Command-line Terminal Environment

During the course, you'll build and manage apps in a Linux environment (specifically, Ubuntu virtual machines on AWS).

The Linux Terminal will be your best friend. You'll use it to run commands, troubleshoot issues, and interact with servers.

To match the course environment as closely as possible, set up **Ubuntu via WSL2 (Windows Subsystem for Linux)**, which provides a fully functional Ubuntu environment inside Windows.

1. Install WSL.
   Open PowerShell as Administrator and run:

   ```powershell
   wsl --install
   ```

   This installs Ubuntu by default. If not, see the manual install guide:
   https://learn.microsoft.com/en-us/windows/wsl/install

2. Update WSL to Version 2 if needed.

   ```powershell
   wsl --set-default-version 2
   ```

3. Launch an Ubuntu terminal by searching for Ubuntu in the **Start** menu.

Installing WSL is the only step that uses PowerShell. **After Ubuntu is installed, run every later command in the Ubuntu terminal**.

This Ubuntu terminal is the one you use for the whole course. Your command prompt should look like `yourname@computer:~$`. If a prompt starts with `PS`, you are in PowerShell. 

## Step II: Python Interpreter

We'll be using Python as the primary programming language, version 3.12 or higher.

Your system may already have Python installed, but we want to make sure it is the right version and that you also have `pip` and virtual environment tools available.

In your Ubuntu (WSL) terminal, run:

```bash
sudo apt update
sudo apt install -y python3 python3-pip python3-venv
```

## Step III: Git

Git is how you save the history of your work and share it.

Stay in the Ubuntu (WSL) terminal. Install Git there, so Git is part of the same Linux environment you will use in the course:

```bash
sudo apt install -y git
```

Then set your name and email. Run both commands in this same Ubuntu terminal. `--global` saves these settings for every repository inside Ubuntu, so you only do this once. You are free to choose any name and email you like, but we recommend using the name and email on your GitHub account.

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

Replace `Your Name` and `you@example.com` with your own details. Keep the quotes.


## Step IV: IDE (Integrated Development Environment)

The last step is to set up an IDE.
You will use it extensively to write, run, and debug your code.

You are free to use any IDE you like, but we recommend VS Code, it's lightweight, fast, and works well on Linux, Windows, and macOS.

You'll install VS Code **on Windows**, with a Remote - WSL extension to connect to the Ubuntu terminal.
That way you run VS Code in Windows (more convenient), but the terminal inside VS Code and the applications you run are in Ubuntu.

1. Download and install VS Code:
   https://code.visualstudio.com/Download
2. Install the Remote - WSL extension:
   `ms-vscode-remote.remote-wsl`
3. Install the Python extension:
   `ms-python.python`
4. Install the Python Debugger extension:
   `ms-python.debugpy`
5. Install Claude Code for VS Code:
   `anthropic.claude-code`
6. Install GitHub Copilot Chat, if you do not already have it:
   `GitHub.copilot-chat`

Open the Extensions view with `Ctrl+Shift+X`, search for the name, and click Install. When VS Code asks you to sign in to Claude or GitHub, sign in with the account you will use for the course.

### Use Ubuntu as the VS Code terminal

The terminal inside VS Code must be Ubuntu on WSL. Make that the default, so new terminals open in Linux, which is where you develop in this course.

1. Open the Command Palette (`Ctrl+Shift+P`).
2. Run `Terminal: Select Default Profile`.
3. Choose `Ubuntu (WSL)`. If the list only shows `WSL`, choose that.
4. Open a new terminal from **Terminal** → **New Terminal**. The keyboard shortcut is Ctrl and the backtick key (the key above Tab).
5. Confirm the prompt looks like `yourname@computer:~$`.


> [!TIP] Turn on Auto Save from **File** -> **Auto Save**.
