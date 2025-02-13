# Proxmox

Various notes and pointers for multiple Proxmox experiments.

## Hardware
Current hardware (a bit anaemic in 2025, but dirty cheap and good enough for experimenting) 
* Asus [PRIME B450M-A II](https://www.asus.com/ch-en/motherboards-components/motherboards/prime/prime-b450m-a-ii/) mATX motherboard
* AMD [Ryzen 3 Pro 4350G](https://www.amd.com/en/support/downloads/drivers.html/processors/ryzen-pro/ryzen-pro-4000-series/amd-ryzen-3-pro-4350g.html)

## Rationale
Wanted a low budget APU system with ECC support:
* Integrated GPU -> no need for descrete GPU
* Relatively low power
* Low noise
* Enables experimenting with integrated GPU passthrough & ZFS for NAS use
