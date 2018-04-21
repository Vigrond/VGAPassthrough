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
Install latest QEMU

TODO:  Add instructions for virtualization package updates

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
kvm
kvm_intel
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

use virt-manager to create a vm with:

* Q35 chipset
* UEFI firmware
* 'host-passthrough' CPU model
* SATA drives (not ide)
* Add PCI hardware -> select GPU

Save / start / install windows
Download and extract NVIDIA drivers ( I used 391.35)
Do not install yet
Install VIRTIO windows drivers

Follow instructions here https://github.com/sk1080/nvidia-kvm-patcher to finalize installing Nvidia drivers
* Install WDK (step 2 here: https://docs.microsoft.com/en-us/windows-hardware/drivers/download-the-wdk)
* Enable test mode - `Bcdedit.exe -set TESTSIGNING ON` - then restart.  (After restart confirm test mode text at bottom right of windows desktop screen)
* Run the patcher
* Run the driver setup.  Snafu:  Permissions issues.  For some reason the NVIDIA extractor fucks up the permissions on C:\NVIDIA\...International.  You need to right click the folder / go to security / ensure Administrators have full permissions and all children of the directory do as well.  Repeat after me... Windows is love, Windows is life.

Now you can continue on with setting up your preferred input / monitor config.



### Windows VIRTIO drivers
https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso
