# Binary files
This folder contains the binary files extracted from the motherboard ([Asus PRIME B450M-A II](https://www.asus.com/ch-en/motherboards-components/motherboards/prime/prime-b450m-a-ii/)) BIOS bundles.

There is one subdir for each motherboard BIOS version. Each subdir contain the following files:
* ```vbios_1636.dat```: The vBIOS file for the AMD 4350G Pro GPU, device ID: 1636
* ```AMDGopDriver.efi```: The GOP BIOS file required for the integrated HD Audio device
* ```AMDGopDriver.rom```: Same as above, but in .ROM format
* ```PB450MA2_<version>.cap```: The motherboad BIOS file in .CAP format extracted from the BIOS `<version>` ZIP archive. This can be used with [UBU](https://winraid.level1techs.com/t/tool-guide-news-uefi-bios-updater-ubu/30357) to upack the full set of files.

## Versions
* [BIOS Version 4622](4622/) - 2024/11/14, 15.77 MB 
* [BIOS Version 4604](4604/) - 2024/04/08, 15.77 MB
