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

# Laptop Setup
Start with a server+GUI RHEL 9.x installation, with the "container management" software group selected.  During RHEL
installation, configure a regular user with `sudo` privileges on the host.
I have successfully deployed this setup on a NUC with 12 GB RAM and an Intel
N97 processor, which are pretty minimal specifications, but you can likely go
even lower on RAM if needed.

Once the system is back up

* sudo dnf install -y git 
* git clone https://github.com/tarexveff/jetson-rhem-workshop


Run the following script to register with Red Hat and update the system.

    sudo ./register-and-update.sh
    sudo reboot

After the system reboots, run the following script to install container
and ISO image tools.

    cd ~/rhel-bootc-image-gen
    sudo ./config-bootc.sh

You can use a publicly accessible registry like [Quay](https://quay.io)
but if you want to run this demo disconnected, you can also optionally
set up a local container registry using the following script.

    cd ~/rhel-bootc-image-gen
    sudo ./config-registry.sh

NB: If you set up an insecure registry on another RHEL instance,
please make sure to copy the `999-local-registry.conf` file to the
`~/rhel-bootc-image-gen` and `/etc/containers/registries.conf.d`
directories on this RHEL instance that will build the bootable container
images.

Login to Red Hat's container registry using your Red Hat customer portal
credentials and then pull the container image for the base bootable
container.

    podman login registry.redhat.io
    podman pull registry.redhat.io/rhel9/rhel-bootc:9.6

At this point, setup is complete.

## Build the base container image
Use the following command to build the `base` bootable container
image. This image contains the Firefox browser running in kiosk mode.

    cd ~/rhel-bootc-image-gen
    . demo.conf
    podman build -f BaseContainerfile -t $CONTAINER_REPO:base \
        --build-arg DEMO_USER=$DEMO_USER

Push the image to the registry.

    podman push $CONTAINER_REPO:base


## Deploy the image using an ISO file
Run the following command to generate an installable ISO file for your
bootable container. This command prepares a kickstart file to pull
the bootable container image from the registry and install that to the
filesystem on the target system. This kickstart file is then injected
into the standard RHEL boot ISO you downloaded earlier. It's important to
note that the content for the target system is actually in the bootable
container image in the registry. This ISO merely contains enough to start
the system and then use the kickstart file to pull the operating system
content from the container registry.

    sudo ./gen-iso.sh

The generated file is named `bootc-rhel.iso`. Use that file to boot
a physical edge device or virtual guest. Ensure that you use the UEFI
firmware option for a virtual guest or install to a physical edge device
that supports UEFI. Make sure this system is able to access your public
registry to pull down the bootable container image.

You may see a core dump when running in a guest VM if
memory is low. I've successfully tested running the bootable containers
on a laptop with 16GB of memory and a 512GB SDD.

Test the deployment by verifying that the kiosk user automatically logs
into a desktop where only the web browser is available with no other
desktop controls.
