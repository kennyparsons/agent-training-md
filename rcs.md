# Agent Training Guide: Using `rcs` for Remote Execution

This document provides a comprehensive guide for an AI agent on how to effectively use the `rcs` command-line tool for remote command execution over SSH.

## 1. Core Purpose

`rcs` is a tool that makes executing commands on a remote server feel like running them locally. For an AI agent, it provides a reliable, scriptable, and predictable interface for interacting with remote environments.

**Primary Goal**: Execute a command on a remote host and get the output, errors, and exit code back, just as if you had run it on the local machine.

## 2. Key Concepts

There are two fundamental modes of operation that you must understand to use the tool correctly.

### A. `argv` Mode (Default)

This is the **safest and most predictable** mode. It should be your default choice.

- **How it works**: Each argument after `--` is passed *literally* to the remote system for direct execution. There is **no shell** involved on the remote end to interpret the command.
- **When to use it**: Use this for any command that does not require shell features like pipes (`|`), redirection (`>`), or globbing (`*`).
- **Example**:
  ```sh
  # Correct: Runs 'ls' with two arguments '-la' and '/var/log'
  rcs -h my-server -- ls -la /var/log

  # Correct: Handles arguments with spaces correctly
  rcs -h my-server -- ./my-script --user "John Doe"
  ```

### B. `shell` Mode (`--shell`)

This mode is powerful but should be used **only when necessary**.

- **How it works**: The entire command string after `--` is wrapped and executed by a remote login shell (`bash -lc '...'`).
- **When to use it**: Use this **only** when your command contains shell-specific syntax:
    - Pipes: `|`
    - Redirection: `>`, `>>`, `<`
    - Command Chaining: `&&`, `||`
    - Globbing/Wildcards: `*`, `?`
    - Variable Expansion: `$HOME`, `$USER` (when you want the *remote* shell to expand it)
- **Example**:
  ```sh
  # Correct: Needs a shell to handle the pipe '|'
  rcs -h my-server --shell -- 'tail -n 50 /var/log/syslog | grep "CRON"'

  # Correct: Needs a shell to handle output redirection '>'
  rcs -h my-server --shell -- 'echo "hello" > /tmp/testfile'
  ```

**Agent Decision Rule**: Start with `argv` mode. Analyze the command string. If it contains characters like `|`, `>`, `*`, `&&`, switch to `--shell` mode.

## 3. Essential Flags

These flags control the behavior of the remote session.

| Flag      | Purpose                                                              | When to Use                                                                                             |
|-----------|----------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| `-h`      | Specifies the SSH host or alias to connect to.                       | **Always required**, unless a default is set in `project.toml`.                                         |
| `--dir`   | Sets the working directory on the remote host before execution.      | When the command needs to run in a specific location (e.g., a project or log directory).                |
| `--stdin` | Streams local standard input to the remote command.                  | When you are piping data *into* `rcs` (e.g., `cat local.file | rcs ...`).                           |
| `--pty`   | Allocates a pseudo-terminal (PTY).                                   | For interactive, screen-based applications like `vim`, `nano`, `top`, `htop`, `less`, or any command that draws a user interface. |
| `--env`   | Sets an environment variable on the remote host.                     | When the remote command depends on a specific environment variable.                                     |

## 4. Interactive Sessions (A Critical Skill)

To interact with a program that requires keyboard input beyond a simple stream (e.g., using arrow keys in `vim`, pressing `q` to quit `top`), you **must use both `--pty` and `--stdin`**.

- `--pty`: Creates the "screen" for the application to draw on.
- `--stdin`: Connects your "keyboard" to the application.

**Example**:
```sh
# To edit a file interactively
rcs -h my-server --pty --stdin -- vim /etc/hosts

# To view a process list interactively
rcs -h my-server --pty --stdin -- top
```

## 5. Configuration File (`project.toml`)

`rcs` can load default settings from a `project.toml` file.

- **Purpose**: To avoid repeating common flags like `-h my-server` for a specific project.
- **Precedence**: **Command-line flags always override settings from the config file.**
- **Example `project.toml`**:
  ```toml
  [ssh]
  host = "dev-server"

  [exec]
  dir = "/home/agent/app"

  [env]
  DEFAULT_VAR = "set_from_config"
  ```
- **Usage with the above config**:
  ```sh
  # No -h or --dir needed; they come from the file. Runs 'ls -la' in '/home/agent/app' on 'dev-server'.
  rcs -- ls -la
  ```

## 6. Common Patterns for Agents

| Task                                    | Command                                                                              | Mode    | Explanation                                                                                             |
|-----------------------------------------|--------------------------------------------------------------------------------------|---------|---------------------------------------------------------------------------------------------------------|
| List files in a directory               | `rcs -h host -- ls -la /path/to/dir`                                              | `argv`  | Simple, direct command execution.                                                                       |
| Search for text in a remote file        | `rcs -h host -- grep "ERROR" /var/log/app.log`                                    | `argv`  | `grep` is a standard command; no shell is needed.                                                       |
| Search logs using a pipe                | `rcs -h host --shell -- 'dmesg | grep "memory error"'`                             | `shell` | The pipe (`|`) requires the shell for interpretation.                                                   |
| Create a remote file from local data    | `echo "some data" | rcs -h host --stdin -- 'cat > /tmp/data.txt'`                    | `shell` | `--stdin` is needed for the pipe. `cat > ...` is a shell feature, so `--shell` is also required.        |
| Run a deployment script                 | `rcs -h host --dir /srv/app -- ./deploy.sh --env production`                       | `argv`  | A script with arguments. `argv` mode handles this perfectly.                                            |
| Edit a configuration file               | `rcs -h host --pty --stdin -- nano /etc/nginx/sites-available/default`             | `argv`  | Interactive session. Requires both `--pty` and `--stdin`.                                               |
| Check remote environment variables      | `rcs -h host --env FOO=bar --shell -- 'env | grep FOO'`                            | `shell` | `--env` sets the variable. The pipe in the command requires `--shell`.                                  |

By following these guidelines, an agent can reliably use `rcs` to perform complex remote operations. The key is to correctly identify whether a command requires a shell and whether the session needs to be interactive.
