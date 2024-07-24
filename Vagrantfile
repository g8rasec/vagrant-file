BOX_IMAGE = "ubuntu/focal64"
HOSTNAME = BOX_IMAGE.split("/").first
VM_NAME = BOX_IMAGE.split("/")[1]
USERNAME = "gheezzer"
PASSWORD = "pass"
MEMORY = "12288"
CPUs = 7
NETWORK_INTERFACE_PREFIX = "wlp" # If you use a wired network, change the interface to "en0"
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
    #host.vm.network "public_network", type: "dhcp", bridge: detect_interface
    host.vm.network "public_network", ip: "172.20.10.10", bridge: detect_interface
  end

  config.vm.provider "virtualbox" do |vb|
    vb.memory = MEMORY
    vb.cpus = CPUs
  end

  config.vm.provision "shell", inline: <<-SHELL
    echo "Provisioning shell script..."
    
    sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config.d/60-cloudimg-settings.conf
    systemctl restart sshd    
    
    sudo apt-get update && apt-get install vim wget curl net-tools htop nmap ssh-pass y

    if ! id #{USERNAME} &>/dev/null; then
      echo "Creating user #{USERNAME}..."
      sudo useradd -d /home/#{USERNAME} -s /bin/bash #{USERNAME}
      sudo usermod -aG sudo #{USERNAME}
    else
      echo "User #{USERNAME} already exists."
    fi

    if [ ! -d "/home/#{USERNAME}" ]; then
      echo "Creating home directory for #{USERNAME}..."
      sudo mkdir /home/#{USERNAME}
      sudo chown #{USERNAME}:#{USERNAME} /home/#{USERNAME}
      echo "#{USERNAME}:#{PASSWORD}" | sudo chpasswd
    else
      echo "Home directory for #{USERNAME} already exists."
    fi
  SHELL
end

puts "Vagrant configuration completed."

