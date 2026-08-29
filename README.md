# Vagrant Sandbox

This repository contains an automated Vagrant configuration to set up a development virtual machine. It is designed to be flexible and portable, working seamlessly across different network environments (such as work or home).

---

## Features
* **Dynamic Network Modes**: Supports Host-Only (Private) and Bridge (Public, static, or DHCP) connections.
* **Idempotent Provisioning**: Automated and fast package installation that skips already-installed tools to save time.
* **SSH Key Integration**: Automatically injects host SSH keys into the VM for passwordless access (without crashing if keys are missing on the host).
* **Host Terminfo Sync**: Captures your host's `$TERM` terminfo entry and installs it inside the VM, so SSH from terminal emulators the guest doesn't already recognize (e.g. Ghostty) doesn't get garbled/duplicated input.
* **Host DNS Resolution**: The VM resolves DNS through the host's resolver, so it follows the host's VPN/corporate DNS instead of using its own.
* **Dotfiles via chezmoi**: Automatically clones and applies your `DOTFILES_REPO` on every boot/provision (see [Prerequisites](#prerequisites)).
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

4. A [chezmoi](https://www.chezmoi.io/)-compatible dotfiles repo on GitHub, and a read-only PAT to clone it. `vagrant up`/`vagrant provision` **refuse to run without this** — chezmoi applying your dotfiles is a mandatory boot step, not optional. Set it up in four parts:

   1. **Have a dotfiles repo already on GitHub** (private is fine — it doesn't need to be public). This is what chezmoi clones and applies inside the VM on boot; if you don't have one yet, see chezmoi's [Quick start](https://www.chezmoi.io/quick-start/) for how to turn a dotfiles directory into one.

   2. **Point `DOTFILES_REPO` at it**, in your gitignored `Vagrantfile.local` (the tracked `Vagrantfile` only has a placeholder default):
      ```ruby
      DOTFILES_REPO = "https://github.com/your-username/your-dotfiles.git"
      ```
      (See [Customization](#customization-vagrantfile) for all overridable variables.)

   3. **Generate a fine-grained PAT** scoped to that repo, and save it as plain text at `~/.ssh/github_pat_readonly`:
      ```bash
      echo -n "your-token-here" > ~/.ssh/github_pat_readonly
      chmod 600 ~/.ssh/github_pat_readonly
      ```
      Generate one at GitHub → Settings → Developer settings → Personal access tokens → **Fine-grained tokens**, scoped to your dotfiles repo (or more) with **Contents: Read-only** permission. Unlike a per-repo Deploy Key, a single token can cover multiple repositories.

   4. **Verify the PAT before building the VM.** A missing file is caught at
      `vagrant up`, but an expired, revoked, or mis-saved token (e.g. truncated
      on paste, or with stray characters) only surfaces as a `chezmoi: git:
      exit status 128` / `Authentication failed` mid-provision. Check it
      against the GitHub API — expect `200`:
      ```bash
      tok=$(tr -d '\n' < ~/.ssh/github_pat_readonly)
      curl -s -o /dev/null -w "%{http_code}\n" \
        -H "Authorization: Bearer $tok" \
        https://api.github.com/repos/your-username/your-dotfiles
      ```
      `401` → the token itself is bad (regenerate it). `404` → the token is
      valid but has no access to that repo (fix its repository scope /
      **Contents: Read-only** permission).

   This PAT also gives the VM general read-only HTTPS access to GitHub as that user, available for any other `git clone`/`pull` you'd run inside the VM — though regular repos live on the host and are shared read/write into the VM via the `~/repos` synced folder instead, so they never need it — see [Repos & Git Workflow](#repos--git-workflow).

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
* `DOTFILES_REPO`: The HTTPS URL of your dotfiles repo, applied via `chezmoi` on first boot (default: `"https://github.com/your-username/dotfiles.git"`).
* `VM_GIT_PAT_FILENAME`: The filename (inside `~/.ssh` on your **host**) holding the required, read-only GitHub PAT (default: `"github_pat_readonly"`) — see [Prerequisites](#prerequisites). Without it, `vagrant up`/`vagrant provision` fail immediately with a clear error before touching the VM.

---

## Running `vagrant` from any directory

`vagrant` only finds this VM when run from a directory that contains the
`Vagrantfile` (or an ancestor of one). To keep using the real `vagrant`
subcommands (`status`, `up`, `halt`, `destroy`, …) from anywhere without
`cd`-ing here first, point Vagrant at this repo via `VAGRANT_CWD`.

Add this wrapper function to your `~/.zshrc` — it respects a local
`Vagrantfile` when you're inside another Vagrant project, and only falls back
to this sandbox otherwise:

```bash
# vagrant: use the current directory's Vagrantfile if present, else this sandbox
vagrant() {
  if [[ -f Vagrantfile || -f ../Vagrantfile || -n "$VAGRANT_CWD" ]]; then
    command vagrant "$@"
  else
    VAGRANT_CWD="$HOME/repos/vagrant-sandbox" command vagrant "$@"
  fi
}
```

If you only ever use this one Vagrant project, a plain
`export VAGRANT_CWD="$HOME/repos/vagrant-sandbox"` in `~/.zshrc` is enough —
but note it makes *every* `vagrant` call target this sandbox, even from inside
another project.

*Remember to run `source ~/.zshrc` (or open a new shell) afterwards to apply the changes.*

---

## Repos & Git Workflow

### Recommended: `~/repos` synced folder (git stays on the host)

`~/repos` on your host is shared into the VM at `/home/USERNAME/repos` as a synced folder (bidirectional VirtualBox shared folder). This means:

* Any repo you keep under `~/repos` on the host is the **same files** you see inside the VM — there's no separate clone to keep in sync.
* All git network operations (`clone`, `fetch`, `pull`, `push`) for these repos should be run **from the host**, in that same `~/repos/...` directory, using your host's own SSH keys/agent.
* Inside the VM, just edit files under `~/repos/...` normally (CLI tools, editors, Claude Code, etc.) — the VM's own GitHub PAT (see [Prerequisites](#prerequisites)) is never used for these, only for `DOTFILES_REPO` below.

This is the recommended way to work with repos in this sandbox, for two reasons: the repo always exists on the host regardless of whether the VM is even running (it's meant to be disposable — `vagrant destroy` and rebuild at will, without losing any repo state), and keeping git operations off the VM avoids exposing your remote repos to accidental or unwanted writes from something disposable. This is also why the VM's own GitHub PAT (below) is read-only by design rather than a write-capable credential.

### Alternative: cloning directly inside the VM

The VM's own GitHub PAT gives it general **read-only** HTTPS access — not just for `DOTFILES_REPO`. You can `git clone`/`git pull` any repo the token covers directly inside the VM if you want a copy that isn't tied to the host's `~/repos`, but keep in mind that copy only exists inside the VM and is lost on `vagrant destroy`. `git push` over that same HTTPS remote will always be rejected, since the PAT is read-only — that's intentional, to keep a disposable VM from writing to your remote repos by accident. To push from inside the VM anyway, pick one:

* **Authenticate on demand**: when `git push` prompts for credentials, supply a token/password with write access instead of the read-only PAT.
* **Forward your host's SSH agent**: connect with `ssh -A user@192.168.56.10` (or `ssh -A vm-ubuntu` with the alias from [Getting Started](#getting-started)) so the VM can sign pushes using your host's own SSH key for the duration of that session, then point the repo's push URL at SSH once: `git remote set-url --push origin git@github.com:your-username/your-repo.git`.

`DOTFILES_REPO` (chezmoi) is the one thing that automatically uses the VM's own PAT, to clone/update itself over HTTPS on boot — it never pushes on its own. Its push URL is set to SSH automatically too (derived from `DOTFILES_REPO`), so if you do want to push a dotfiles change made from inside the VM, connecting with `ssh -A` and running `git push` in `~/.local/share/chezmoi` is all that's needed — no manual `git remote set-url` step, unlike other repos cloned inside the VM.
