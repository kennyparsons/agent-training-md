# Agent Training Guide: Managing SSH Known Hosts

This guide instructs the AI agent on how to manage the `known_hosts` file, specifically for clearing old keys and adding new keys for a remote host. This is essential when a remote host's identity changes (e.g., re-installing a server, recreating a container) or when connecting to a new host for the first time non-interactively.

## 1. Discovery Phase (Read-Only)

Before modifying the `known_hosts` file, you must identify the target host's IP address or hostname and verify if it is currently reachable.

### 1.1. Identify Target Host
Define the IP address or hostname you need to manage.
**Variable:** `<TARGET_HOST>` (e.g., `192.168.2.60` or `my-server.local`)

### 1.2. Verify Reachability
Ensure the host is online before attempting to scan its keys.

**Command:**
```bash
ping -c 1 <TARGET_HOST>
```

**Output Analysis:**
-   If `1 packet transmitted, 1 received`, proceed.
-   If unreachable, do not proceed with `ssh-keyscan`.

## 2. Definition Phase (Preparation)

Ensure you have the following variables:
-   `<TARGET_HOST>`: The IP/hostname from Phase 1.
-   `<KNOWN_HOSTS_FILE>`: Typically `~/.ssh/known_hosts` (or `/home/user/.ssh/known_hosts`).

## 3. Execution Phase (State-Changing)

This phase involves two steps: removing the old key (if any) and adding the new key.

### 3.1. Remove Old Host Key (If applicable)
If the host key has changed (e.g., "WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!"), you must remove the old entry first.

**Command (Modifies local file system):**
```bash
ssh-keygen -R <TARGET_HOST>
```

**Action:** This command updates `~/.ssh/known_hosts` by commenting out or removing the line associated with `<TARGET_HOST>`. It also creates a backup file `known_hosts.old`.

### 3.2. Scan and Add New Host Key
To allow future non-interactive SSH connections, you must scan the remote host's public key and append it to your `known_hosts` file.

**Command (Modifies local file system):**
```bash
ssh-keyscan -H <TARGET_HOST> >> ~/.ssh/known_hosts
```

*Breakdown:*
-   `ssh-keyscan`: Connects to the host and prints its public keys.
-   `-H`: Hashes the hostname in the output (optional, but good practice if your known_hosts is hashed).
-   `>> ~/.ssh/known_hosts`: Appends the output to your known hosts file.

## 4. Verification Phase (Read-Only)

Verify that the key has been added and that you can connect (or at least initiate a connection) without a host key verification error.

### 4.1. Check for Key Entry
Verify the key was added to the file.

**Command:**
```bash
grep <TARGET_HOST> ~/.ssh/known_hosts
```
*(Note: If `-H` was used in `ssh-keyscan`, simple grep might not work. In that case, rely on step 4.2)*

### 4.2. Test SSH Connection
Attempt a connection. If the key is correct, you should *not* see "authenticity of host... can't be established" or "WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!".

**Command:**
```bash
ssh -o BatchMode=yes -o ConnectTimeout=5 <TARGET_HOST> "hostname"
```

**Output Analysis:**
-   **Success:** Prints the hostname (or asks for password if keys aren't set up, but *doesn't* complain about the host key).
-   **Failure:** "Host key verification failed" (means step 3.1 or 3.2 failed).

## 5. Cleanup Phase (State-Changing)

Optionally remove the backup file created by `ssh-keygen -R`.

**Command:**
```bash
rm ~/.ssh/known_hosts.old
```

## Summary Checklist for Agents

1.  Identify `<TARGET_HOST>`.
2.  `ping -c 1 <TARGET_HOST>` (Verify online).
3.  `ssh-keygen -R <TARGET_HOST>` (Remove old key).
4.  `ssh-keyscan -H <TARGET_HOST> >> ~/.ssh/known_hosts` (Add new key).
5.  `ssh ... <TARGET_HOST> "hostname"` (Verify connection).
