# Vagrant Sandbox

This repository contains an automated Vagrant configuration to set up a development virtual machine. It is designed to be flexible and portable, working seamlessly across different network environments (such as work or home).

---

## Features
* **Dynamic Network Modes**: Supports Host-Only (Private) and Bridge (Public, static, or DHCP) connections.
* **Idempotent Provisioning**: Automated and fast package installation that skips already-installed tools to save time.
* **SSH Key Integration**: Automatically injects host SSH keys into the VM for passwordless access (without crashing if keys are missing on the host).
* **Pre-installed Tools**:
  * Node.js (version 24 by default) and the latest npm.
  * Antigravity CLI (`agy`).
  * Utilities: Vim, Zsh, Htop, Curl, Nmap, Net-tools, etc.

---

## Prerequisites
Before starting, ensure you have the following installed on your host machine:
1. **VirtualBox** 7.0
2. **Vagrant** 2.4.2
3. An SSH key generated at `~/.ssh/id_ed25519` (the script will automatically inject it for passwordless access).

---

## Network Modes (NETWORK_MODE)

At the top of the `Vagrantfile`, you can modify the `NETWORK_MODE` variable to choose how the VM connects to the network:

| Mode | Value | Description | Recommended Use Case |
| :--- | :--- | :--- | :--- |
| **Private (Host-Only)** | `"private"` | Creates an isolated virtual network between your PC and the VM with a fixed IP (`192.168.56.10`). The VM accesses the internet via NAT. | **Recommended for daily use**. Works at home, work, and offline with no risk of IP conflicts. |
| **Public Static** | `"public_static"` | Connects the VM directly to the physical network (Bridge) with a configured static IP (`10.202.92.202`). | When other devices on the same local physical network need to access the VM. |
| **Public Dynamic** | `"public_dhcp"` | Connects the VM to the physical network (Bridge) and requests a dynamic IP from the local router via DHCP. | When you need a public connection but are in an environment where IPs are assigned dynamically. |

---

## Getting Started

1. Navigate to the repository directory:
   ```bash
   cd ~/repos/vagrant-file
   ```

2. Start the virtual machine:
   ```bash
   vagrant up
   ```

3. Access the virtual machine via SSH:
   ```bash
   vagrant ssh
   ```
   *(Or connect directly via standard SSH using the configured IP: `ssh user@192.168.56.10`)*

   For convenience, add this entry to your `~/.ssh/config` and connect with just `ssh vm-ubuntu`:
   ```ssh-config
   # Vagrant VM - host-only network
   Host vm-ubuntu
       HostName 192.168.56.10
       User user
       StrictHostKeyChecking accept-new
   ```

4. To stop, restart, or update the machine:
   ```bash
   vagrant halt       # Powers off the VM
   vagrant reload     # Restarts and applies Vagrantfile network changes
   vagrant provision  # Re-runs shell provisioning (installs packages/updates)
   ```

---

## Customization (Vagrantfile)

You can customize the VM by modifying the configuration variables at the top of the `Vagrantfile`:

* `BOX_IMAGE`: The base box image to use (default: `"ubuntu/jammy64"`).
* `CPUs`: The number of CPU cores allocated to the VM (default: `8`).
* `MEMORY`: The amount of RAM in MB (default: `15890` ~16GB).
* `USERNAME` / `PASSWORD`: The default user created inside the VM (default: `user` / `pass`).
* `VM_PRIVATE_IP`: The static IP used in `"private"` mode (default: `192.168.56.10`).
* `VM_PUBLIC_IP`: The static IP used in `"public_static"` mode (default: `10.202.92.202`).
* `NETWORK_INTERFACE_PREFIX`: The prefix of your host's physical network interface for Bridge modes (e.g., `"wlp"` for Wi-Fi, `"en"` for ethernet).

---

## Recommended Aliases (Zsh)

To manage the virtual machine from any directory in your terminal, you can add these aliases to your `~/.zshrc`:

```bash
alias vm-status='cd ~/repos/vagrant-file && vagrant status && cd -'
alias vm-up='cd ~/repos/vagrant-file && vagrant up && cd -'
alias vm-halt='cd ~/repos/vagrant-file && vagrant halt && cd -'
alias vm-destroy='cd ~/repos/vagrant-file && vagrant destroy && cd -'
```

*Remember to run `source ~/.zshrc` in your terminal after adding them to apply the changes.*
