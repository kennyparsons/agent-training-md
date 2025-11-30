# Agent Training Guide: Clipboard Interaction

This document instructs the AI agent on how to place text or commands directly into the user's clipboard. This is highly useful for providing long commands, complex URLs, or configuration snippets that the user needs to paste elsewhere (e.g., into a separate terminal window, browser, or GUI application).

## 1. Core Concept

Instead of just printing a command block and expecting the user to select and copy it, you can programmatically inject text into their system clipboard. This reduces friction and copy-paste errors.

## 2. Command Reference

The command depends on the user's operating system.

### macOS (Darwin) - **Primary**
Use the `pbcopy` utility.

```bash
echo "text to copy" | pbcopy
```

### Linux
Use `xclip` or `xsel` (if available).

```bash
# Using xclip (selection clipboard)
echo "text to copy" | xclip -selection clipboard

# Using xsel
echo "text to copy" | xsel --clipboard --input
```

## 3. Workflow Protocol

When you determine that placing text in the clipboard is helpful:

1.  **Construct the Payload**: Ensure the text is exactly what the user needs (no extra whitespace or markdown formatting unless intended).
2.  **Execute the Command**: Run the appropriate shell command (e.g., `echo "..." | pbcopy`).
    *   *Note*: This is generally considered a **safe/read-only** operation in terms of system stability, but always ensure you aren't overwriting something critical in the clipboard without context.
3.  **Inform the User**: **Crucial Step**. You must explicitly tell the user:
    *   **What** was copied (e.g., "I have copied the `pct enter` command...").
    *   **Where** it is (e.g., "...to your clipboard.").
    *   **Where to run it/paste it** (e.g., "Paste this into your local terminal" or "Paste this into the Proxmox shell").
    *   **What to do next** (e.g., "Run it to verify the key.").

## 4. Examples

### Example A: Providing a Complex Command
**Scenario**: The user needs to run a complex `rcs` command manually to debug.

**Agent Action**:
```bash
echo "rcs -h 192.168.1.100 --shell -- 'tail -f /var/log/syslog | grep error'" | pbcopy
```

**Agent Response**:
"I have copied the debug command to your clipboard. Please paste it into your terminal to monitor the logs."

### Example B: Providing a URL
**Scenario**: The user needs to open a specific documentation page.

**Agent Action**:
```bash
echo "https://pve.proxmox.com/wiki/Linux_Container" | pbcopy
```

**Agent Response**:
"I have copied the documentation URL to your clipboard. You can paste it into your web browser."
