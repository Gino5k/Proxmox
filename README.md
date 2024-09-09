# GPU Passthrough
* Edit the following line in `/etc/default/grub`: 
```
GRUB_CMDLINE_LINUX_DEFAULT="iommu=pt initcall_blacklist=sysfb_init nomodeset video=vesa:off"
```
* Run `update-grub` to make the above changes effective at the next reboot
* Edit `/etc/modprobe.d/blacklist.conf` and blacklist AMD GPU and Intel souund modules:
```
blacklist snd_hda_intel
blacklist amdgpu
blacklist radeon
```
* Edit `/etc/modules` and add vfio modules:
```
vfio
vfio_iommu_type1
vfio_pci
```
* Note: `vfio_virqfd` is present in multipe guides online, but it should not be required with kernels >= 6.2.x as it is been folded directly into the kernel See for example [this document](https://forum.proxmox.com/threads/pci-gpu-passthrough-on-proxmox-ve-8-installation-and-configuration.130218/) at the section "VFIO modules and verification of remapping support"
* Check the GPU device and vendor IDs:
```
lspci -nn | grep -e 'AMD/ATI'
```
* Add the GPU and Integrated HD Audio device IDs to `/etc/modprobe.d/vfio.conf`:
```
options vfio-pci ids=1002:1636,1002:1637
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

# References
* [Proxmox - Ryzen 7000 series - AMD Radeon 680M/780M/RDNA2/RDNA3 GPU passthrough](https://github.com/isc30/ryzen-7000-series-proxmox?tab=readme-ov-file)
* [Proxmox 5700G APU GPU Passthrough Notes](https://gist.github.com/matt22207/bb1ba1811a08a715e32f106450b0418a)
