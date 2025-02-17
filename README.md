# Proxmox

Various notes and pointers for multiple Proxmox experiments.

## Hardware
Current hardware (a bit anaemic in 2025, but dirty cheap and good enough for experimenting) 
* Asus [PRIME B450M-A II](https://www.asus.com/ch-en/motherboards-components/motherboards/prime/prime-b450m-a-ii/) mATX motherboard
* AMD [Ryzen 3 Pro 4350G](https://www.amd.com/en/support/downloads/drivers.html/processors/ryzen-pro/ryzen-pro-4000-series/amd-ryzen-3-pro-4350g.html)

### Rationale
Wanted a low budget APU system with ECC support:
* Integrated GPU -> does not require a discrete GPU for basic usage.
* Relatively low power
* Relatively low noise
* Good enoguh for experimenting with integrated GPU passthrough & ZFS for NAS use

## Proxmox installation
After completing the basic installation, login as root via ssh and execute the following steps:
* Proxmox VE comes with Enterprise repositories configured by default, switch to the non-subscription ones to get proxmox updates:
```
bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/misc/post-pve-install.sh)"
```
* Install the CPU Microcode packages:
```
bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/misc/microcode.sh)"
```
##GPU Passthrough
This is covered in the 
