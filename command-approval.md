# Command Approval Guidelines

This document outlines the guidelines for the AI agent regarding command execution, specifically differentiating between read-only and state-changing (CRUD) operations, and the required approval process.

## 1. Read-Only Commands

**Definition**: Commands that retrieve information from the system without altering its state. These include, but are not limited to, `ls`, `cat`, `grep`, `pvesh get`, `pvesm status`, `ip -o link show`.

**Approval Requirement**: None. The agent can execute these commands without explicit user permission.

## 2. State-Changing (CRUD) Commands

**Definition**: Commands that modify the file system, codebase, processes, services, or any other aspect of the system state. These include, but are not limited to, `write_file`, `replace`, `run_shell_command` when used for:
- Creating files or directories (`mkdir`, `touch`)
- Modifying files (`echo ... > file`, `sed -i`, `mv`, `rm`)
- Starting, stopping, or destroying services/containers (`systemctl start`, `pct create`, `pct stop`, `pct destroy`)
- Downloading or updating software (`pveam update`, `apt install`, `npm install`)
- Any command that results in persistent changes to the environment.

**Approval Requirement**: **Explicit User Approval is REQUIRED.**

### How the Agent Will Request Approval:

Before executing any state-changing command, the agent will:
1.  State the command it intends to run.
2.  Provide a brief description of the command's purpose and potential impact.
3.  Explicitly ask for permission to run the command, halting its operation until a positive approval response is received from the user.

**Example Approval Request:**

`<start of message>`

> `rcs -h <host> -- pct create <VMID> ...`

This command will create a new LXC container, which modifies the system state of the Proxmox host.

Do you approve running this command?
`<end of message>`

## 3. Interaction and Response

-   The user must reply with a clear, positive affirmation (e.g., "yes", "approve", "go ahead") for state-changing commands to proceed.
-   If the user declines or provides a non-affirmative response, the agent will *not* execute the command and will await further instructions.
-   The agent will *not* execute a previously cancelled command unless explicitly asked to do so again in a subsequent prompt.

By adhering to these guidelines, we ensure transparency and user control over critical system modifications.
