# Copyright 2018 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# kind node base image
#
# For systemd + docker configuration used below, see the following references:
# https://systemd.io/CONTAINER_INTERFACE/

# start from ubuntu, this image is reasonably small as a starting point
# for a kubernetes node image, it doesn't contain much we don't need
ARG BASE_IMAGE=ubuntu:21.10
FROM $BASE_IMAGE as build

# `docker buildx` automatically sets this arg value
ARG TARGETARCH

# Configure containerd and runc binaries from kind-ci/containerd-nightlies repository
# The repository contains latest stable releases and nightlies built for multiple architectures
ARG CONTAINERD_VERSION="1.5.5"
ARG CONTAINERD_BASE_URL="https://github.com/kind-ci/containerd-nightlies/releases/download/containerd-${CONTAINERD_VERSION}"
ARG CONTAINERD_URL="${CONTAINERD_BASE_URL}/containerd-${CONTAINERD_VERSION}.linux-${TARGETARCH}.tar.gz"
ARG CONTAINERD_AMD64_SHA256SUM="8130d1db8ba5a45678013465fc8dfebb252fc9bf068248c06f9dbb5d8554a5f7"
ARG CONTAINERD_ARM64_SHA256SUM="c452b52bd5ce3f788bbfbd39741c9066752803aaca5fa6da79b9658902a3a2ec"
ARG CONTAINERD_PPC64LE_SHA256SUM="28d34cec34a5df0f84d7e6b04d15b7d04be175ff1a370b8442217913cd4fb0d6"

ARG RUNC_URL="${CONTAINERD_BASE_URL}/runc.${TARGETARCH}"
ARG RUNC_AMD64_SHA256SUM="c68e8d63cedae73df92352dcdffb3279fdb7e1d0cc823b3972c8d94ad86b9222"
ARG RUNC_ARM64_SHA256SUM="72da4b87204a955b947af939cdbeda9caeae9c33d7d7aef86bd7fdcee112dd38"
ARG RUNC_PPC64LE_SHA256SUM="b95b4dc4f4624266e9982c5c411d8cffc011d99b0c9187f689025ad41ce24c06"

# Configure crictl binary from upstream
ARG CRICTL_VERSION="v1.21.0"
ARG CRICTL_URL="https://github.com/kubernetes-sigs/cri-tools/releases/download/${CRICTL_VERSION}/crictl-${CRICTL_VERSION}-linux-${TARGETARCH}.tar.gz"
ARG CRICTL_AMD64_SHA256SUM="85c78a35584971625bf1c3bcd46e5404a90396f979d7586f18b11119cb623e24"
ARG CRICTL_ARM64_SHA256SUM="454eecd29fe636282339af5b73c60234a7d10e4b11b9e18937e33056763d72cf"
ARG CRICTL_PPC64LE_SHA256SUM="0770100d30d430dbb67a58119ffed459856163ba01b6d71ac6fd4be7336253cf"

# Configure CNI binaries from upstream
ARG CNI_PLUGINS_VERSION="v0.9.1"
ARG CNI_PLUGINS_TARBALL="${CNI_PLUGINS_VERSION}/cni-plugins-linux-${TARGETARCH}-${CNI_PLUGINS_VERSION}.tgz"
ARG CNI_PLUGINS_URL="https://github.com/containernetworking/plugins/releases/download/${CNI_PLUGINS_TARBALL}"
ARG CNI_PLUGINS_AMD64_SHA256SUM="962100bbc4baeaaa5748cdbfce941f756b1531c2eadb290129401498bfac21e7"
ARG CNI_PLUGINS_ARM64_SHA256SUM="ef17764ffd6cdcb16d76401bac1db6acc050c9b088f1be5efa0e094ea3b01df0"
ARG CNI_PLUGINS_PPC64LE_SHA256SUM="5bd3c82ef248e5c6cc388f25545aa5a7d318778e5f9bc0a31475361bb27acefe"

# Configure containerd-fuse-overlayfs snapshotter binary from upstream
ARG CONTAINERD_FUSE_OVERLAYFS_VERSION="1.0.3"
ARG CONTAINERD_FUSE_OVERLAYFS_TARBALL="v${CONTAINERD_FUSE_OVERLAYFS_VERSION}/containerd-fuse-overlayfs-${CONTAINERD_FUSE_OVERLAYFS_VERSION}-linux-${TARGETARCH}.tar.gz"
ARG CONTAINERD_FUSE_OVERLAYFS_URL="https://github.com/containerd/fuse-overlayfs-snapshotter/releases/download/${CONTAINERD_FUSE_OVERLAYFS_TARBALL}"
ARG CONTAINERD_FUSE_OVERLAYFS_AMD64_SHA256SUM="26c7af08d292f21e7067c0424479945bb9ff6315b49851511b2917179c5ae59a"
ARG CONTAINERD_FUSE_OVERLAYFS_ARM64_SHA256SUM="68ef0896f3d5c0af73ad3d13b1b9a27f9b57cf22bdc30e36915d0f279b965bc3"
ARG CONTAINERD_FUSE_OVERLAYFS_PPC64LE_SHA256SUM="49679827fa2b46dd28899bdc53c2926e83f42d305ad7ee31aeaf50dbb774a840"

# copy in static files
# all scripts are 0755: http://www.filepermissions.com/file-permission/0755
COPY --chmod=0755 files/usr/local/bin/* /usr/local/bin/

# all configs are 0644: http://www.filepermissions.com/file-permission/0644
COPY --chmod=0644 files/etc/* /etc/
COPY --chmod=0644 files/etc/containerd/* /etc/containerd/
COPY --chmod=0644 files/etc/default/* /etc/default/
COPY --chmod=0644 files/etc/sysctl.d/* /etc/sysctl.d/
COPY --chmod=0644 files/etc/systemd/system/* /etc/systemd/system/
COPY --chmod=0644 files/etc/systemd/system/kubelet.service.d/* /etc/systemd/system/kubelet.service.d/

# Install dependencies, first from apt, then from release tarballs.
# NOTE: we use one RUN to minimize layers.
#
# First we must ensure that our util scripts are executable.
#
# The base image already has a basic userspace + apt but we need to install more packages.
# Packages installed are broken down into (each on a line):
# - packages needed to run services (systemd)
# - packages needed for kubernetes components
# - packages needed by the container runtime
# - misc packages kind uses itself
# - packages that provide semi-core kubernetes functionality
# After installing packages we cleanup by:
# - removing unwanted systemd services
# - disabling kmsg in journald (these log entries would be confusing)
#
# Then we install containerd from our nightly build infrastructure, as this
# build for multiple architectures and allows us to upgrade to patched releases
# more quickly.
#
# Next we download and extract crictl and CNI plugin binaries from upstream.
#
# Next we ensure the /etc/kubernetes/manifests directory exists. Normally
# a kubeadm debian / rpm package would ensure that this exists but we install
# freshly built binaries directly when we build the node image.
#
# Finally we adjust tempfiles cleanup to be 1 minute after "boot" instead of 15m
# This is plenty after we've done initial setup for a node, but before we are
# likely to try to export logs etc.

RUN echo "Installing Packages ..." \
    && DEBIAN_FRONTEND=noninteractive clean-install \
      systemd \
      conntrack iptables iproute2 ethtool socat util-linux mount ebtables kmod \
      libseccomp2 pigz \
      bash ca-certificates curl rsync \
      nfs-common fuse-overlayfs \
    && find /lib/systemd/system/sysinit.target.wants/ -name "systemd-tmpfiles-setup.service" -delete \
    && rm -f /lib/systemd/system/multi-user.target.wants/* \
    && rm -f /etc/systemd/system/*.wants/* \
    && rm -f /lib/systemd/system/local-fs.target.wants/* \
    && rm -f /lib/systemd/system/sockets.target.wants/*udev* \
    && rm -f /lib/systemd/system/sockets.target.wants/*initctl* \
    && rm -f /lib/systemd/system/basic.target.wants/* \
    && echo "ReadKMsg=no" >> /etc/systemd/journald.conf \
    && ln -s "$(which systemd)" /sbin/init

RUN echo "Enabling kubelet ... " \
    && systemctl enable kubelet.service

RUN echo "Installing containerd ..." \
    && curl -sSL --retry 5 --output /tmp/containerd.${TARGETARCH}.tgz "${CONTAINERD_URL}" \
    && echo "${CONTAINERD_AMD64_SHA256SUM}  /tmp/containerd.amd64.tgz" | tee /tmp/containerd.sha256 \
    && echo "${CONTAINERD_ARM64_SHA256SUM}  /tmp/containerd.arm64.tgz" | tee -a /tmp/containerd.sha256 \
    && echo "${CONTAINERD_PPC64LE_SHA256SUM}  /tmp/containerd.ppc64le.tgz" | tee -a /tmp/containerd.sha256 \
    && sha256sum --ignore-missing -c /tmp/containerd.sha256 \
    && rm -f /tmp/containerd.sha256 \
    && tar -C /usr/local -xzvf /tmp/containerd.${TARGETARCH}.tgz \
    && rm -rf /tmp/containerd.${TARGETARCH}.tgz \
    && rm -f /usr/local/bin/containerd-stress /usr/local/bin/containerd-shim-runc-v1 \
    && curl -sSL --retry 5 --output /tmp/runc.${TARGETARCH} "${RUNC_URL}" \
    && echo "${RUNC_AMD64_SHA256SUM}  /tmp/runc.amd64" | tee /tmp/runc.sha256 \
    && echo "${RUNC_ARM64_SHA256SUM}  /tmp/runc.arm64" | tee -a /tmp/runc.sha256 \
    && echo "${RUNC_PPC64LE_SHA256SUM}  /tmp/runc.ppc64le" | tee -a /tmp/runc.sha256 \
    && sha256sum --ignore-missing -c /tmp/runc.sha256 \
    && mv /tmp/runc.${TARGETARCH} /usr/local/sbin/runc \
    && chmod 755 /usr/local/sbin/runc \
    && containerd --version \
    && runc --version \
    && systemctl enable containerd

RUN echo "Installing crictl ..." \
    && curl -sSL --retry 5 --output /tmp/crictl.${TARGETARCH}.tgz "${CRICTL_URL}" \
    && echo "${CRICTL_AMD64_SHA256SUM}  /tmp/crictl.amd64.tgz" | tee /tmp/crictl.sha256 \
    && echo "${CRICTL_ARM64_SHA256SUM}  /tmp/crictl.arm64.tgz" | tee -a /tmp/crictl.sha256 \
    && echo "${CRICTL_PPC64LE_SHA256SUM}  /tmp/crictl.ppc64le.tgz" | tee -a /tmp/crictl.sha256 \
    && sha256sum --ignore-missing -c /tmp/crictl.sha256 \
    && rm -f /tmp/crictl.sha256 \
    && tar -C /usr/local/bin -xzvf /tmp/crictl.${TARGETARCH}.tgz \
    && rm -rf /tmp/crictl.${TARGETARCH}.tgz

RUN echo "Installing CNI plugin binaries ..." \
    && curl -sSL --retry 5 --output /tmp/cni.${TARGETARCH}.tgz "${CNI_PLUGINS_URL}" \
    && echo "${CNI_PLUGINS_AMD64_SHA256SUM}  /tmp/cni.amd64.tgz" | tee /tmp/cni.sha256 \
    && echo "${CNI_PLUGINS_ARM64_SHA256SUM}  /tmp/cni.arm64.tgz" | tee -a /tmp/cni.sha256 \
    && echo "${CNI_PLUGINS_PPC64LE_SHA256SUM}  /tmp/cni.ppc64le.tgz" | tee -a /tmp/cni.sha256 \
    && sha256sum --ignore-missing -c /tmp/cni.sha256 \
    && rm -f /tmp/cni.sha256 \
    && mkdir -p /opt/cni/bin \
    && tar -C /opt/cni/bin -xzvf /tmp/cni.${TARGETARCH}.tgz \
    && rm -rf /tmp/cni.${TARGETARCH}.tgz \
    && find /opt/cni/bin -type f -not \( \
         -iname host-local \
         -o -iname ptp \
         -o -iname portmap \
         -o -iname loopback \
      \) \
      -delete

RUN echo "Installing containerd-fuse-overlayfs ..." \
    && curl -sSL --retry 5 --output /tmp/containerd-fuse-overlayfs.${TARGETARCH}.tgz "${CONTAINERD_FUSE_OVERLAYFS_URL}" \
    && echo "${CONTAINERD_FUSE_OVERLAYFS_AMD64_SHA256SUM}  /tmp/containerd-fuse-overlayfs.amd64.tgz" | tee /tmp/containerd-fuse-overlayfs.sha256 \
    && echo "${CONTAINERD_FUSE_OVERLAYFS_ARM64_SHA256SUM}  /tmp/containerd-fuse-overlayfs.arm64.tgz" | tee -a /tmp/containerd-fuse-overlayfs.sha256 \
    && echo "${CONTAINERD_FUSE_OVERLAYFS_PPC64LE_SHA256SUM}  /tmp/containerd-fuse-overlayfs.ppc64le.tgz" | tee -a /tmp/containerd-fuse-overlayfs.sha256 \
    && sha256sum --ignore-missing -c /tmp/containerd-fuse-overlayfs.sha256 \
    && rm -f /tmp/containerd-fuse-overlayfs.sha256 \
    && tar -C /usr/local/bin -xzvf /tmp/containerd-fuse-overlayfs.${TARGETARCH}.tgz \
    && rm -rf /tmp/containerd-fuse-overlayfs.${TARGETARCH}.tgz

RUN echo "Ensuring /etc/kubernetes/manifests" \
    && mkdir -p /etc/kubernetes/manifests

RUN echo "Adjusting systemd-tmpfiles timer" \
    && sed -i /usr/lib/systemd/system/systemd-tmpfiles-clean.timer -e 's#OnBootSec=.*#OnBootSec=1min#'

# squash
FROM scratch
COPY --from=build / /

# tell systemd that it is in docker (it will check for the container env)
# https://systemd.io/CONTAINER_INTERFACE/
ENV container docker
# systemd exits on SIGRTMIN+3, not SIGTERM (which re-executes it)
# https://bugzilla.redhat.com/show_bug.cgi?id=1201657
STOPSIGNAL SIGRTMIN+3
# NOTE: this is *only* for documentation, the entrypoint is overridden later
ENTRYPOINT [ "/usr/local/bin/entrypoint", "/sbin/init" ]
