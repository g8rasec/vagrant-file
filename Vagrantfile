# -*- mode: ruby -*-
# vi: set ft=ruby :

# Enable Vagrant's built-in disk management (no plugin required)
ENV["VAGRANT_EXPERIMENTAL"] = "disks"

# ==============================================================================
# 1. VM CONFIGURATION CONSTANTS
# ==============================================================================
BOX_IMAGE        = "ubuntu/jammy64"
PROJECT          = "dev-project"
CPUs             = 8
MEMORY           = "15890"
USERNAME         = "user"
PASSWORD         = "pass"
SSH_KEY_FILENAME = "id_ed25519"
NODE_VERSION     = "24"
DISK_SIZE        = "200GB"

# ==============================================================================
# 2. NETWORK CONFIGURATION
# ==============================================================================
# Network mode: 
# - "private"       : Host-Only IP (always accessible via fixed IP from host)
# - "public_static" : Bridge with static IP
# - "public_dhcp"   : Bridge with DHCP (dynamic IP from router)
NETWORK_MODE             = "private"
VM_PRIVATE_IP            = "192.168.56.10"
VM_BRIDGED_IP            = "172.23.11.200"

# Interface prefix to auto-detect the host network card for bridged networking:
# - "en" or "eth"   : Wired Ethernet interfaces (e.g., enp1s0, eth0)
# - "wlp" or "wlan" : Wireless Wi-Fi interfaces (e.g., wlp0s20f3, wlan0)
NETWORK_INTERFACE_PREFIX = "en"

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

SSH_PUBLIC_KEY_CONTENT  = read_ssh_key(SSH_KEY_FILENAME, true)
SSH_PRIVATE_KEY_CONTENT = read_ssh_key(SSH_KEY_FILENAME, false)

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
   
    # 2. Package Installation
    echo "Installing base packages..."
    sudo apt-get update && sudo apt-get install -y \
      vim zsh wget curl net-tools htop nmap apt-transport-https ca-certificates software-properties-common

    # 3. Node.js Installation (Only if not already installed)
    if ! command -v node &>/dev/null; then
      echo "Installing Node.js #{NODE_VERSION}..."
      curl -fsSL https://deb.nodesource.com/setup_#{NODE_VERSION}.x | sudo -E bash -
      sudo apt-get install -y nodejs
      echo "Updating npm to the latest version..."
      sudo npm install -g npm@latest
    else
      echo "Node.js $(node -v) is already installed."
    fi

    # 4. User Creation
    if ! id #{USERNAME} &>/dev/null; then
      echo "Creating user #{USERNAME}..."
      sudo useradd -m -s /bin/bash -G sudo #{USERNAME}
      echo "#{USERNAME}:#{PASSWORD}" | sudo chpasswd
      echo "Copying default profiles..."
      sudo -u #{USERNAME} cp /etc/skel/.bashrc /home/#{USERNAME}/.bashrc
      sudo -u #{USERNAME} cp /etc/skel/.profile /home/#{USERNAME}/.profile
    else
      echo "User #{USERNAME} already exists."
    fi

    # 5. Antigravity CLI Installation (Ensure it is present)
    if ! sudo -u #{USERNAME} -i command -v agy &>/dev/null; then
      echo "Installing Antigravity CLI (agy) for #{USERNAME}..."
      sudo -u #{USERNAME} -i bash -c "curl -fsSL https://antigravity.google/cli/install.sh | bash"
    else
      echo "Antigravity CLI (agy) is already installed."
    fi

    # 6. SSH Credentials Configuration
    # Setup public key
    if [ -n "#{SSH_PUBLIC_KEY_CONTENT}" ]; then
      echo "Setting up SSH key for #{USERNAME}..."
      mkdir -p /home/#{USERNAME}/.ssh
      if ! grep -qF "#{SSH_PUBLIC_KEY_CONTENT}" /home/#{USERNAME}/.ssh/authorized_keys 2>/dev/null; then
        echo "#{SSH_PUBLIC_KEY_CONTENT}" >> /home/#{USERNAME}/.ssh/authorized_keys
      fi
      chmod 700 /home/#{USERNAME}/.ssh
      chmod 600 /home/#{USERNAME}/.ssh/authorized_keys
      chown -R #{USERNAME}:#{USERNAME} /home/#{USERNAME}/.ssh
    fi

    # Setup private key for Git / Bitbucket access
    if [ -n "#{SSH_PRIVATE_KEY_CONTENT}" ]; then
      echo "Setting up private key for #{USERNAME}..."
      mkdir -p /home/#{USERNAME}/.ssh
      echo "#{SSH_PRIVATE_KEY_CONTENT}" > /home/#{USERNAME}/.ssh/#{SSH_KEY_FILENAME}
      chown #{USERNAME}:#{USERNAME} /home/#{USERNAME}/.ssh/#{SSH_KEY_FILENAME}
      chmod 600 /home/#{USERNAME}/.ssh/#{SSH_KEY_FILENAME}
    fi
  SHELL
end

puts "Vagrant configuration completed."
