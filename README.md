# Intention

This repository is intended to hold artifacts for creating a Flight Control (AKA Red Hat Edge Manager) 
homelab.  If you want to learn more about this project, check out its repo [here](https://github.com/flightctl/flightctl/blob/main/docs/user/introduction.md).
Also included in this repository are commands and ContainerFiles for creating NVIDIA Jetson Red Hat Enterprise Linux Image Mode (bootc) images 
that will be used to flash edge devices managed by Flight Control.

# Prerequisites
 
* A laptop running Red Hat Enterprise Linux 9.x
* DHCP server on your home network.  This can be installed on the laptop if needed per [these instructions](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/managing_networking_infrastructure_services/index#setting-up-the-dhcp-service-for-subnets-directly-connected-to-the-dhcp-server_providing-dhcp-services) or you can just let your home network router assign IP addresses.
* Ideally, a way to connect to your network via cables.  Wireless networking will involve additional steps outside the scope of this repo.
* One or more NVIDIA Orin-based devices to manage (Nano, NX, AGX)

# Initial Laptop Setup
Start with a server+GUI RHEL 9.x installation, with the "container management" software group selected.  During RHEL
installation, configure a regular user with `sudo` privileges on the host.
I have successfully deployed this setup on a NUC with 12 GB RAM and an Intel
N97 processor, which are pretty minimal specifications, but you can likely go
even lower on RAM if needed.

Once the system is back up

* sudo dnf install -y git 
* git clone https://github.com/tarexveff/jetson-rhem-workshop

# Flight Control Server Installation

Go into the ./jetson-rhem-workshop directory that you cloned and run the  
