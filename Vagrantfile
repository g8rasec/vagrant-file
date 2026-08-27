# -*- mode: ruby -*-
# vi: set ft=ruby :

require "fileutils"

# Enable Vagrant's built-in disk management (no plugin required)
ENV["VAGRANT_EXPERIMENTAL"] = "disks"

# Per-machine overrides (gitignored), e.g.: DISK_SIZE = "200GB"
load "#{__dir__}/Vagrantfile.local" if File.exist?("#{__dir__}/Vagrantfile.local")

# ==============================================================================
# 1. VM CONFIGURATION CONSTANTS
# ==============================================================================
BOX_IMAGE        = "ubuntu/jammy64" unless defined?(BOX_IMAGE)
PROJECT          = "dev-project" unless defined?(PROJECT)
CPUs             = 2 unless defined?(CPUs)
MEMORY           = "8192" unless defined?(MEMORY)
USERNAME         = "user" unless defined?(USERNAME)
PASSWORD         = "pass" unless defined?(PASSWORD)
SSH_KEY_FILENAME     = "id_ed25519" unless defined?(SSH_KEY_FILENAME)             # host key authorized to SSH INTO the VM
VM_GIT_PAT_FILENAME  = "github_pat_readonly" unless defined?(VM_GIT_PAT_FILENAME) # host file with a read-only GitHub fine-grained PAT, for HTTPS Git access inside the VM
DISK_SIZE            = "50GB" unless defined?(DISK_SIZE)
DOTFILES_REPO        = "https://github.com/your-username/dotfiles.git" unless defined?(DOTFILES_REPO)

# ==============================================================================
# 2. NETWORK CONFIGURATION
# ==============================================================================
# Network mode:
# - "private"       : Host-Only IP (always accessible via fixed IP from host)
# - "public_static" : Bridge with static IP
# - "public_dhcp"   : Bridge with DHCP (dynamic IP from router)
NETWORK_MODE             = "private" unless defined?(NETWORK_MODE)
VM_PRIVATE_IP            = "192.168.56.10" unless defined?(VM_PRIVATE_IP)
VM_BRIDGED_IP            = "192.168.15.10" unless defined?(VM_BRIDGED_IP)

# Interface prefix to auto-detect the host network card for bridged networking:
# - "en" or "eth"   : Wired Ethernet interfaces (e.g., enp1s0, eth0)
# - "wlp" or "wlan" : Wireless Wi-Fi interfaces (e.g., wlp0s20f3, wlan0)
NETWORK_INTERFACE_PREFIX = "en" unless defined?(NETWORK_INTERFACE_PREFIX)

# ==============================================================================
# 3. HELPER FUNCTIONS
# ==============================================================================
# Read SSH key safely without crashing if the file is missing on the host
def read_ssh_key(filename, is_public = false)
  path = File.expand_path("~/.ssh/#{filename}" + (is_public ? ".pub" : ""))
  if File.exist?(path)
    File.read(path).strip
  else
    puts "WARNING: Host SSH key not found at #{path}."
    ""
  end
end

# Detect the network interface matching the prefix
def detect_interface
  puts "Detecting network interface..."
  interface = `ip -o link show | awk -F': ' '{print $2}' | grep ^#{NETWORK_INTERFACE_PREFIX}`.strip
  if interface.empty?
    raise "Error: No network interface matching '#{NETWORK_INTERFACE_PREFIX}' was found."
  else
    puts "Detected network interface: #{interface}"
    interface
  end
end

# ==============================================================================
# 4. LOAD STATE & LOG CONFIGURATION
# ==============================================================================
HOSTNAME  = "vm-" + BOX_IMAGE.split("/").first
VM_NAME   = ("vm-" + BOX_IMAGE.split("/")[1] + "-" + PROJECT).upcase

VM_SSH_PUB_KEY  = read_ssh_key(SSH_KEY_FILENAME, true)
VM_GIT_PAT      = read_ssh_key(VM_GIT_PAT_FILENAME, false)

if VM_GIT_PAT.empty?
  raise "Error: GitHub PAT not found at ~/.ssh/#{VM_GIT_PAT_FILENAME}. It's required to install and apply " \
        "dotfiles (DOTFILES_REPO) via chezmoi on VM boot. See README Prerequisites for how to generate one."
end

# Push-capable SSH equivalent of DOTFILES_REPO, so the chezmoi source repo can be
# pushed to from inside the VM (via `ssh -A`) without carrying the read-only PAT
# over into its push URL. Only rewrites a plain "https://github.com/..." URL.
DOTFILES_REPO_SSH = DOTFILES_REPO.sub(%r{\Ahttps://github\.com/}, "git@github.com:")

GATEWAY_NETWORK = if NETWORK_MODE.start_with?("public")
                    `ip route | awk '/default/ && $5 ~ /#{NETWORK_INTERFACE_PREFIX}/ {print $3}'`.strip
                  else
                    "N/A (Private Network)"
                  end

# ==============================================================================
# 5. HOST TERMINFO SYNC
# ==============================================================================
# Captures the host's current $TERM terminfo entry so it can be installed on
# the guest. Without this, SSH clients using a TERM the guest doesn't know
# about (e.g. xterm-ghostty on older distros) get garbled/duplicated input,
# because cursor-control sequences from the shell prompt can't be resolved.
HOST_TERM = ENV["TERM"]
HOST_TERMINFO_PATH = "#{__dir__}/.vagrant/host-terminfo.terminfo"

if HOST_TERM
  FileUtils.mkdir_p("#{__dir__}/.vagrant")
  if system("infocmp -x #{HOST_TERM} > #{HOST_TERMINFO_PATH} 2>/dev/null")
    puts "Captured host TERM='#{HOST_TERM}' terminfo for guest sync."
  else
    HOST_TERMINFO_PATH = nil
    puts "Could not capture host TERM='#{HOST_TERM}' terminfo (infocmp unavailable or unknown to host too)."
  end
else
  HOST_TERMINFO_PATH = nil
end

puts "=============================================================================="
puts "Configurations set:"
puts "  BOX_IMAGE:        #{BOX_IMAGE}"
puts "  HOSTNAME:         #{HOSTNAME}"
puts "  VM_NAME:          #{VM_NAME}"
puts "  USERNAME:         #{USERNAME}"
puts "  PASSWORD:         #{PASSWORD}"
puts "  MEMORY:           #{MEMORY} MB"
puts "  CPUs:             #{CPUs}"
puts "  DISK_SIZE:        #{DISK_SIZE}"
puts "  NETWORK_MODE:     #{NETWORK_MODE}"
if NETWORK_MODE == "private"
  puts "  VM_IP:            #{VM_PRIVATE_IP}"
elsif NETWORK_MODE == "public_static"
  puts "  VM_IP:            #{VM_BRIDGED_IP}"
else
  puts "  VM_IP:            DHCP (Dynamic)"
end
if NETWORK_MODE.start_with?("public")
  puts "  INTERFACE_PREFIX: #{NETWORK_INTERFACE_PREFIX}"
  puts "  GATEWAY_NETWORK:  #{GATEWAY_NETWORK}"
end
puts "=============================================================================="

# ==============================================================================
# 6. VAGRANT CONFIGURATION
# ==============================================================================
Vagrant.configure("2") do |config|
  puts "Configuring Vagrant..."

  config.vm.define VM_NAME do |host|
    puts "Defining VM: #{VM_NAME}"
    host.vm.box      = BOX_IMAGE
    host.vm.hostname = HOSTNAME
    host.vm.disk :disk, size: DISK_SIZE, primary: true

    # Share the host's ~/repos into the VM for repos cloned (and pushed) from the
    # host via SSH. Git network access for these repos never happens inside the
    # VM itself — only the resulting working tree is visible here.
    #
    # uid/gid=1000: placeholder only. Synced folders are mounted before the
    # shell provisioner creates USERNAME, so owner:/group: can't be used
    # (Vagrant would run `id -u` on a user that doesn't exist yet) and the
    # real UID/GID can't be known here on the host. It also isn't reliably
    # 1000 on the guest: official Canonical boxes already have "vagrant"
    # (1000) and cloud-init's default "ubuntu" user (1001) before useradd
    # ever runs, so USERNAME can land on 1002 or higher. The provisioner
    # below detects the real UID/GID after creating USERNAME and remounts
    # this share with the correct values.
    host.vm.synced_folder "~/repos", "/home/#{USERNAME}/repos", create: true, mount_options: ["uid=1000", "gid=1000"]

    # Apply network configuration based on mode
    case NETWORK_MODE
    when "private"
      puts "Configuring private network (Host-Only) with IP: #{VM_PRIVATE_IP}"
      host.vm.network "private_network", ip: VM_PRIVATE_IP
    when "public_dhcp"
      bridge_iface = detect_interface
      puts "Configuring public network (Bridge) via DHCP on #{bridge_iface}..."
      host.vm.network "public_network", type: "dhcp", bridge: bridge_iface
    when "public_static"
      bridge_iface = detect_interface
      puts "Configuring public network (Bridge) via Static IP #{VM_BRIDGED_IP} on #{bridge_iface}..."
      host.vm.network "public_network", ip: VM_BRIDGED_IP, bridge: bridge_iface
    else
      raise "Invalid NETWORK_MODE: '#{NETWORK_MODE}'. Choose 'private', 'public_static' or 'public_dhcp'."
    end
  end

  # VirtualBox Provider Settings
  config.vm.provider "virtualbox" do |vb|
    vb.memory = MEMORY
    vb.cpus   = CPUs
    # Resolve DNS via the host resolver so the VM follows the host VPN DNS
    vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
  end

  # Sync host terminfo (see section 5 above) so SSH from the host's terminal
  # doesn't get garbled input if the guest doesn't recognize the host's TERM.
  if HOST_TERMINFO_PATH
    config.vm.provision "file", source: HOST_TERMINFO_PATH, destination: "/tmp/host.terminfo"
  end

  # Shell Provisioning
  config.vm.provision "shell", inline: <<-SHELL
    echo "Starting shell provisioning..."

    # 1. SSH Password Authentication
    echo "Enabling password authentication for SSH..."
    if [ -f /etc/ssh/sshd_config.d/60-cloudimg-settings.conf ]; then
      sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config.d/60-cloudimg-settings.conf
    else
      sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config
      sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
    fi
    sudo systemctl restart sshd

    # 2. Bootstrap Package Installation
    # Only what's strictly required before the user exists and dotfiles can
    # take over: zsh (login shell), curl+ca-certificates (fetch chezmoi
    # itself), ncurses-term (terminfo entries for common terminal emulators).
    # Everything else lives in the dotfiles repo's run_once scripts.
    echo "Installing bootstrap packages..."
    sudo apt-get update && sudo apt-get install -y zsh curl ca-certificates ncurses-term

    # 3. User Creation
    if ! id #{USERNAME} &>/dev/null; then
      echo "Creating user #{USERNAME}..."
      sudo useradd -m -s /usr/bin/zsh -G sudo #{USERNAME}
      echo "#{USERNAME}:#{PASSWORD}" | sudo chpasswd

      # useradd -m finds /home/#{USERNAME} already existing (created by root
      # by Vagrant as the mountpoint for the repos synced folder, before this
      # provisioner runs) and refuses to change its ownership or copy skel
      # files into it. Fix the home dir's ownership here — NOT recursive,
      # since /home/#{USERNAME}/repos is a separate vboxsf mount, fixed up
      # on its own in step 3.1 below.
      sudo chown #{USERNAME}:#{USERNAME} /home/#{USERNAME}

      echo "Copying default profiles..."
      sudo -u #{USERNAME} cp /etc/skel/.bashrc /home/#{USERNAME}/.bashrc
      sudo -u #{USERNAME} cp /etc/skel/.profile /home/#{USERNAME}/.profile
    else
      echo "User #{USERNAME} already exists."
    fi

    # 3.1. Fix Repos Synced-Folder Ownership
    # The synced_folder mount_options in the Vagrantfile use a static
    # placeholder uid/gid=1000, since Vagrant mounts it before #{USERNAME}
    # exists and the real UID/GID depends on which regular users the box
    # already ships with (e.g. "vagrant", and cloud-init's default "ubuntu"
    # user on official Canonical images) — so it isn't always 1000. Now that
    # #{USERNAME} exists, detect its real UID/GID and remount with the
    # correct values if they don't already match.
    REPOS_MOUNT="/home/#{USERNAME}/repos"
    REAL_UID=$(id -u #{USERNAME})
    REAL_GID=$(id -g #{USERNAME})
    # vboxsf never surfaces uid=/gid= in /proc/mounts (findmnt -o OPTIONS comes back
    # empty for them), so read the mapped owner off the mount point itself instead.
    CURRENT_UID=$(stat -c '%u' "$REPOS_MOUNT" 2>/dev/null)
    if [ -n "$CURRENT_UID" ] && [ "$CURRENT_UID" != "$REAL_UID" ]; then
      SHARE_NAME=$(awk -v mnt="$REPOS_MOUNT" '$2 == mnt {print $1}' /etc/fstab)
      echo "Remounting $REPOS_MOUNT with uid=$REAL_UID,gid=$REAL_GID (was uid=$CURRENT_UID) for #{USERNAME}..."
      # umount can fail with "target is busy" (e.g. a shell with its cwd inside
      # $REPOS_MOUNT during provisioning). Only rewrite fstab if the remount
      # actually succeeded — otherwise fstab and the live mount end up out of
      # sync, silently leaving #{USERNAME} locked out of $REPOS_MOUNT.
      if sudo umount "$REPOS_MOUNT" && sudo mount -t vboxsf -o uid=$REAL_UID,gid=$REAL_GID "$SHARE_NAME" "$REPOS_MOUNT"; then
        sudo sed -i -E "\\|$REPOS_MOUNT|s/uid=[0-9]+,gid=[0-9]+/uid=$REAL_UID,gid=$REAL_GID/" /etc/fstab
      else
        echo "WARNING: failed to remount $REPOS_MOUNT (target busy?) — left fstab untouched. Re-run 'vagrant provision' once nothing has its cwd inside $REPOS_MOUNT." >&2
      fi
    fi

    # 3.5. Host Terminfo Installation
    # Installs the host's TERM terminfo (copied in by the "file" provisioner
    # above, see section 5) into #{USERNAME}'s own ~/.terminfo, so SSH
    # sessions from the host's terminal emulator (e.g. Ghostty) don't get
    # garbled/duplicated input from unresolved cursor-control sequences.
    if [ -f /tmp/host.terminfo ]; then
      echo "Installing host terminfo for #{USERNAME}..."
      sudo -u #{USERNAME} -i tic -x /tmp/host.terminfo
      rm -f /tmp/host.terminfo
    fi

    # 4. SSH Credentials Configuration
    # Setup public key
    if [ -n "#{VM_SSH_PUB_KEY}" ]; then
      echo "Setting up SSH key for #{USERNAME}..."
      mkdir -p /home/#{USERNAME}/.ssh
      if ! grep -qF "#{VM_SSH_PUB_KEY}" /home/#{USERNAME}/.ssh/authorized_keys 2>/dev/null; then
        echo "#{VM_SSH_PUB_KEY}" >> /home/#{USERNAME}/.ssh/authorized_keys
      fi
      chmod 700 /home/#{USERNAME}/.ssh
      chmod 600 /home/#{USERNAME}/.ssh/authorized_keys
      chown -R #{USERNAME}:#{USERNAME} /home/#{USERNAME}/.ssh
    fi

    # Setup read-only GitHub PAT for #{USERNAME} (HTTPS, covers any repo the token grants read access to)
    echo "Setting up read-only GitHub PAT for #{USERNAME}..."
    sudo -u #{USERNAME} -i git config --global credential.helper store
    echo "https://x-access-token:#{VM_GIT_PAT}@github.com" > /home/#{USERNAME}/.git-credentials
    chown #{USERNAME}:#{USERNAME} /home/#{USERNAME}/.git-credentials
    chmod 600 /home/#{USERNAME}/.git-credentials

    # 5. Apply Dotfiles (chezmoi)
    echo "Applying dotfiles via chezmoi..."

    # Grant #{USERNAME} passwordless sudo only for this step: the dotfiles repo's
    # run_once scripts call plain "sudo apt-get install" internally, and this whole
    # provisioner runs non-interactively with no TTY, so sudo can't prompt for a
    # password here. Revoked immediately after chezmoi finishes, below — every other
    # step (AI CLI installs, interactive `vagrant ssh` sessions) never needs this.
    echo "#{USERNAME} ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/90-#{USERNAME}-nopasswd > /dev/null
    sudo chmod 440 /etc/sudoers.d/90-#{USERNAME}-nopasswd

    # A previously failed clone (e.g. bad auth) can leave an incomplete, non-git source dir behind
    if [ -d "/home/#{USERNAME}/.local/share/chezmoi" ] && [ ! -d "/home/#{USERNAME}/.local/share/chezmoi/.git" ]; then
      echo "Removing incomplete chezmoi source directory from a previous failed attempt..."
      rm -rf "/home/#{USERNAME}/.local/share/chezmoi"
    fi

    if [ -d "/home/#{USERNAME}/.local/share/chezmoi/.git" ]; then
      echo "chezmoi source already present, pulling latest dotfiles..."
      # --force: this runs with no TTY, so if a managed file (e.g. .zshrc) was
      # touched outside chezmoi since the last apply — the AI CLI installers below
      # append PATH lines to it — chezmoi would otherwise try to prompt on /dev/tty,
      # fail to open it, and abort before running the run_once_after scripts. Since
      # this VM is disposable/reproducible, the dotfiles state should always win.
      sudo -u #{USERNAME} -i bash -c '~/.local/bin/chezmoi update --apply --force'
    else
      echo "Installing chezmoi and applying #{DOTFILES_REPO}..."
      sudo -u #{USERNAME} -i bash -c 'sh -c "$(curl -fsLS https://get.chezmoi.io)" -- -b ~/.local/bin init --apply #{DOTFILES_REPO}'
    fi

    # Point the chezmoi source repo's push URL at SSH instead of the read-only PAT's
    # HTTPS remote, so a `git push` from inside the VM (e.g. over `ssh -A`) has
    # somewhere to succeed, without ever making the read-only PAT itself push-capable.
    sudo -u #{USERNAME} -i git -C /home/#{USERNAME}/.local/share/chezmoi remote set-url --push origin #{DOTFILES_REPO_SSH}

    # Revoke passwordless sudo now that chezmoi (and any apt installs it triggered) is done.
    echo "Revoking passwordless sudo for #{USERNAME}..."
    sudo rm -f /etc/sudoers.d/90-#{USERNAME}-nopasswd

    # 6. AI CLI Tools (VM-only, not part of dotfiles)
    if ! sudo -u #{USERNAME} -i command -v agy &>/dev/null; then
      echo "Installing Antigravity CLI (agy) for #{USERNAME}..."
      sudo -u #{USERNAME} -i bash -c "curl -fsSL https://antigravity.google/cli/install.sh | bash"
    else
      echo "Antigravity CLI (agy) is already installed."
    fi

    if ! sudo -u #{USERNAME} -i command -v claude &>/dev/null; then
      echo "Installing Claude Code for #{USERNAME}..."
      sudo -u #{USERNAME} -i bash -c "curl -fsSL https://claude.ai/install.sh | bash"
    else
      echo "Claude Code is already installed."
    fi

    if ! sudo -u #{USERNAME} -i command -v codex &>/dev/null; then
      echo "Installing Codex CLI for #{USERNAME}..."
      sudo -u #{USERNAME} -i bash -c "curl -fsSL https://chatgpt.com/codex/install.sh | sh"
    else
      echo "Codex CLI is already installed."
    fi

  SHELL
end

puts "Vagrant configuration completed."
