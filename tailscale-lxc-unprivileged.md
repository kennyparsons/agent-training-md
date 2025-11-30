# Agent Training Guide: Configuring Tailscale in Unprivileged LXC Containers

This guide outlines the steps for an AI agent to configure Tailscale within an unprivileged LXC container. It assumes the agent has direct root shell access to the Proxmox host where the LXC container resides, and that the container has already been created (e.g., using the `proxmox-lxc-container.md` guide).

## 1. Prerequisites

-   An existing unprivileged LXC container. Let's assume its ID is `<VMID>`.
-   Direct root shell access to the Proxmox host.
-   The container should be in a stopped state before applying the configuration changes for `/dev/net/tun` access.

## 2. Enable TUN Device Access and Necessary Features

Tailscale requires access to a TUN device and some specific kernel features (`keyctl`, `nesting`) within the container. These configurations are applied directly on the Proxmox host using `pct` commands.

### 2.1. Stop the LXC Container
Configuration changes to device passthrough often require the container to be shut down and restarted.

**Action**: Stop the target LXC container.

**Command (Modifies system state on Proxmox host):**
```bash
pct stop <VMID>
```

### 2.2. Configure Device Passthrough for TUN and Features
Use the `pct set` command on the Proxmox host to grant the container access to `/dev/net/tun` and enable `keyctl` and `nesting` features.

**Action**: Apply the necessary `pct set` commands.

**Commands (Modifies system state on Proxmox host):**
```bash
pct set <VMID> --dev0 /dev/net/tun
pct set <VMID> --features keyctl=1,nesting=1
```
*Note: The `nesting=1` feature might already be set if the container was created using the `proxmox-lxc-container.md` guide, but it's safe to re-apply.*

### 2.3. Start the LXC Container
After applying the configuration changes, start the container to activate them.

**Action**: Start the target LXC container.

**Command (Modifies system state on Proxmox host):**
```bash
pct start <VMID>
```

### 2.4. Verify TUN Device
Once the container is running, verify that `/dev/net/tun` is available inside the container.

**Action**: Execute a command inside the container to check for the TUN device.

**Command (Read-only operation from Proxmox host, executed inside container):**
```bash
pct exec <VMID> -- ls -l /dev/net/tun
```
**Expected Output**: The command should return details about `/dev/net/tun`, indicating its presence (e.g., `crw-rw-rw- 1 root root 10, 200 Nov 29 12:00 /dev/net/tun`).

## 3. Install Tailscale Inside the Container

Once the TUN device is available, Tailscale can be installed using its standard Linux installation instructions. This example assumes a Debian/Ubuntu-based container.

**Action**: Install Tailscale within the LXC container.

**Commands (Modifies system state inside container, executed from Proxmox host):**
```bash
pct exec <VMID> -- apt update
pct exec <VMID> -- apt install -y curl
# IMPORTANT: Ensure the pipe | is executed INSIDE the container. 
# The best way is to download the script first, then run it.
pct exec <VMID> -- curl -fsSL https://tailscale.com/install.sh -o install.sh
pct exec <VMID> -- sh install.sh
```

## 4. Authenticate and Connect to Tailscale

After installation, authenticate the Tailscale client and connect to your tailnet.

**Action**: Run `tailscale up` to authenticate.

**Command (Modifies system state inside container, executed from Proxmox host):**
```bash
pct exec <VMID> -- tailscale up
```
**Expected Output**: The command will output an authentication URL directly in the live console output. The user should open this URL in a web browser to complete authentication. The `tailscale up` command will automatically exit with code `0` once authentication is successful.

**Agent Instruction to User**: "Please observe the live output for the Tailscale authentication URL. Open it in your web browser and log in to authenticate the container with your Tailnet. The command will complete automatically after successful login."

## 5. Verify Tailscale Status

After authentication, verify that Tailscale is connected and operational.

**Action**: Check the Tailscale status.

**Command (Read-only operation from Proxmox host, executed inside container):**
```bash
pct exec <VMID> -- tailscale status
```
**Expected Output**: The output should show the container's Tailscale IP address, peer connections, and that it is connected.

**Agent Action**: Analyze the output for any "Health check" warnings (e.g., `# - Some peers are advertising routes but --accept-routes is false`). If found, inform the user and ask if they would like to re-run `tailscale up` with the suggested flags (e.g., `--accept-routes`).

## Summary Checklist for Agents
1.  `pct stop <VMID>`
2.  `pct set <VMID> --dev0 /dev/net/tun`
3.  `pct set <VMID> --features keyctl=1,nesting=1`
4.  `pct start <VMID>`
5.  `pct exec <VMID> -- ls -l /dev/net/tun`
6.  `pct exec <VMID> -- apt update && pct exec <VMID> -- apt install -y curl && pct exec <VMID> -- curl -fsSL https://tailscale.com/install.sh | sh`
7. `pct exec <VMID> -- tailscale up` -> **User will interact via browser, command auto-exits on success**
8.  `pct exec <VMID> -- tailscale status`
