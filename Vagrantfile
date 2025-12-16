# NETWORK_INTERFACE_PREFIX specifies the prefix of the network interface name.
# For wireless connections, you might use a prefix like "wlp".
# For wired connections, change the prefix to something like "en0" or the appropriate interface name for your setup.
NETWORK_INTERFACE_PREFIX = "wlp"

USE_DHCP_NETWORK = false
BOX_IMAGE = "ubuntu/focal64"
PROJECT = "test"
CPUs = 8
MEMORY = "15890"
VM_IP_ADDRESS = "0.0.0.0"
USERNAME = "user"
PASSWORD = "pass"
SSH_KEY_FILENAME = "id_ed25519"
NODE_VERSION = "20"

HOSTNAME = "vm-" + BOX_IMAGE.split("/").first
VM_NAME = ("vm" + "-" + BOX_IMAGE.split("/")[1] + "-" + PROJECT).upcase

# Read the SSH public key from the host machine
SSH_PUBLIC_KEY_CONTENT = File.read(File.expand_path("~/.ssh/#{SSH_KEY_FILENAME}.pub")).strip

# Read the SSH private key from the host machine
SSH_PRIVATE_KEY_CONTENT = File.read(File.expand_path("~/.ssh/#{SSH_KEY_FILENAME}")).strip

GATEWAY_NETWORK = `ip route | awk '/default/ && $5 ~ /#{NETWORK_INTERFACE_PREFIX}/ {print $3}'`.strip

puts "Configurations set:"
puts "BOX_IMAGE: #{BOX_IMAGE}"
puts "HOSTNAME: #{HOSTNAME}"
puts "VM_NAME: #{VM_NAME}"
puts "USERNAME: #{USERNAME}"
puts "PASSWORD: #{PASSWORD}"
puts "MEMORY: #{MEMORY}"
puts "CPUs: #{CPUs}"
puts "NETWORK_INTERFACE_PREFIX: #{NETWORK_INTERFACE_PREFIX}"
puts "GATEWAY_NETWORK: #{GATEWAY_NETWORK}"

def detect_interface
  puts "Detecting network interface..."
  interface = `ip -o link show | awk -F': ' '{print $2}' | grep ^#{NETWORK_INTERFACE_PREFIX}`.strip
  if interface.empty?
    raise "No compatible network interface found."
  else
    puts "Detected network interface: #{interface}"
    interface
  end
end

Vagrant.configure("2") do |config|
  puts "Configuring Vagrant..."
  
  config.vm.define VM_NAME do |host|
    puts "Defining VM: #{VM_NAME}"
    host.vm.box = BOX_IMAGE
    host.vm.hostname = HOSTNAME
    
      # Configure the public network based on USE_DHCP_NETWORK variable
      if USE_DHCP_NETWORK
        host.vm.network "public_network", type: "dhcp", bridge: detect_interface
      else
        host.vm.network "public_network", ip: VM_IP_ADDRESS, bridge: detect_interface
      end
  end

  config.vm.provider "virtualbox" do |vb|
    vb.memory = MEMORY
    vb.cpus = CPUs
  end

  config.vm.provision "shell", inline: <<-SHELL
    echo "Provisioning shell script..."
    
    echo "Enabling password authentication for SSH..."
    sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config.d/60-cloudimg-settings.conf
    sudo systemctl restart sshd
   
    echo "Installing packages..."
    sudo apt-get update && sudo apt-get install -y vim zsh wget curl net-tools htop nmap apt-transport-https ca-certificates curl software-properties-common

    echo "Installing Node.js #{NODE_VERSION}..."
    curl -fsSL https://deb.nodesource.com/setup_#{NODE_VERSION}.x | sudo -E bash -
    sudo apt-get install -y nodejs
    echo "Updating npm to the latest version..."
    sudo npm install -g npm@latest

    echo "Installing Gemini CLI..."
    sudo npm install -g @google/gemini-cli

    if ! id #{USERNAME} &>/dev/null; then
      echo "Creating user #{USERNAME}..."
      sudo useradd -m -s /bin/bash -G sudo #{USERNAME}
      echo "#{USERNAME}:#{PASSWORD}" | sudo chpasswd
      echo "Copying default .bashrc and .profile to new user..."
      sudo -u #{USERNAME} cp /etc/skel/.bashrc /home/#{USERNAME}/.bashrc
      sudo -u #{USERNAME} cp /etc/skel/.profile /home/#{USERNAME}/.profile
    else
      echo "User #{USERNAME} already exists."
    fi
    # SSH Key Setup - Always execute this block
    echo "Setting up SSH key for #{USERNAME}..."
    mkdir -p /home/#{USERNAME}/.ssh
    echo "#{SSH_PUBLIC_KEY_CONTENT}" >> /home/#{USERNAME}/.ssh/authorized_keys
    chmod 700 /home/#{USERNAME}/.ssh
    chmod 600 /home/#{USERNAME}/.ssh/authorized_keys
    chown -R #{USERNAME}:#{USERNAME} /home/#{USERNAME}/.ssh
    echo "Setting up Bitbucket private key for #{USERNAME}..."
    echo "#{SSH_PRIVATE_KEY_CONTENT}" > /home/#{USERNAME}/.ssh/#{SSH_KEY_FILENAME}
    chown #{USERNAME}:#{USERNAME} /home/#{USERNAME}/.ssh/#{SSH_KEY_FILENAME}
    chmod 600 /home/#{USERNAME}/.ssh/#{SSH_KEY_FILENAME}
              SHELL
    end

puts "Vagrant configuration completed."
