# Hardware Accelerated XFCE4 Desktop in Proxmox LXC (Debian 13)

This guide provides a comprehensive walkthrough for setting up a high-performance, hardware-accelerated RDP workstation using xrdp, xorgxrdp, and PipeWire audio redirection on an AMD Radeon GPU.

This is based on [A hardware accelerated Gnome workstation running in a container on Proxmox](https://pe0alx.nl/2025/04/a-hardware-accelerated-gnome-workstation-running-in-a-container-on-proxmox/)
page by PE0ALX | Alexander van der Leun. Plus various errors & retrials and inputs by Gemini.

## 1. User & Group Creation

First, establish the user identities and security groups. This ensures the services run with minimal necessary privileges.

``` bash
# Create the xrdp system user  
sudo groupadd -r xrdp  
sudo useradd -r -g xrdp -d /var/run/xrdp -s /sbin/nologin -c "xrdp daemon" xrdp  
  
# Create the primary user gino (if not already present)  
sudo useradd -m -s /bin/bash -G sudo gino  
sudo passwd gino  
```

## 2. Package Installation

### A. Update debian sources:

Add ```contrib non-free non-free-firmware``` to ```/etc/apt/sources.list.d/debian.sources```:

``` bash
Types: deb deb-src
URIs: http://deb.debian.org/debian
Suites: trixie trixie-updates
Components: contrib main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb deb-src
URIs: http://security.debian.org
Suites: trixie-security
Components: contrib main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg  
```

### A. Target Environment & Diagnostic Tools

These packages are required for the daily operation of the desktop and for verifying hardware acceleration.

``` bash
sudo apt update  
sudo apt install -y xfce4 xfce4-goodies xserver-xorg-core xorg-video-abi-25 \
pipewire-bin pipewire-audio-client-libraries pulseaudio-utils at-spi2-core \
openssl ssl-cert radeontop vainfo mesa-utils  
```

### B. Compilation & Development Headers

These are the headers and compilers required specifically for building the XRDP stack from source.

``` bash
sudo apt install -y build-essential git autoconf libtool pkg-config nasm \
libssl-dev libpam0g-dev libturbojpeg0-dev libjpeg-dev libx11-dev libxfixes-dev \
libxrandr-dev libpulse-dev libopus-dev libfdk-aac-dev \
xserver-xorg-dev libgbm-dev libepoxy-dev libxfont-dev \
libpipewire-0.3-dev libspa-0.2-dev libva-dev
```

## 3. Hardware & Security Permissions

Link the users to the GPU and SSL subsystems. Adding xrdp to ssl-cert allows the non-root service to read the TLS private keys.

``` bash
sudo usermod -aG video,render gino  
sudo usermod -aG video,render,ssl-cert xrdp  
```

## 4. Source Compilation
### A. XRDP (Protocol Server)

Configured with high-performance flags and modern audio support.

``` bash
git clone --recursive https://github.com/neutrinolabs/xrdp.git
cd xrdp
./bootstrap
./configure --enable-opus --enable-fdk-aac --enable-ipv6 --enable-tjpeg \
--enable-pixman --enable-vsock --enable-fuse --enable-vaapi --enable-accel
make && sudo make install
```

### B. xorgxrdp (Video Driver)

This driver provides the bridge between Xorg and the RDP protocol, utilizing Glamor for GPU acceleration.

``` bash
git clone https://github.com/neutrinolabs/xorgxrdp.git  
cd xorgxrdp  
./bootstrap  
./configure --with-simd --enable-glamor
make && sudo make install  
```

### C. PipeWire-XRDP (Audio Bridge)

This module redirects sound from the container's PipeWire daemon to the RDP client.

``` bash
git clone https://github.com/neutrinolabs/pipewire-module-xrdp.git  
cd pipewire-module-xrdp  
./bootstrap && ./configure  
make && sudo make install  
```
### D. Alternative: Use Debian Sources
In case of issues compiling the latest code source from GitHub, the debian source packages could be use:

``` bash
# Install packaging tools
sudo apt install -y dpkg-dev devscripts

# Get dependencies and source
sudo apt-get build-dep xrdp xorgxrdp pipewire-module-xrdp
apt-get source xrdp xorgxrdp pipewire-module-xrdp

# Note: To add --enable-glamor or --enable-vaapi here, 
# it may be necessry to edit the "debian/rules" file before building.

cd xrdp-*/
dpkg-buildpackage -rfakeroot -b -uc -us
cd ../xorgxrdp-*/
dpkg-buildpackage -rfakeroot -b -uc -us
cd ../pipewire-*/
dpkg-buildpackage -rfakeroot -b -uc -us

# Install resulting packages
sudo apt install ../xrdp_*.deb ../xorgxrdp_*.deb ../pipewire_*.deb
```

## 5. User Session Setup

Run these commands as the user to configure the XFCE environment. This must be done before the service is initialized to ensure proper session handling.

``` bash
# Create .xsessionrc for environment variables  
cat <<EOF > ~/.xsessionrc  
export DESKTOP_SESSION=xfce  
export XDG_CURRENT_DESKTOP=XFCE  
export XDG_SESSION_TYPE=x11  
EOF  
  
# Create .xsession to launch the DE  
cat <<EOF > ~/.xsession  
exec startxfce4  
EOF  
  
# Ensure scripts are executable  
chmod +x ~/.xsession ~/.xsessionrc  
```

## 6. System Configuration & Security
### A. SSL Certificate Generation

Generate a self-signed certificate for TLS security and set correct permissions for the ssl-cert group.

``` bash
sudo openssl req -x509 -newkey rsa:4096 -nodes -keyout /etc/xrdp/key.pem -out /etc/xrdp/cert.pem -days 365  
sudo chown root:ssl-cert /etc/xrdp/key.pem /etc/xrdp/cert.pem  
sudo chmod 640 /etc/xrdp/key.pem  
sudo chmod 644 /etc/xrdp/cert.pem  
```

### B. XRDP Main Config (/etc/xrdp/xrdp.ini)

Apply cert and non-root ownership settings.

``` ini
[Globals]    
certificate=/etc/xrdp/cert.pem  
key_file=/etc/xrdp/key.pem  
runtime_user=xrdp  
runtime_group=xrdp  
SessionSockdirGroup=xrdp  
```

Verfiy performance settings - in most cases they should be already set.

``` ini
[Globals]  
use_fastpath=yes  
max_bpp=32  
allow_channels=true  
rdpdr=true  
rdpsnd=true  
```

### C. Xorg & Wrapper Config

Update `/etc/X11/xrdp/xorg.conf` to enable Glamor:

``` ini
Section "Device"  
    Identifier "xrdpdev"  
    Driver "xrdpdev"  
    Option "GLAMOR" "1"  
    Option "DRMDevice" "/dev/dri/renderD128"  
EndSection  
```

Update `/etc/X11/Xwrapper.config` to allow non-root X sessions:

``` plaintext
allowed_users=anybody  
needs_root_rights=no  
```

### D. Enable and Start

``` bash
sudo systemctl daemon-reload  
sudo systemctl enable --now xrdp-sesman xrdp  
```
If the command fails due to lack of systemd scripts, use the templates in the next section.

### E. Systemd Service Deployment

Create the service files with proper dependency tracking.

**File: `/lib/systemd/system/xrdp-sesman.service`**

``` ini
[Unit]  
Description=xrdp session manager  
After=network.target  
  
[Service]  
Type=forking  
PIDFile=/var/run/xrdp-sesman.pid  
ExecStart=/usr/local/sbin/xrdp-sesman  
  
[Install]  
WantedBy=multi-user.target  
```

**File: `/lib/systemd/system/xrdp.service`**

``` ini
[Unit]  
Description=xrdp daemon  
After=network.target xrdp-sesman.service  
Requires=xrdp-sesman.service  
  
[Service]  
User=xrdp  
Group=xrdp  
Type=forking  
PIDFile=/var/run/xrdp.pid  
ExecStart=/usr/local/sbin/xrdp  
  
[Install]  
WantedBy=multi-user.target  
```
Then [enable the scripts](DesktopLXC-GPU-Sound.md#d-enable-and-start)

## 7. Verification

Log in via RDP and run the following commands to confirm everything is working:

  * **Audio:** `pactl list sinks short` (Look for `xrdp-sink`).
  * **GPU Rendering:** `glxinfo | grep "renderer"` (Should show **AMD Radeon**).
  * **VA-API Acceleration:** `vainfo` (Should list profiles without using sudo).
  * **GPU Load:** `radeontop` (Watch for activity during video playback).
