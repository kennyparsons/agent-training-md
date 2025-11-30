# Agent Training Guide: Building Instruction Sets

This document serves as a meta-instruction set for the AI agent. Its purpose is to define the standard, structure, and philosophy for creating *new* agent training documents (e.g., `proxmox-lxc-container.md`, `tailscale-lxc-unprivileged.md`).

When asked to "create an instruction file" or "write a guide for [task]," follow these principles.

## 1. Core Philosophy: "Soup to Nuts"

Your instructions must be exhaustive. **Assume nothing.**
-   Do not assume network interface names (e.g., `eth0` vs `ens18`).
-   Do not assume storage IDs (e.g., `local-lvm` vs `zfs-pool`).
-   Do not assume software is installed (check for it).
-   Do not assume specific IDs (find the next available one).

**Goal:** An agent following your guide should be able to complete the task on a completely alien system without needing to ask the user for basic configuration parameters.

## 2. Standard Document Structure

Every instruction file must follow this logical flow:

### Phase 1: Discovery (Read-Only)
Before changing anything, the agent must inspect the environment to gather variables.
-   **Goal:** Populate variables like `<TARGET_ID>`, `<STORAGE_PATH>`, `<INTERFACE_NAME>`.
-   **Method:** Use specific commands to list/get status.
-   **Example:** "Run `pvesh get /cluster/nextid` to define `<VMID>`."

### Phase 2: Definition (Preparation)
Explicitly list the variables that have been gathered or need to be defined.
-   **Method:** Create a list of "Configuration Variables" based on Phase 1.
-   **Example:** "Ensure you have values for: `<VMID>`, `<TEMPLATE>`, `<PASSWORD>`."

### Phase 3: Execution (State-Changing)
The actual work.
-   **Granularity:** Break complex pipes into separate steps if there is a risk of execution context issues (e.g., piping `curl` to `sh`).
-   **Safety:** Clearly mark commands that modify system state.
-   **Commands:** Provide the *exact* command strings. Use placeholders like `<VMID>` clearly.

### Phase 4: Verification (Read-Only)
Did it work?
-   **Method:** Run commands to check the status, exit code, or file existence.
-   **Example:** "Run `systemctl status service_name` and check for 'active (running)'."

### Phase 5: Cleanup (State-Changing)
Remove any temporary files, keys, or scripts created during the process.

## 3. Drafting Rules

### 3.1. Variable Discovery
Never hardcode values. Always provide the command to find them.
*   *Bad:* "Use `vmbr0`."
*   *Good:* "Run `ip -o link show type bridge` to find the bridge name. Let's call this `<BRIDGE>`."

### 3.2. Command Precision
*   **Flags:** Use full flags (e.g., `--password` instead of `-p`) for clarity unless the tool only supports short flags.
*   **Pipes:** Be extremely careful with pipes (`|`) and redirects (`>`). Ensure the instructions clarify *where* these execute (inside a container vs. on the host).
*   **Paths:** Use absolute paths (e.g., `/usr/bin/tailscale`, `/tmp/file.tmp`) to avoid `$PATH` issues.

### 3.3. Interaction & Output
*   If a command requires user interaction (like a browser login), explicitly state: "**User Action Required**."
*   If a command outputs data the user needs (like an IP address or URL), include a step to retrieve and display it.

### 3.4. Tool Agnosticism (Context Dependent)
Unless strictly specified (like "use rcs"), write the commands as standard Shell/Bash commands.
*   The *agent* reading the guide will know how to wrap these commands in their specific toolset (e.g., `rcs`, `ssh`, `kubectl`).
*   *Exception:* If the guide is *specifically* about using a tool (like `rcs.md`), then be specific.

## 4. The Validation Protocol (Test-Driven Documentation)

If you are currently connected to an environment where you can test the instructions (and you have permission):

1.  **Draft** the initial plan.
2.  **Execute** the Discovery commands to ensure they produce the expected output format.
3.  **Execute** the Creation commands (with explicit user approval).
4.  **Observe** errors (e.g., "unknown flag", "command not found").
5.  **Refine** the instruction file immediately based on the errors.
6.  **Verify** the final state.
7.  **Finalize** the markdown file.

## 5. Markdown Style Guide

-   Use `#` for the Document Title.
-   Use `##` for Major Phases.
-   Use `###` for Sub-steps.
-   Use `**Bold**` for Actions and Warnings.
-   Use `code blocks` for all commands.
-   Keep explanatory text concise.

## 6. Example Template

```markdown
# [Task Name]

## 1. Discovery
Find the necessary IDs and paths.
Command: `some-discovery-command`
Variable: `<VAR_NAME>`

## 2. Execution
Perform the task.
Command: `do-the-thing --id <VAR_NAME>`

## 3. Verification
Check the result.
Command: `check-status <VAR_NAME>`
```
