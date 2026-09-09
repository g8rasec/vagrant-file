# Vagrant Sandbox

Automated, portable Vagrant setup for a disposable development VM. Works
across network environments (home, work, offline).

## Features

* **Network modes** — Host-Only (private) or Bridge (static / DHCP).
* **Idempotent provisioning** — skips already-installed tools.
* **SSH key injection** — host public key added to the VM for passwordless login.
* **Host terminfo sync** — installs the host `$TERM` in the VM (fixes Ghostty etc.).
* **Host DNS** — VM resolves through the host resolver (follows host VPN DNS).
* **Dotfiles via chezmoi** — clones and applies `DOTFILES_REPO` on every boot.
* **AI CLIs preinstalled** — Antigravity (`agy`), Claude Code (`claude`), Codex (`codex`).

## Prerequisites

**1. VirtualBox 7.1** ([Oracle apt repo](https://www.virtualbox.org/wiki/Linux_Downloads)):

```bash
wget -O- https://www.virtualbox.org/download/oracle_vbox_2016.asc | sudo gpg --dearmor --yes --output /usr/share/keyrings/oracle-virtualbox-2016.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/oracle-virtualbox-2016.gpg] https://download.virtualbox.org/virtualbox/debian $(lsb_release -cs) contrib" | sudo tee /etc/apt/sources.list.d/virtualbox.list
sudo apt update && sudo apt install virtualbox-7.1
```

> **Secure Boot on?** `vboxdrv` won't load until its module-signing key is
> enrolled: `sudo /sbin/vboxconfig`, then
> `sudo mokutil --import /var/lib/shim-signed/mok/MOK.der` and reboot
> (confirm the key in the blue MOK Manager screen).

**2. Vagrant** ([HashiCorp apt repo](https://developer.hashicorp.com/vagrant/install)):

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor --yes --output /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vagrant
```

**3. SSH key pair** at `~/.ssh/id_ed25519{,.pub}` (`ssh-keygen -t ed25519`). The
public half is injected into the VM.

> Copied a key from a drive/backup/cloud sync? Fix its modes first — they
> often land world-readable and OpenSSH then silently falls back to a
> password prompt:
> `chmod 700 ~/.ssh && chmod 600 ~/.ssh/id_ed25519 && chmod 644 ~/.ssh/id_ed25519.pub`

**4. Dotfiles repo + read-only PAT** — mandatory; `vagrant up` refuses to run
without them.

1. A [chezmoi](https://www.chezmoi.io/)-compatible dotfiles repo on GitHub.
2. Point `DOTFILES_REPO` at it in a gitignored `Vagrantfile.local`:
   ```ruby
   DOTFILES_REPO = "https://github.com/your-username/your-dotfiles.git"
   ```
3. A fine-grained PAT (**Contents: Read-only**, covers 1+ repos) saved plain:
   ```bash
   echo -n "your-token" > ~/.ssh/github_pat_readonly && chmod 600 ~/.ssh/github_pat_readonly
   ```
4. Verify it (expect `200`; `401` = bad token, `404` = no repo access):
   ```bash
   curl -s -o /dev/null -w "%{http_code}\n" \
     -H "Authorization: Bearer $(tr -d '\n' < ~/.ssh/github_pat_readonly)" \
     https://api.github.com/repos/your-username/your-dotfiles
   ```

This PAT also gives the VM read-only HTTPS access to GitHub for any in-VM
`git clone`/`pull` — see [Repos & Git](#repos--git).

## Network modes (`NETWORK_MODE`)

| Value | Mode | IP | Use |
| :--- | :--- | :--- | :--- |
| `"private"` | Host-Only (default) | `192.168.56.10` | Daily use; no IP conflicts, works offline. |
| `"public_static"` | Bridge, static | `192.168.15.10` | Other LAN devices need to reach the VM. |
| `"public_dhcp"` | Bridge, DHCP | from router | Public connection where IPs are dynamic. |

## Getting started

```bash
cd ~/repos/vagrant-sandbox
vagrant up
vagrant ssh                    # or: ssh user@192.168.56.10
vagrant halt | reload | provision
```

> `ssh` asking for a password instead of using the key = bad key permissions
> (see Prerequisites 3). Confirm with `ssh -v user@192.168.56.10`.

`~/.ssh/config` entry for `ssh vm-ubuntu`:

```ssh-config
Host vm-ubuntu
    HostName 192.168.56.10
    User user
    StrictHostKeyChecking accept-new
```

## Customization (`Vagrantfile`)

Variables at the top of the `Vagrantfile` (override per machine in a
gitignored `Vagrantfile.local`):

| Variable | Default | Notes |
| :--- | :--- | :--- |
| `BOX_IMAGE` | `ubuntu/jammy64` | Base box. |
| `PROJECT` | `dev-project` | Label mixed into `VM_NAME`. |
| `CPUs` / `MEMORY` | `2` / `8192` | Cores / RAM (MB). |
| `DISK_SIZE` | `50GB` | Grow on `vagrant reload`; shrink needs a rebuild. |
| `USERNAME` / `PASSWORD` | `user` / `pass` | VM user. |
| `SSH_KEY_FILENAME` | `id_ed25519` | Host `~/.ssh` public key to inject. |
| `VM_PRIVATE_IP` | `192.168.56.10` | IP for `"private"` mode. |
| `VM_BRIDGED_IP` | `192.168.15.10` | IP for `"public_static"` mode. |
| `NETWORK_INTERFACE_PREFIX` | `en` | Host NIC prefix for Bridge (`wlp` = Wi-Fi). |
| `DOTFILES_REPO` | placeholder | chezmoi dotfiles HTTPS URL. |
| `VM_GIT_PAT_FILENAME` | `github_pat_readonly` | Host `~/.ssh` PAT file; required. |

## Running `vagrant` from any directory

`vagrant` only finds this VM from a directory with a `Vagrantfile`. Add this
wrapper to `~/.zshrc` or `~/.bashrc` (works in both) to point it here by
default while still respecting other Vagrant projects:

```bash
vagrant() {
  if [[ -f Vagrantfile || -f ../Vagrantfile || -n "$VAGRANT_CWD" ]]; then
    command vagrant "$@"
  else
    VAGRANT_CWD="$HOME/repos/vagrant-sandbox" command vagrant "$@"
  fi
}
```

(A plain `export VAGRANT_CWD="$HOME/repos/vagrant-sandbox"` also works but
targets this sandbox from *everywhere*.) Re-`source` the rc file afterwards.

## Commit conventions

[Conventional Commits](https://www.conventionalcommits.org/pt-br/v1.0.0-beta.4/),
enforced by `.githooks/commit-msg`:

* Subject `<type>(<scope>)?: <desc>` — ≤ 50 recommended, **72 hard limit**.
  Types: `build chore ci docs feat fix perf refactor revert style test`.
  `!` before `:` marks a breaking change.
* Body optional: blank line after the subject, lines wrapped at **72**.
* No `Co-Authored-By:` trailer.

Enable the hook once per clone: `git config core.hooksPath .githooks`

## Repos & Git

**Recommended — keep git on the host.** `~/repos` is shared into the VM at
`/home/USERNAME/repos`. Edit inside the VM; run all `clone`/`fetch`/`pull`/`push`
from the host with your own keys. The repo then survives `vagrant destroy`, and
a disposable VM never touches your remotes.

**Cloning inside the VM.** The VM's PAT is read-only HTTPS — `clone`/`pull` of
any covered repo works, but the copy dies with the VM and `push` is rejected.
To push from inside anyway: forward your agent (`ssh -A vm-ubuntu`) and set the
push URL to SSH (`git remote set-url --push origin git@github.com:you/repo.git`).

`DOTFILES_REPO` (chezmoi) uses the PAT to clone/update on boot; its push URL is
already SSH, so `ssh -A` + `git push` in `~/.local/share/chezmoi` just works.
