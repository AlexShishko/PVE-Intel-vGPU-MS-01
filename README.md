# Intel Gen 13 vGPU (SR-IOV) on Proxmox on ms-01

This guide is designed to help you virtualize the 13th-generation Intel integrated GPU (iGPU) and share it as a virtual GPU (vGPU) with hardware acceleration and video encoding/decoding capabilities across multiple VMs.

### Disclaimer

This workaround is not officially supported by Proxmox. Use at your own risk.

### Environment

The environment used for this guide:

• Model: Minisforum MS-01

• CPU: 13th Gen Intel(R) Core(TM) i9-13900H

• RAM: 64GB DDR5

• linux: 6.8.12-4-pve

• Proxmox: 8.3


### Prerequisites

Before proceeding, ensure the following:

• VT-d (IOMMU) and SR-IOV are enabled in BIOS.

• Proxmox Virtual Environment (VE) version 8.1.4 or newer is **installed with GRUB bootloader**.

• EFI is **enabled**, and Secure Boot is **disabled**. (Please refer to **Appendix 2** during the guide if your Secure Boot is Enabled)

• Linux kernel version 6.1 or newer is present. Validate with 
```bash
uname -r
```

If your kernel is older than 6.1, refer to **[Appendix 1](#appedix-1---kernal-lower-than-61)** for instructions on updating it.

## Proxmox Setup
### DKMS Setup
1. Update your system and install required packages:
```bash
apt update && apt install pve-headers-$(uname -r)
apt update && apt install git pve-headers mokutil
rm -rf /var/lib/dkms/i915-sriov-dkms*
rm -rf /usr/src/i915-sriov-dkms*
rm -rf ~/i915-sriov-dkms
KERNEL=$(uname -r); KERNEL=${KERNEL%-pve}
```
2. Proceed to clone the DKMS repository and adjust its configuration:
```bash
cd ~
git clone https://github.com/strongtz/i915-sriov-dkms.git
cd ~/i915-sriov-dkms
cp -a ~/i915-sriov-dkms/dkms.conf{,.bak}
sed -i 's/ -j$(nproc)//g' ~/i915-sriov-dkms/dkms.conf
sed -i 's/GUCFIRMWARE_MINOR:-9/GUCFIRMWARE_MINOR:-13/' Makefile
dkms_ver=$(grep 'PACKAGE_VERSION=' dkms.conf | awk -F '=' '{print $2}' | tr -d '"')
cat ~/i915-sriov-dkms/dkms.conf
```
3. Install the DKMS module:
```bash
apt install --reinstall dkms -y
dkms add .
cd /usr/src/i915-sriov-dkms-*
dkms status
```
4. Build the i915-sriov-dkms
```bash
dkms install -m i915-sriov-dkms -v $dkms_ver -k $(uname -r) --force -j 1
dkms status
```

### GRUB Bootloader Setup
Backup and update the GRUB configuration:
```bash
cp -a /etc/default/grub{,.bak}
sudo sed -i '/^GRUB_CMDLINE_LINUX_DEFAULT/c\GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt i915.enable_guc=3 i915.max_vfs=7"' /etc/default/grub
update-grub
update-initramfs -u -k all
apt install sysfsutils -y
```

### Create vGPU
Identify the PCIe bus number of the VGA card, usually **00:02.0**: 
```bash
lspci | grep VGA
```

Edit the sysfs configuration to enable the vGPUs. In this case I’m using **00:02.0**. To verify the file was modified, cat the file and ensure it was modified.
```bash
echo "devices/pci0000:00/0000:00:02.0/sriov_numvfs = 7" > /etc/sysfs.conf
cat /etc/sysfs.conf
```

### Secure Boot Configuration (Optional)
If you are using Secure Boot, please follow the **[Appedix 2](#appedix-2---efi--enabled-and-secure-boot-enabled)** before next step

### Completing  vGPU setup
Now reboot your Host
```bash
reboot
```
And you should able to see the minor **PCIe IDs 1-7** and finally **Enabled 7 VFs**
```bash
lspci | grep VGA
dmesg | grep i915
```
Now your host is prepared, and you can set up Windows 10/11 and Linux VMs with SR-IOV vGPU support.

## On Windows VM:
Set pci-e device to
![изображение](https://github.com/user-attachments/assets/e93e9630-3d6a-474c-870b-b161b73e0333)
## On Linux VM:
You need to compile DKMS module in to VM.
So, go to DKMS Setup part of guide and repeat it in VM

Change:
```
apt update && apt install pve-headers-$(uname -r)
apt update && apt install git pve-headers mokutil
```
to
```
apt update && apt install linux-headers-$(uname -r)
apt update && apt install git  mokutil
```
After installation, set pci-e device to 
![изображение](https://github.com/user-attachments/assets/d7c775a0-58f5-4370-9d9e-5cf8e49feacc)

Now, you are able to use GPU to accelerate workloads (like jellyfin hardware transcoding)
add Primary GPU if you want to use GPU in GUI acceleration
## Appedixies
### Appedix 1 - Kernal lower than 6.1
You can update the PVE kernel to 6.2 5.19 using these commands:
1.  Disable enterprise repo:
```bash
vi /etc/apt/sources.list
```
Then use the following repos from Proxmox:  
(https://pve.proxmox.com/wiki/Package_Repositories)  
My sources.list looks like this:  
```bash
deb http://ftp.uk.debian.org/debian bookworm main contrib
deb http://ftp.uk.debian.org/debian bookworm-updates main contrib
deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription
# security updates
deb http://security.debian.org bookworm-security main contrib
```
2. Install 6.5 kernel.
```bash
apt install pve-kernel-6.5
```
3.  Update apt and install headers.
```bash
apt update && apt install pve-headers-$(uname -r)
```
4. Update initramfs and reboot.\
```bash
update-initramfs -u
reboot
```
5. Check your kernel
```bash
uname -r
```
   
**[Back to the Guide where you left](#proxmox-setup)**

### Appedix 2 - EFI  **Enabled** and Secure Boot **Enabled**
You need to install mokutill for Secure Boot
```bash
apt update && apt install mokutil
mokutil --import /var/lib/dkms/mok.pub
```
**Reboot Proxmox Host**, monitor the boot process and wait for the **Perform MOK management** window (screenshot below). If you miss the first reboot you will need to re-run the mokutil command and reboot again. The DKMS module will NOT load until you step through this setup. 

Secure Boot MOK Configuration (Proxmox 8.1+)

Select **Enroll MOK, Continue, Yes, (password), Reboot.**

**[Back to the Guide where you left](#secure-boot-configuration-optional)**

## Conclusion
By completing this guide, you should be able to share your Intel Gen 12 iGPU across multiple VMs for enhanced video processing and virtualized graphical tasks within a Proxmox environment.


## Credits

The DKMS module by Strongz is instrumental in making this possible ([i915-sriov-dkms GitHub repository](https://github.com/strongtz/i915-sriov-dkms?ref=michaels-tinkerings)).  
Additionally, Derek Seaman and Michael's blog post was an inspirational resource ([Derek Seaman’s Proxmox vGPU Guide](https://www.derekseaman.com/2023/11/proxmox-ve-8-1-windows-11-vgpu-vt-d-passthrough-with-intel-alder-lake.html) & [vGPU (SR-IOV) with Intel 12th Gen iGPU](https://www.michaelstinkerings.org/gpu-virtualization-with-intel-12th-gen-igpu-uhd-730/)).  
This installer is tailored for Proxmox 8, aiming to streamline the installation process. 

## License
This project is licensed under Apache License, Version 2.0.

## Author
Nova Upinel Chow  
Email: dev@upinel.com

If you wish to donate original author go to a root project
