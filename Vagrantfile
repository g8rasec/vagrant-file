# -*- mode: ruby -*-
# vi: set ft=ruby :

# Enable Vagrant's built-in disk management (no plugin required)
ENV["VAGRANT_EXPERIMENTAL"] = "disks"

# Per-machine overrides (gitignored), e.g.: DISK_SIZE = "200GB"
load "#{__dir__}/Vagrantfile.local" if File.exist?("#{__dir__}/Vagrantfile.local")

# ==============================================================================
# 1. VM CONFIGURATION CONSTANTS
# ==============================================================================
BOX_IMAGE        = "ubuntu/jammy64" unless defined?(BOX_IMAGE)
PROJECT          = "dev-project" unless defined?(PROJECT)
CPUs             = 8 unless defined?(CPUs)
MEMORY           = "15890" unless defined?(MEMORY)
USERNAME         = "user" unless defined?(USERNAME)
PASSWORD         = "pass" unless defined?(PASSWORD)
SSH_KEY_FILENAME     = "id_ed25519" unless defined?(SSH_KEY_FILENAME)             # host key authorized to SSH INTO the VM
VM_GIT_KEY_FILENAME  = "id_ed25519_readonly" unless defined?(VM_GIT_KEY_FILENAME) # read-only key placed INSIDE the VM for Git access
DISK_SIZE            = "100GB" unless defined?(DISK_SIZE)
DOTFILES_REPO        = "git@github.com:your-username/dotfiles.git" unless defined?(DOTFILES_REPO)

# ==============================================================================
# 2. NETWORK CONFIGURATION
# ==============================================================================
# Network mode: 
# - "private"       : Host-Only IP (always accessible via fixed IP from host)
# - "public_static" : Bridge with static IP
# - "public_dhcp"   : Bridge with DHCP (dynamic IP from router)
NETWORK_MODE             = "private" unless defined?(NETWORK_MODE)
VM_PRIVATE_IP            = "192.168.56.10" unless defined?(VM_PRIVATE_IP)
VM_BRIDGED_IP            = "172.23.11.200" unless defined?(VM_BRIDGED_IP)

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
VM_GIT_PRIV_KEY = read_ssh_key(VM_GIT_KEY_FILENAME, false)
VM_GIT_PUB_KEY  = read_ssh_key(VM_GIT_KEY_FILENAME, true)

GATEWAY_NETWORK = if NETWORK_MODE.start_with?("public")
                    `ip route | awk '/default/ && $5 ~ /#{NETWORK_INTERFACE_PREFIX}/ {print $3}'`.strip
                  else
                    "N/A (Private Network)"
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
# 5. VAGRANT CONFIGURATION
# ==============================================================================
Vagrant.configure("2") do |config|
  puts "Configuring Vagrant..."
  
  config.vm.define VM_NAME do |host|
    puts "Defining VM: #{VM_NAME}"
    host.vm.box      = BOX_IMAGE
    host.vm.hostname = HOSTNAME
    host.vm.disk :disk, size: DISK_SIZE, primary: true

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
    # itself). Everything else lives in the dotfiles repo's run_once scripts.
    echo "Installing bootstrap packages..."
    sudo apt-get update && sudo apt-get install -y zsh curl ca-certificates

    # 3. User Creation
    if ! id #{USERNAME} &>/dev/null; then
      echo "Creating user #{USERNAME}..."
      sudo useradd -m -s /usr/bin/zsh -G sudo #{USERNAME}
      echo "#{USERNAME}:#{PASSWORD}" | sudo chpasswd
      echo "Copying default profiles..."
      sudo -u #{USERNAME} cp /etc/skel/.bashrc /home/#{USERNAME}/.bashrc
      sudo -u #{USERNAME} cp /etc/skel/.profile /home/#{USERNAME}/.profile
    else
      echo "User #{USERNAME} already exists."
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
 
    # Setup private key for Git access
    if [ -n "#{VM_GIT_PRIV_KEY}" ]; then
      echo "Setting up private key for #{USERNAME}..."
      mkdir -p /home/#{USERNAME}/.ssh
      echo "#{VM_GIT_PRIV_KEY}" > /home/#{USERNAME}/.ssh/#{VM_GIT_KEY_FILENAME}
      chown #{USERNAME}:#{USERNAME} /home/#{USERNAME}/.ssh/#{VM_GIT_KEY_FILENAME}
      chmod 600 /home/#{USERNAME}/.ssh/#{VM_GIT_KEY_FILENAME}

      if [ -n "#{VM_GIT_PUB_KEY}" ]; then
        echo "Setting up matching public key for #{USERNAME}..."
        echo "#{VM_GIT_PUB_KEY}" > /home/#{USERNAME}/.ssh/#{VM_GIT_KEY_FILENAME}.pub
        chown #{USERNAME}:#{USERNAME} /home/#{USERNAME}/.ssh/#{VM_GIT_KEY_FILENAME}.pub
        chmod 644 /home/#{USERNAME}/.ssh/#{VM_GIT_KEY_FILENAME}.pub
      fi
    fi

    # 5. Apply Dotfiles (chezmoi)
    if [ -n "#{VM_GIT_PRIV_KEY}" ]; then
      echo "Applying dotfiles via chezmoi..."

      if ! grep -q "^github.com" /home/#{USERNAME}/.ssh/known_hosts 2>/dev/null; then
        ssh-keyscan -t ed25519 github.com >> /home/#{USERNAME}/.ssh/known_hosts 2>/dev/null
        chown #{USERNAME}:#{USERNAME} /home/#{USERNAME}/.ssh/known_hosts
        chmod 600 /home/#{USERNAME}/.ssh/known_hosts
      fi

      # A previously failed clone (e.g. bad SSH auth) can leave an incomplete, non-git source dir behind
      if [ -d "/home/#{USERNAME}/.local/share/chezmoi" ] && [ ! -d "/home/#{USERNAME}/.local/share/chezmoi/.git" ]; then
        echo "Removing incomplete chezmoi source directory from a previous failed attempt..."
        rm -rf "/home/#{USERNAME}/.local/share/chezmoi"
      fi

      if [ -d "/home/#{USERNAME}/.local/share/chezmoi/.git" ]; then
        echo "chezmoi source already present, pulling latest dotfiles..."
        sudo -u #{USERNAME} -i bash -c 'GIT_SSH_COMMAND="ssh -i ~/.ssh/#{VM_GIT_KEY_FILENAME} -o IdentitiesOnly=yes" chezmoi update --apply'
      else
        echo "Installing chezmoi and applying #{DOTFILES_REPO}..."
        sudo -u #{USERNAME} -i bash -c 'GIT_SSH_COMMAND="ssh -i ~/.ssh/#{VM_GIT_KEY_FILENAME} -o IdentitiesOnly=yes" sh -c "$(curl -fsLS https://get.chezmoi.io)" -- -b ~/.local/bin init --apply #{DOTFILES_REPO}'
      fi
    fi

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
