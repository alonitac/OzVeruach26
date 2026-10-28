# Environment Setup (Linux)

Read the below instructions carefully and follow them step by step to set up your local working environment.

## Step I: Command-line Terminal Environment

During the course, you'll build and manage apps in a Linux environment (specifically, Ubuntu virtual machines on AWS).

The Linux Terminal will be your best friend. You'll use it to run commands, troubleshoot issues, and interact with servers.

Ubuntu is the recommended distribution, but other distros will work too as long as you're comfortable using the terminal.

To open a terminal session, press `Ctrl + Alt + T` or search for Terminal in your applications.

## Step II: Python Interpreter

We'll be using Python as the primary programming language, version 3.12 or higher.

Your system may already have Python installed, but we want to make sure it is the right version and that you also have `pip` and virtual environment tools available.

In your opened Terminal window, run:

```bash
sudo apt update
sudo apt install -y python3 python3-pip python3-venv
```

## Step III: Git

Git is how you save the history of your work and share it.

In the same Terminal window, install Git:

```bash
sudo apt install -y git
```

Then set your name and email. Run both commands. `--global` saves these settings for every repository on this computer, so you only do this once. You are free to choose any name and email you like, but we recommend using the name and email on your GitHub account.

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

Replace `Your Name` and `you@example.com` with your own details. Keep the quotes.


## Step IV: IDE (Integrated Development Environment)

The last step is to set up an IDE.
You will use it extensively to write, run, and debug your code.

You are free to use any IDE you like, but we recommend VS Code, it's lightweight, fast, and works well on Linux, Windows, and macOS.

Download and install VS Code:
   https://code.visualstudio.com/Download

> [!TIP] 
> Turn on Auto Save from **File** -> **Auto Save**.

### Extensions

Install these extensions in VS Code. Open the Extensions view with `Ctrl+Shift+X`, search for the name, and click Install.

1. Python:
   `ms-python.python`
2. Python Debugger:
   `ms-python.debugpy`
3. Claude Code for VS Code:
   `anthropic.claude-code`
4. GitHub Copilot Chat, if you do not already have it:
   `GitHub.copilot-chat`

When VS Code asks you to sign in to Claude or GitHub, sign in with the account you will use for the course.
