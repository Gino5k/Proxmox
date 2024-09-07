# GPU Passthrough
* Edit `/etc/default/grub` and run `update-grub`
```
GRUB_CMDLINE_LINUX_DEFAULT="iommu=pt initcall_blacklist=sysfb_init nomodeset video=vesa:off"
```
* Edit `/etc/modprobe.d/blacklist.conf` and blacklist AMD GPU and Intel souund modules
```
blacklist snd_hda_intel
blacklist amdgpu
blacklist radeon
```
* Edit ```/etc/modules```and add vfio modules:
```
vfio
vfio_iommu_type1
vfio_pci
```
* Check the GPU device and vendor IDs:
```
lspci -nn | grep -e 'AMD/ATI'
```
* Add the GPU and Integrated HD Audio device IDs to ```/etc/modprobe.d/vfio.conf```:
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
