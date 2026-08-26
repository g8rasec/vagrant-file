# Vagrant Sandbox

This repository contains an automated Vagrant configuration to set up a development virtual machine. It is designed to be flexible and portable, working seamlessly across different network environments (such as work or home).

---

## Features
* **Dynamic Network Modes**: Supports Host-Only (Private) and Bridge (Public, static, or DHCP) connections.
* **Idempotent Provisioning**: Automated and fast package installation that skips already-installed tools to save time.
* **SSH Key Integration**: Automatically injects host SSH keys into the VM for passwordless access (without crashing if keys are missing on the host).
* **Host Terminfo Sync**: Captures your host's `$TERM` terminfo entry and installs it inside the VM, so SSH from terminal emulators the guest doesn't already recognize (e.g. Ghostty) doesn't get garbled/duplicated input.
* **Host DNS Resolution**: The VM resolves DNS through the host's resolver, so it follows the host's VPN/corporate DNS instead of using its own.
* **Pre-installed AI CLIs**: Antigravity (`agy`), Claude Code (`claude`), Codex CLI (`codex`).

---

## Prerequisites
Before starting, ensure you have the following installed on your host machine (both installed via `apt`):

1. **VirtualBox** 7.0.26 — `virtualbox-7.0` isn't in Ubuntu's default repos, so add the [Oracle apt repo](https://www.virtualbox.org/wiki/Linux_Downloads) first:
   ```bash
   wget -O- https://www.virtualbox.org/download/oracle_vbox_2016.asc | sudo gpg --dearmor --yes --output /usr/share/keyrings/oracle-virtualbox-2016.gpg
   echo "deb [arch=amd64 signed-by=/usr/share/keyrings/oracle-virtualbox-2016.gpg] https://download.virtualbox.org/virtualbox/debian jammy contrib" | sudo tee /etc/apt/sources.list.d/virtualbox.list
   sudo apt update
   sudo apt install virtualbox-7.0
   ```

2. **Vagrant** 2.4.9 — same story with the [HashiCorp apt repo](https://developer.hashicorp.com/vagrant/install):
   ```bash
   wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor --yes --output /usr/share/keyrings/hashicorp-archive-keyring.gpg
   echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com jammy main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
   sudo apt update
   sudo apt install vagrant
   ```

   > These version numbers were the latest available in each repo as of August 2026 — a plain `apt install` currently lands on them exactly, but isn't pinned. A newer release in either repo will install ahead of the reference version instead of matching it; pin the package (e.g. `apt install vagrant=2.4.9-1`) if exact parity matters, keeping in mind these repos typically only keep the latest build available under that exact version string.

3. An SSH key generated at `~/.ssh/id_ed25519` (the script will automatically inject it for passwordless access).

4. *(Optional, for dotfiles via chezmoi)* A GitHub **fine-grained Personal Access Token** with **read-only** access, saved as plain text at `~/.ssh/github_pat_readonly`:
   ```bash
   echo -n "your-token-here" > ~/.ssh/github_pat_readonly
   chmod 600 ~/.ssh/github_pat_readonly
   ```
   Generate one at GitHub → Settings → Developer settings → Personal access tokens → **Fine-grained tokens**, scoped to the repositories you need (e.g. your dotfiles repo) with **Contents: Read-only** permission. Unlike a per-repo Deploy Key, a single token can cover multiple repositories. This only grants the VM read access for the one-way `DOTFILES_REPO` clone/update via chezmoi — see [Repos & Git Workflow](#repos--git-workflow) for how regular repos under `~/repos` are handled instead.

---

## Network Modes (NETWORK_MODE)

At the top of the `Vagrantfile`, you can modify the `NETWORK_MODE` variable to choose how the VM connects to the network:

| Mode | Value | Description | Recommended Use Case |
| :--- | :--- | :--- | :--- |
| **Private (Host-Only)** | `"private"` | Creates an isolated virtual network between your PC and the VM with a fixed IP (`192.168.56.10`). The VM accesses the internet via NAT. | **Recommended for daily use**. Works at home, work, and offline with no risk of IP conflicts. |
| **Public Static** | `"public_static"` | Connects the VM directly to the physical network (Bridge) with a configured static IP (`192.168.15.10`). | When other devices on the same local physical network need to access the VM. |
| **Public Dynamic** | `"public_dhcp"` | Connects the VM to the physical network (Bridge) and requests a dynamic IP from the local router via DHCP. | When you need a public connection but are in an environment where IPs are assigned dynamically. |

---

## Getting Started

1. Navigate to the repository directory:
   ```bash
   cd ~/repos/vagrant-sandbox
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
* `PROJECT`: A label mixed into the VM's name (`VM_NAME`) to tell multiple sandboxes apart (default: `"dev-project"`).
* `CPUs`: The number of CPU cores allocated to the VM (default: `2`).
* `MEMORY`: The amount of RAM in MB (default: `"8192"` ~8GB).
* `DISK_SIZE`: The size of the VM primary disk (default: `"50GB"`). Override it per machine in a gitignored `Vagrantfile.local` (e.g. `DISK_SIZE = "200GB"`). Growing an existing disk takes effect on `vagrant reload`; shrinking requires recreating the VM.
* `USERNAME` / `PASSWORD`: The default user created inside the VM (default: `user` / `pass`).
* `SSH_KEY_FILENAME`: The filename (inside `~/.ssh` on your **host**) of the public key injected for passwordless SSH into the VM (default: `"id_ed25519"`).
* `VM_PRIVATE_IP`: The static IP used in `"private"` mode (default: `192.168.56.10`).
* `VM_BRIDGED_IP`: The static IP used in `"public_static"` mode (default: `192.168.15.10`).
* `NETWORK_INTERFACE_PREFIX`: The prefix of your host's physical network interface for Bridge modes (default: `"en"`; e.g., `"wlp"` for Wi-Fi, `"en"` for wired ethernet).
* `DOTFILES_REPO`: The HTTPS URL of your dotfiles repo, applied via `chezmoi` on first boot (default: `"https://github.com/your-username/dotfiles.git"`). Requires the PAT from [Prerequisites](#prerequisites) to be set up, otherwise this step is skipped.
* `VM_GIT_PAT_FILENAME`: The filename (inside `~/.ssh` on your **host**) holding the read-only GitHub PAT (default: `"github_pat_readonly"`).

---

## Recommended Aliases (Zsh)

To manage the virtual machine from any directory in your terminal, you can add these aliases to your `~/.zshrc`:

```bash
alias vm-status='cd ~/repos/vagrant-sandbox && vagrant status && cd -'
alias vm-up='cd ~/repos/vagrant-sandbox && vagrant up && cd -'
alias vm-halt='cd ~/repos/vagrant-sandbox && vagrant halt && cd -'
alias vm-destroy='cd ~/repos/vagrant-sandbox && vagrant destroy && cd -'
```

*Remember to run `source ~/.zshrc` in your terminal after adding them to apply the changes.*

---

## Repos & Git Workflow

`~/repos` on your host is shared into the VM at `/home/USERNAME/repos` as a synced folder (bidirectional VirtualBox shared folder). This means:

* Any repo you keep under `~/repos` on the host is the **same files** you see inside the VM — there's no separate clone to keep in sync.
* All git network operations (`clone`, `fetch`, `pull`, `push`) for these repos should be run **from the host**, in that same `~/repos/...` directory, using your host's own SSH keys/agent.
* Inside the VM, just edit files under `~/repos/...` normally (CLI tools, editors, Claude Code, etc.) — there's no need for the VM to hold any git write credentials.

This is separate from `DOTFILES_REPO` (chezmoi), which the VM clones and updates itself over HTTPS using the optional read-only PAT from [Prerequisites](#prerequisites) — a one-way read-only pull that never needs to push.
