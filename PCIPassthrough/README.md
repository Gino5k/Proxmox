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
The instructions [here](https://github.com/isc30/ryzen-7000-series-proxmox?tab=readme-ov-file#configuring-the-gpu-in-the-windows-vm) are generally applicable, with the following caveats:
  1. For Ryzen APU passthrough the section marked as [Optional](https://github.com/isc30/ryzen-7000-series-proxmox?tab=readme-ov-file#configuring-the-gpu-in-the-windows-vm) is requird, i.e.: *two* files must be passed to the VM:
  * The VGA vBIOS
  * The "AMDGopDriver"
 2. Neither of theese methods to extract the vBIOS worked in my case:
  * Using the `vbios.c` file as reported [here](https://github.com/isc30/ryzen-7000-series-proxmox?tab=readme-ov-file#configuring-the-gpu-in-the-windows-vm)
  * Using [GPU-Z](https://www.techpowerup.com/download/techpowerup-gpu-z/) "save bios" function running on baremetal Windows 11.

The only way that really was to use the [UBU utility](https://winraid.level1techs.com/t/tool-guide-news-uefi-bios-updater-ubu/30357) to extract the required files from the BIOS motherboard file provided by the hardware vendor.

### Extracting vBIOS and AMDGopDriver with UBU
This method, which is based on the note reported [here](https://gist.github.com/matt22207/bb1ba1811a08a715e32f106450b0418a?permalink_comment_id=4955044#gistcomment-4955044) also has the following advantages:
1. It enables extraction of *both* the vBIOS and the AMDGopDriver files from the vendor provided BIOS motherboard file. No need to download the AMDGopDriver from random sources.
2. It ensures the files that will be passed to the VM are in sync with the motherbaord model and BIOS version.

Still, it relies on UBU, a 3rd party unverified BIOS extracting utility, so it certainly introduces some unknowns... Use it at your own risk 😃.

To extract the required files:
* Download the the motherboard BIOS update from the Vendor site. Make sure to use the same BIOS version as the one flashed on the motherboard.
   * Example: `PRIME-B450M-A-II-ASUS-4622.zip`
* Download and extract the UBU utility to a directory
* The UBU utility expects a file named `bios.bin` in this directory
  * Unzip the BIOS file (e.g.: `PRIME-B450M-A-II-ASUS-4622.zip`), copy its content (`PRIME-B450M-A-II-ASUS-4622.cap`) to thre UBU directory and rename it as `bios.bin`, then run `ubu.cmd`
  *  The UBU utility will extract the file and present a set of interactive menus. Make sure to select the "Video OnBoard" option and then the "Extracted" option (to save the files) E.g.:  
```
                      Main Menu
            [Current version in BIOS file]
1 - Disk Controller
     EFI AMD RAIDXpert2-Fxx      - 9.3.0-00308
     EFI NVMe Driver present
2 - Video OnBoard
     EFI AMD GOP Driver          - 2.23.0.17.10_NoSign
     OROM VBIOS Cezanne          - 017.010.000.030.000000
     OROM VBIOS Renoir           - 017.010.000.030.000000
     OROM VBIOS Renoir           - 017.010.000.030.000000
3 - Network
     EFI Realtek UNDI Driver     - 2.041
     OROM Realtek Boot Agent GE  - 2.67
4 - Other SATA Controller
5 - CPU MicroCode
     View/Extract/Search/Replace
S - AMI Setup IFR Extractor
0 - Exit
RS - Re-Scanning
A - About
Choice: 2


                Video OnBoard
        [Current version]
     EFI AMD GOP Driver          - 2.23.0.17.10_NoSign
     OROM VBIOS Cezanne          - 017.010.000.030.000000
     OROM VBIOS Renoir           - 017.010.000.030.000000
     OROM VBIOS Renoir           - 017.010.000.030.000000

        [Available version]
     EFI AMD GOP Driver          - 2.23.0.17.10_NoSign
     OROM VBIOS Cezanne          - 017.010.000.030.000000
     OROM VBIOS Renoir           - 017.010.000.030.000000

1 - Replace GOP Driver
2 - Replace OROM VBIOS
X - Extracted
0 - Exit to Main Menu
Choice:X
        1 file(s) copied.
        1 file(s) copied.
        1 file(s) copied.
        1 file(s) copied.
Press any key to continue . . .
```
* When UBU exists, thre will be an `Extracted`  folder with two subfolders, one for the vBIOS and one for GOP driver, respectively. These contains the required files!
* For the AMDGopDriver, one extra step is required: UBU produces a file in `efi` format, but according to the [source notes](https://github.com/isc30/ryzen-7000-series-proxmox?tab=readme-ov-file#optional-getting-ovmf-uefi-bios-working-error-43) a file in `rom` format is required instead.
* A `rom` file can be gerated following the procedure [here](https://gist.github.com/matt22207/bb1ba1811a08a715e32f106450b0418a?permalink_comment_id=4955044#gistcomment-4955044). I.e.:
  * Download `EfiRom.exe` from [here](https://github.com/tianocore/edk2-BaseTools-win32)
  * Get the the Vendor and Device ID for the Audio Controller by running `lspci -nn | grep -e 'AMD/ATI'`, e.g.: 
    ```
    lspci -nn | grep -e 'AMD/ATI'
    09:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Renoir [1002:1636] (rev da)
    09:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Renoir Radeon High Definition Audio Controller [1002:1637]
    ```
    In my case 1002 is the Vendor ID and 1637 is Device ID.
  * Change to directory where the `AMDGopDriver.efi` is located and run `EfiRom.exe -f VendorId -i DeviceId -e AMDGopDriver.efi -o AMDGopDriver.rom`. E.g.:
    ```
    EfiRom.exe -f 1002 -i 1637 -e AMDGopDriver.efi -o AMDGopDriver.rom
    ```
* The file `AMDGopDriver.rom` and the file `vbios_1636.dat` (**make sure to select the one matching the GPU device ID if there are multiple files!** can be copied to the ProxMox host in the directory `/usr/share/kvm/" so it can be referenced in the VM config.

# References
* [Proxmox - Ryzen 7000 series - AMD Radeon 680M/780M/RDNA2/RDNA3 GPU passthrough](https://github.com/isc30/ryzen-7000-series-proxmox?tab=readme-ov-file)
* [Proxmox 5700G APU GPU Passthrough Notes](https://gist.github.com/matt22207/bb1ba1811a08a715e32f106450b0418a)
