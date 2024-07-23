# Vagrant Ubuntu Development Environment Setup

This script automates the setup of a development environment using Vagrant and Ubuntu. It ensures that you have the necessary tools and configurations to get started quickly. Here's how to use it:

## Prerequisites

- **VirtualBox**: VirtualBox is used to run the Ubuntu virtual machine.
- **Vagrant**: Vagrant is utilized to manage the Ubuntu virtual machine.

## Instructions

1. Clone this repository to your local machine or download the script.

2. Open a terminal and navigate to the directory where the script is located.

3. Run the script using the following command:

   ```bash
   vagrant up

4. Run to check the assigned IP using the following command:

   ```bash
   vagrant ssh
   exit

5. To log in to the virtual machine from any directory, use the following command:

   ```bash
   ssh user@[your-assigned-ip]

## Installation and Configuration

- The script will create a Vagrant virtual machine based on the "ubuntu/focal64" box image.

- Once the VM is up and running, you can connect to it using SSH.

- The VM is connected to your network via DHCP, and the default gateway is automatically detected.

- The script also installs several useful packages, including Vim, Wget, Curl, Net-tools, Htop, Nmap, OpenSSH server, and OpenFortiVPN.

- The default SSH configuration allows password authentication for ease of use.

## Customization

You can customize the script by modifying the following variables:

- **`BOX_IMAGE`**: The base box image to use.
- **`USERNAME`**: The desired username for the VM.
- **`PASSWORD`**: The password for the user.
- **`MEMORY`**: The amount of memory to allocate to the VM.
- **`CPUs`**: The number of virtual CPUs for the VM.
- **`NETWORK_INTERFACE`**: Defines the type of network interface.

## Network Configuration

The script automatically detects the network interface based on your setup. If you are using a cable connection, it defaults to "en0." If you are using Wi-Fi, it defaults to "wlp."

## Acknowledgments

This script simplifies the setup of a development environment using Vagrant and Ubuntu, ensuring a smooth and consistent development experience.
