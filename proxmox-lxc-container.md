# Proxmox LXC Container Creation Guide for Agents

This guide provides a comprehensive, step-by-step procedure for an automated agent to create an LXC container on a Proxmox VE host via a root shell. It assumes zero prior knowledge of the specific environment configuration (storage IDs, network bridges, available templates, etc.).

## 1. Discovery Phase

Before creating a container, you must discover the environment's specific resources and constraints.

### 1.1. Find the Next Available VMID
Proxmox requires a unique ID for every container. Do not guess.

**Command:**
```bash
pvesh get /cluster/nextid
```

**Action:** capture the integer output. Let's call this `<VMID>`.

### 1.2. Identify Valid Storage for Containers
You need to find a storage identifier (e.g., `local-lvm`, `local-zfs`) that supports container root filesystems (`rootdir`).

**Command:**
```bash
pvesm status -content rootdir
```

**Output Analysis:**
- Look for entries where `Status` is `active`.
- Note the `Name` (first column).
- **Decision Logic:** Prefer `local-lvm` or `local-zfs` if available for better performance. If not, use any active storage listed. Let's call this `<STORAGE_ID>`.

### 1.3. Identify Network Bridges
You need to attach the container's network interface to a host bridge.

**Command:**
```bash
ip -o link show type bridge
```
*Alternatively, if `brctl` is installed:* `brctl show`

**Output Analysis:**
- Look for standard bridges like `vmbr0`, `vmbr1`.
- **Decision Logic:** Default to `vmbr0` if present, as it is the standard WAN/LAN bridge. Let's call this `<BRIDGE>`.

### 1.4. Identify a Container Template

LXC containers require a template (e.g., Ubuntu, Debian, Alpine).

**Step A: List templates already downloaded**
You need to check if a template exists on your chosen storage or any storage holding templates (`vztmpl`).

```bash
pvesm status -content "vztmpl"
```
For each storage ID found that supports `vztmpl` (often `local`):
```bash
pvesm list <STORAGE_ID_FOR_TEMPLATES> --content "vztmpl"
```

**Action:** If a suitable template is found, capture its full volume ID (e.g., `local:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst`). Let's call this `<TEMPLATE_VOLID>`.

**Step B: Download a template (if none suitable exist)**
If no suitable template is found in Step A:
1.  **Update the list of available remote templates (Modifies system state)**:
    ```bash
    pveam update
    ```
2.  List available system templates:
    ```bash
    pveam available --section system
    ```
3.  Download a chosen template (e.g., `ubuntu-22.04-standard`):
    ```bash
    pveam download <STORAGE_ID_FOR_TEMPLATES> <TEMPLATE_NAME>
    ```

**Action:** After downloading, verify with `pvesm list <STORAGE_ID_FOR_TEMPLATES> --content "vztmpl"` and capture the full volume ID. Let's call this `<TEMPLATE_VOLID>`..

## 2. Construction Phase

Now that you have gathered all variables, construct the creation command.

### 2.1. Define Configuration Variables
Ensure you have values for:
- `<VMID>` (from 1.1)
- `<TEMPLATE_VOLID>` (from 1.4)
- `<STORAGE_ID>` (from 1.2 - for the root disk)
- `<BRIDGE>` (from 1.3)
- `<HOSTNAME>` (Generate a name, e.g., `container-<VMID>`)
- `<PASSWORD>` (Read from `lxc.pass` via `rcs --env LXC_PASSWORD="$(cat lxc.pass)" --shell`)

### 2.1.1. Prepare SSH Public Key (If required)
If you need to add an SSH public key to the root user of the container during creation, define it here:

**Public Key:**
```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDeD+i+rk0bBZhU1shKvxoHP1fDs+M+KVtCtf5vDDZelQs1X55C6SIXqmSDdKVA/VuYbXk0lmVVEkxc+vOdn665KuSbKMQQj4eSQdoJxFyyQQw36kdWY/tIgF+tBpD3SbeSNL/u54q0vI0VeuP4F7d05Vd7o535BfOak2iKzMr+nIxXDNQkJTIfGbpDbHEGniY/a+Y45GM8908XcnP/hRYNksO1pyvqx/7wRMrYynD+MOIYtqFOdFx2ChzBj5tvKQfTrZmEG+YEWxCyXpQdQ1XTzHeOV7WNTWVwSZmXfG1pbwy3VGnecllW+RGb01fnadfXYYc+W0DrICILGO/fgBVz
```
**Action**: The agent must write this key to a temporary file on the Proxmox host.

**Command (Modifies system state on Proxmox host):**
```bash
rcs -h <PROXMOX_HOST_IP> --shell -- 'echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDeD+i+rk0bBZhU1shKvxoHP1fDs+M+KVtCtf5vDDZelQs1X55C6SIXqmSDdKVA/VuYbXk0lmVVEkxc+vOdn665KuSbKMQQj4eSQdoJxFyyQQw36kdWY/tIgF+tBpD3SbeSNL/u54q0vI0VeuP4F7d05Vd7o535BfOak2iKzMr+nIxXDNQkJTIfGbpDbHEGniY/a+Y45GM8908XcnP/hRYNksO1pyvqx/7wRMrYynD+MOIYtqFOdFx2ChzBj5tvKQfTrZmEG+YEWxCyXpQdQ1XTzHeOV7WNTWVwSZmXfG1pbwy3VGnecllW+RGb01fnadfXYYc+W0DrICILGO/fgBVz" > /tmp/lxc_pubkey.tmp'
```
Let's call the file path `<PUBLIC_KEY_FILE_ON_HOST>`.

### 2.2. Execute Creation Command
Use `pct create`. The following flags are standard defaults but should be adjusted if the user request specifies otherwise.

**Command Structure:**
```bash
rcs -h <PROXMOX_HOST_IP> --env LXC_PASSWORD="$(cat lxc.pass)" --shell -- \
  'pct create <VMID> <TEMPLATE_VOLID> \\
  --arch amd64 \\
  --hostname <HOSTNAME> \\
  --cores 1 \\
  --memory 512 \\
  --swap 0 \\
  --net0 name=eth0,bridge=<BRIDGE>,ip=dhcp,type=veth \\
  --rootfs <STORAGE_ID>:2 \\
  --onboot 1 \\
  --password $LXC_PASSWORD \\
  --ssh-public-keys <PUBLIC_KEY_FILE_ON_HOST> \\
  --features nesting=1 \\
  --unprivileged 1'
```

*Breakdown of flags:*
- `--rootfs <STORAGE_ID>:2`: Creates a 2GB root disk on the discovered storage.
- `--net0 ... ip=dhcp`: Configures IPv4 via DHCP. Change `dhcp` to a static IP (e.g., `192.168.1.50/24`) if requested.
- `--features nesting=1`: Useful for running Docker/Systemd inside the container.
- `--unprivileged 1`: Security best practice.

### 2.3. Verification
Check if the creation was successful.

**Command:**
```bash
pct config <VMID>
```
If this returns the config, the container exists.

## 3. Operational Phase

### 3.1. Start the Container
```bash
pct start <VMID>
```

### 3.2. Check Status
```bash
pct status <VMID>
```
Wait until status is `running`.

### 3.3. Retrieve Network Info (Optional)
If you used DHCP, you might need to find the assigned IP.
```bash
lxc-info -n <VMID> -iH
```
*Note: This usually works better if `lxc-net` is active, but on Proxmox, parsing `ip` inside the container is more reliable:*

```bash
pct exec <VMID> -- ip -4 -o addr show eth0
```

### 3.4. Enable Root SSH Login (Optional)
To explicitly allow SSH login for the root user, modify the `sshd_config` file inside the container and restart the SSH service.

**Action**: Use `pct exec` to modify the configuration file and restart SSH.

**Command (Modifies system state inside container):**
```bash
rcs -h <PROXMOX_HOST_IP> --shell -- '''pct exec <VMID> -- sed -i \"s/^#PermitRootLogin.*/PermitRootLogin prohibit-password/\" /etc/ssh/sshd_config'''
rcs -h <PROXMOX_HOST_IP> --shell -- '''pct exec <VMID> -- systemctl restart ssh'''
```

### 3.5. Clean up Temporary Public Key File (Optional, but recommended)
If a temporary public key file was created on the Proxmox host, it should be removed after container creation and verification.

**Command (Modifies system state on Proxmox host):**
```bash
rcs -h <PROXMOX_HOST_IP> -- rm /tmp/lxc_pubkey.tmp
```

### 3.6. Print Container Hostname and IP Address
To confirm the container's network identity after setup, retrieve its hostname and IP address.

**Action**: Use `pct exec` to get the hostname and IP address from within the container.

**Command (Read-only operation):**
```bash
rcs -h <PROXMOX_HOST_IP> --shell -- '''pct exec <VMID> -- hostname && pct exec <VMID> -- ip -4 -o addr show eth0'''
```

## 4. Troubleshooting / Error Handling

- **Error: "storage does not support content type 'rootdir'"**: You selected the wrong `<STORAGE_ID>`. Re-run discovery (1.2).
- **Error: "bridge 'vmbr0' does not exist"**: You assumed the bridge name. Re-run discovery (1.3).
- **Error: "template not found"**: Ensure you ran `pveam download` successfully or used the exact Volume ID string returned by `pvesm list`.

## 5. Summary Checklist for Agents
1. `pvesh get /cluster/nextid` -> VMID
2. `pvesm status -content rootdir` -> STORAGE
3. `ip link show type bridge` -> BRIDGE
4. `pvesm status -content vztmpl` & `pvesm list <STORAGE_ID_FOR_TEMPLATES> --content "vztmpl"` (and `pveam update`/`pveam available`/`pveam download` if needed) -> TEMPLATE
5. `pct create ...`
6. `pct start ...`
