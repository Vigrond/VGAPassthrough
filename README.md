# VGAPassthrough Guide

## Machine
Host: Ubuntu 14.04
Guest: Windows 10

Mobo: MSI Z97M GAMING
CPU: Intel(R) Core(TM) i7-4790K CPU @ 4.00GHz
GPU: NVIDIA Corporation GM204 [GeForce GTX 970]

Mobo has integrated intel graphics

Two monitors.  One using Display Port @ 144hz.  One using HDMI.

## Setup

### OVMF
Install latest OVMF
http://archive.ubuntu.com/ubuntu/ubuntu/pool/universe/e/edk2/

"ovmf_0~20180205.c0d9813c-2_all.deb"

### QEMU
Build latest QEMU using docker

https://wiki.qemu.org/Testing/DockerBuild

```
curl -fsSL get.docker.com -o get-docker.sh
sudo sh get-docker.sh
# Might need to restart Ubuntu here

git clone git://git.qemu-project.org/qemu.git
cd qemu
make docker-test-quick@ubuntu
...this might take a while..like...do it overnight or go get lunch
```
### Enable IOMMU
Ensure `GRUB_CMDLINE_LINUX_DEFAULT` contains `intel_iommu=on`
```
sudo gedit /etc/default/grub

...
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_iommu=on"
...
```
Then
```
sudo update-grub
```
Restart your computer

### Check IOMMU
```
dmesg | grep -e DMAR -e IOMMU

...
[    0.000000] DMAR: IOMMU enabled
...
# Looks good
```

### Collect VGA identifiers
```
lspci -nnk
...
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM204 [GeForce GTX 970] [10de:13c2] (rev a1)
...
01:00.1 Audio device [0403]: NVIDIA Corporation GM204 High Definition Audio Controller [10de:0fbb] (rev a1)
...
```
Copy/paste these two items somewhere.  Make sure no other GPUs are on the 01:00 group

### Assign devices to VFIO
Take note of the IDs from previous step.

`/etc/modprobe.d/vfio.conf` should look like:
```
options vfio-pci ids=10de:13c2,10de:0fbb disable_vga=1
softdep nouveau pre: vfio-pci
```

`/etc/modules-load.d/vfio.conf` should look like:
```
vfio_pci
vfio
vfio_iommu_type1
vfio_virqfd
```

### Check VFIO
Restart computer
```
lspci -nnk

01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM204 [GeForce GTX 970] [10de:13c2] (rev a1)
	Subsystem: eVga.com. Corp. GM204 [GeForce GTX 970] [3842:3975]
	Kernel driver in use: vfio-pci
	Kernel modules: nvidiafb, nouveau
01:00.1 Audio device [0403]: NVIDIA Corporation GM204 High Definition Audio Controller [10de:0fbb] (rev a1)
	Subsystem: eVga.com. Corp. GM204 High Definition Audio Controller [3842:3975]
	Kernel driver in use: vfio-pci
	Kernel modules: snd_hda_intel
  ```
Note the kernel driver in use is `vfio-pci`.  Looks good.

If you installed the NVIDIA driver, you may want to disable it by going to `Additional Settings` and switching it to nouveau to help out the `vfio-pci` device capture

### Setup the VM
Let's create a config file and call it `vm.sh`

```
# Convenience script for assigning all the cores:
export CORES_NUMBER=$(lscpu --all -p=CORE | grep -v ^# | sort | uniq | wc -l)
# Assigned memory, in MiB:
export VGAPT_MEMORY=8192

export VGAPT_VGA_ID='10de:13c2'
export VGAPT_VGA_AUDIO_ID='10de:0fbb'
export VGAPT_VGA_BUS=01:00.0
export VGAPT_VGA_AUDIO_BUS=01:00.1

# Standard locations from the Ubuntu `ovmf` package; last one is arbitrary:
export VGAPT_FIRMWARE_BIN=/usr/share/OVMF/OVMF_CODE.fd
export VGAPT_FIRMWARE_VARS=/usr/share/OVMF/OVMF_VARS.fd
export VGAPT_FIRMWARE_VARS_TMP=/tmp/OVMF_VARS.fd.tmp

# Standard location from the Ubuntu `qemu-system-x86` package:
export QEMU_BINARY=/usr/bin/qemu-system-x86_64

export VGAPT_DISK_IMAGE=/path/to/virtual_disk.img
export VGAPT_WINDOWS_ISO=/path/to/win_installation.iso
export VGAPT_VIRTIO_DRIVERS_ISO=/path/to/virtio-win.iso

export VGAPT_SHARED_FOLDERS=/path/to/shared_folders

cp -f $VGAPT_FIRMWARE_VARS $VGAPT_FIRMWARE_VARS_TMP &&
$QEMU_BINARY \
  -drive if=pflash,format=raw,readonly,file=$VGAPT_FIRMWARE_BIN \
  -drive if=pflash,format=raw,file=$VGAPT_FIRMWARE_VARS_TMP \
  -enable-kvm \
  -machine q35,accel=kvm,mem-merge=off \
  -cpu host,kvm=off,hv_vendor_id=vgaptrocks,hv_relaxed,hv_spinlocks=0x1fff,hv_vapic,hv_time \
  -smp $CORES_NUMBER,cores=$CORES_NUMBER,sockets=1,threads=1 \
  -m $VGAPT_MEMORY \
  -display gtk -vga qxl \
  -rtc base=localtime \
  -serial none -parallel none \
  -usb \
  -device vfio-pci,host=$VGAPT_VGA_BUS,multifunction=on \
  -device vfio-pci,host=$VGAPT_VGA_AUDIO_BUS \
  -device virtio-scsi-pci,id=scsi \
  -drive file=$VGAPT_DISK_IMAGE,id=disk,format=qcow2,if=none,cache=writeback -device scsi-hd,drive=disk \
  -drive file=$VGAPT_WINDOWS_ISO,id=scsicd,format=raw,if=none -device scsi-cd,drive=scsicd \
  -drive file=$VGAPT_VIRTIO_DRIVERS_ISO,id=idecd,format=raw,if=none -device ide-cd,bus=ide.1,drive=idecd \
  -net nic,model=virtio \
  -net user,smb=$VGAPT_SHARED_FOLDERS \
```

### Windows VIRTIO drivers
https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso
