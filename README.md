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

* Go into the ./jetson-rhem-workshop directory that you cloned
* Run sudo ./1_configure-firewall-and-registry.sh
* Run sudo ./2_rhem-server-build-commands.sh

Once these scripts are complete, you should be able to open the Flight Control web UI at https://<laptop hostname>.  The login credentials are admin/admin unless you changed them in the 2_rhem-server-build-commands.sh script.

# Post Installation Steps

With Flight Control now installed, you can generate the config.yaml file that you'll need to place on your managed devices to allow them to enroll to the server:

* flightctl login https://<laptop hostname>:3443 --insecure-skip-tls-verify --web
* flightctl certificate request --signer=enrollment --expiration=365d --output=embedded > config.yaml

Save this config.yaml file for use later!

# Managed Device Image Creation

Currently, building images on an x86 computer for NVIDIA ARM-based devices is very challenging.  I recommend creating an NVIDIA "build server" based on the bootc ContainerFile in this repo (ContainerFile-Jetson-rhem-1.0.2-base).  Bootstrapping this setup can be a challenge if you don't have an ARM server available, though you may be able to perform an initial build using an AWS Graviton instance if you don't have an NVIDIA system available.
