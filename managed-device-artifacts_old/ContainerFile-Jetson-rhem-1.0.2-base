# Start with a standard RHEL Image Mode image for 9.4
FROM registry.redhat.io/rhel9/rhel-bootc:9.4

# Setup folders to store microshift images for offline image creation
ENV IMAGE_STORAGE_DIR=/usr/lib/containers/storage
ENV IMAGE_LIST_FILE=${IMAGE_STORAGE_DIR}/image-list.txt

# Setup NVIDIA repositories
RUN curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo | tee /etc/yum.repos.d/nvidia-container-toolkit.repo
RUN curl -s -L https://repo.download.nvidia.com/jetson/rhel-9.4/jp6.1/nvidia-l4t.repo | tee /etc/yum.repos.d/nvidia-l4t.repo

# Setup NVIDIA Jetson Orin support packages
RUN dnf -y install nvidia-jetpack-kmod nvidia-jetpack-all nvidia-container-toolkit &&\
    dnf clean all

# Prevent the kernel from being updated
RUN echo "exclude=kernel* kmod-kvdo" >> /etc/yum.conf

# Set microshift version to install
ARG USHIFT_VER=4.18

# Install microshift
RUN dnf config-manager \
        --set-enabled rhocp-${USHIFT_VER}-for-rhel-9-$(uname -m)-rpms \
        --set-enabled fast-datapath-for-rhel-9-$(uname -m)-rpms
RUN dnf install -y firewalld microshift microshift-release-info && \
    systemctl enable microshift && \
    dnf clean all

# Create a default 'redhat' user with the specified password.
# Add it to the 'wheel' group to allow for running sudo commands.
ARG USER_PASSWD
RUN if [ -z "${USER_PASSWD}" ] ; then \
        echo USER_PASSWD is a mandatory build argument && exit 1 ; \
    fi
RUN useradd -m -d /var/home/redhat -G wheel redhat && \
    echo "redhat:${USER_PASSWD}" | chpasswd

# Mandatory firewall configuration for microshift
RUN firewall-offline-cmd --zone=public --add-port=22/tcp && \
    firewall-offline-cmd --zone=trusted --add-source=10.42.0.0/16 && \
    firewall-offline-cmd --zone=trusted --add-source=169.254.169.1

# Create a systemd unit to recursively make the root filesystem subtree
# shared as required by OVN images
RUN cat > /etc/systemd/system/microshift-make-rshared.service <<'EOF'
[Unit]
Description=Make root filesystem shared
Before=microshift.service
ConditionVirtualization=container
[Service]
Type=oneshot
ExecStart=/usr/bin/mount --make-rshared /
[Install]
WantedBy=multi-user.target
EOF
RUN systemctl enable microshift-make-rshared.service
# Pull the container images into /usr/lib/containers/storage:
# - Each image goes into a separate sub-directory
# - Sub-directories are named after the image reference string SHA
# - An image list file maps image references to their name SHA
# hadolint ignore=DL4006
RUN --mount=type=secret,id=pullsecret,dst=/run/secrets/pull-secret.json \
    images="$(jq -r ".images[]" /usr/share/microshift/release/release-"$(uname -m)".json)" ; \
    mkdir -p "${IMAGE_STORAGE_DIR}" ; \
    for img in ${images} ; do \
        sha="$(echo "${img}" | sha256sum | awk '{print $1}')" ; \
        skopeo copy --all --preserve-digests \
            --authfile /run/secrets/pull-secret.json \
            "docker://${img}" "dir:$IMAGE_STORAGE_DIR/${sha}" ; \
        echo "${img},${sha}" >> "${IMAGE_LIST_FILE}" ; \
    done

# Install a systemd drop-in unit to address the problem with image upgrades
# overwriting the container images in additional store. The workaround is to
# copy the images from the pre-loaded to the main container storage.
# In this case, it is not necessary to update /etc/containers/storage.conf with
# the additional store path.
# See https://issues.redhat.com/browse/RHEL-75827
RUN cat > /usr/bin/microshift-copy-images <<EOF
#!/bin/bash
set -eux -o pipefail
while IFS="," read -r img sha ; do
    skopeo copy --preserve-digests \
        "dir:${IMAGE_STORAGE_DIR}/\${sha}" \
        "containers-storage:\${img}"
done < "${IMAGE_LIST_FILE}"
EOF

RUN chmod 755 /usr/bin/microshift-copy-images && \
    mkdir -p /usr/lib/systemd/system/microshift.service.d

RUN cat > /usr/lib/systemd/system/microshift.service.d/microshift-copy-images.conf <<EOF
[Service]
ExecStartPre=/usr/bin/microshift-copy-images
EOF

# Create a service to regenerate the cdi mappings for the NVIDIA Jetson Orin iGPU
RUN cat > /usr/lib/systemd/system/nvidia-cdi.service <<EOF
[Unit]
Description=Generate the nvidia cdi yaml file
Requires=multi-user.target
After=multi-user.target
[Service]
Type=oneshot
ExecStart=nvidia-ctk cdi generate --output=/etc/cdi/nvidia
RemainAfterExit=yes
[Install]
WantedBy=multi-user.target
EOF
RUN systemctl enable nvidia-cdi.service

# Set the redhat user to be in the video group to allow rootless use of the iGPU
RUN usermod -a -G video redhat

RUN dnf config-manager --add-repo https://rpm.flightctl.io/flightctl-epel.repo && \
    dnf -y install python3-pip && pip install podman-compose && \
    dnf -y install flightctl-agent-1.0.2-1.el9.aarch64 && \
    dnf -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm lorax && \
    dnf -y clean all && \
    systemctl enable flightctl-agent.service
