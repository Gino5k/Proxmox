# GPU Passthrough
## Enable IOMMU in the BIOS
Via the following options:
* SVM mode
* SR-IOV
* IOMMU

## Enable IOMMMU support in Proxmox
* Edit the following line in `/etc/default/grub`: 
```
GRUB_CMDLINE_LINUX_DEFAULT="iommu=pt initcall_blacklist=sysfb_init nomodeset video=vesa:off"
```
* Run `update-grub` to make the above changes effective at the next reboot
* Blacklist AMD GPU and Intel souund modules in `/etc/modprobe.d/blacklist.conf`:
```
echo "blacklist snd_hda_intel" >> /etc/modprobe.d/blacklist.conf
echo "blacklist amdgpu"  >> /etc/modprobe.d/blacklist.conf
echo "blacklist radeon" >>  >> /etc/modprobe.d/blacklist.conf
```
* **Alternative** to the above: use `sofdep`:
```
echo "softdep radeon pre: vfio-pci" >> /etc/modprobe.d/vfio.conf
echo "softdep amdgpu pre: vfio-pci" >> /etc/modprobe.d/vfio.conf
echo "softdep snd_hda_intel pre: vfio-pci" >> /etc/modprobe.d/vfio.conf
```
* Add vfio modules in `/etc/modules`:
```
echo "vfio" >> /etc/modules
echo "vfio_iommu_type1" >> /etc/modules
echo "vfio_pci" >> /etc/modules
```
* Note: `vfio_virqfd` is present in multipe guides online, but it should not be required with kernels >= 6.2.x as it is been folded directly into the vfio module. See for example [this document](https://forum.proxmox.com/threads/pci-gpu-passthrough-on-proxmox-ve-8-installation-and-configuration.130218/) at the section "VFIO modules and verification of remapping support"
* Check the GPU device and vendor IDs:
```
lspci -nn | grep -e 'AMD/ATI'
```
* Add the GPU and Integrated HD Audio device IDs to `/etc/modprobe.d/vfio.conf`:
```on 
echo "options vfio-pci ids=1002:1636,1002:1637" >> /etc/modprobe.d/vfio.conf
```
* Rebuild the initrd image and reboot:
```
update-initramfs -u -k all
shutdown -r now
```
* After rebooting , Verify IOMMU is enabled:
```
dmesg | grep -i -e DMAR -e IOMMU
```
## VM Configuration in Proxmox
* See the steps in the ["Creating the Windows VM"](https://github.com/isc30/ryzen-7000-series-proxmox?tab=readme-ov-file#creating-the-windows-vm) section of this document.
## Configuring the GPU in the Windows VM
For Ryzen APU passthrough to work *two* files are required:
1. The VGA vBIOS
2. The "AMDGopDriver" 

Neither of the two methods below, which are typycally used to extract the vBIOS card, worked in my case:
* Using the `vbios.c` file as reported [here](https://github.com/isc30/ryzen-7000-series-proxmox?tab=readme-ov-file#configuring-the-gpu-in-the-windows-vm)
* Using [GPU-Z](https://www.techpowerup.com/download/techpowerup-gpu-z/) "save bios" function runing on baremetal Windows 11.

### Extracting vBIOS and AMDGopDriver with UBU
The only way that really worked in case was to use the [UBU utility](https://winraid.level1techs.com/t/tool-guide-news-uefi-bios-updater-ubu/30357) to extract the required files from the BIOS motherboard file provided by Asus.

This method, which is based on the notes reported [here]() also has the following advatages:
1. It enables extraction of *both* the vBIOS and the AMDGopDriver files from the vendor files, as opposed to have to download the second one from random sources
2. It ensures the files that will be paased to the VM are in sync with the motherbaord model and BIOS version.

Still, it relies on UBU, a 3rd party unverified BIOS extracting utility, so it certainly introduces some unknowns... Use it at your own risk :-). 


# References
* [Proxmox - Ryzen 7000 series - AMD Radeon 680M/780M/RDNA2/RDNA3 GPU passthrough](https://github.com/isc30/ryzen-7000-series-proxmox?tab=readme-ov-file)
* [Proxmox 5700G APU GPU Passthrough Notes](https://gist.github.com/matt22207/bb1ba1811a08a715e32f106450b0418a)
