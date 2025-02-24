# How to build and test Cloud Hypervisor on AArch64

This document introduces how to build and test Cloud Hypervisor on AArch64 servers. Currently Cloud Hypervisor cannot be tested on Raspberry PI. Because on AArch64, Cloud Hypervisor requires GICv3-ITS device for PCIe MSI interrupt handling. But GICv3-ITS has not been equipped on any Raspberry PI product so far.

Now Cloud Hypervisor supports 2 ways of booting on AArch64: UEFI booting and direct-kernel booting. The document covers both of the ways.

All the steps are based on Ubuntu. We use the Ubuntu cloud image for guest VM disk.

## Getting started

We create a folder to build and run Cloud Hypervisor at `$HOME/cloud-hypervisor`

```shell
$ export CLOUDH=$HOME/cloud-hypervisor
$ mkdir $CLOUDH
```

## Prerequisites

You need to install some prerequisite packages to build and test Cloud Hypervisor.

### Tools

```bash
# Install rust tool chain
$ curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
# Install the tools used for building guest kernel, EDK2 and converting guest disk
$ sudo apt-get update
$ sudo apt-get install git build-essential m4 bison flex uuid-dev qemu-utils
```

### Building Cloud Hypervisor

```bash
$ pushd $CLOUDH
$ git clone https://github.com/cloud-hypervisor/cloud-hypervisor.git
$ cd cloud-hypervisor
$ cargo build
$ popd
```

### Disk image

Download the Ubuntu cloud image and convert the image type.

```bash
$ pushd $CLOUDH
$ wget https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-arm64.img
$ qemu-img convert -p -f qcow2 -O raw focal-server-cloudimg-arm64.img focal-server-cloudimg-arm64.raw
$ popd
```

## UEFI booting

This part introduces how to build EDK2 firmware and boot Cloud Hypervisor with it.

### Building EDK2

```bash
$ pushd $CLOUDH

# Clone source code repos
$ git clone --depth 1 https://github.com/tianocore/edk2.git -b master
$ cd edk2
$ git submodule update --init
$ cd ..
$ git clone --depth 1 https://github.com/tianocore/edk2-platforms.git -b master
$ git clone --depth 1 https://github.com/acpica/acpica.git -b master

# Build tools
$ export PACKAGES_PATH="$PWD/edk2:$PWD/edk2-platforms"
$ export IASL_PREFIX="$PWD/acpica/generate/unix/bin/"
$ make -C acpica
$ cd edk2/
$ . edksetup.sh
$ cd ..
$ make -C edk2/BaseTools

# Build EDK2
$ build -a AARCH64 -t GCC5 -p ArmVirtPkg/ArmVirtCloudHv.dsc -b RELEASE

$ popd
```

If the build goes well, the EDK2 binary is available at `edk2/Build/ArmVirtCloudHv-AARCH64/RELEASE_GCC5/FV/CLOUDHV_EFI.fd`.

### Booting the guest VM

```bash
$ pushd $CLOUDH
$ sudo RUST_BACKTRACE=1 $CLOUDH/cloud-hypervisor/target/debug/cloud-hypervisor --api-socket /tmp/cloud-hypervisor.sock --kernel $CLOUDH/edk2/Build/ArmVirtCloudHv-AARCH64/RELEASE_GCC5/FV/CLOUDHV_EFI.fd --disk path=$CLOUDH/focal-server-cloudimg-arm64.raw --cpus boot=4 --memory size=4096M --serial tty --console off --log-file log.log -vvv --net tap=,mac=12:34:56:78:90:01,ip=192.168.1.1,mask=255.255.255.0
$ popd
```

## Direct-kernel booting

Alternativelly, you can build your own kernel for guest VM. This way, UEFI is not involved and ACPI cannot be enabled.

### Building kernel

```bash
$ pushd $CLOUDH
$ git clone --depth 1 "https://github.com/cloud-hypervisor/linux.git" -b ch-5.12
$ cd linux
$ cp $CLOUDH/cloud-hypervisor/resources/linux-config-aarch64 .config
$ make -j `nproc`
$ popd
```

### Booting the guest VM

```bash
$ pushd $CLOUDH
$ sudo $CLOUDH/cloud-hypervisor/target/debug/cloud-hypervisor --api-socket /tmp/cloud-hypervisor.sock --kernel $CLOUDH/linux/arch/arm64/boot/Image --disk path=focal-server-cloudimg-arm64.raw --cmdline "keep_bootcon console=ttyAMA0 reboot=k panic=1 root=/dev/vda1 rw" --cpus boot=4 --memory size=4096M --serial tty --console off --log-file log.log -vvv --net "tap=,mac=,ip=,mask="
$ popd
```
