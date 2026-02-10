# Hardware Accelerated XFCE4 Desktop in Proxmox LXC (Debian 13)

This guide provides a comprehensive walkthrough for setting up a high-performance, hardware-accelerated RDP workstation using xrdp, xorgxrdp,
and PipeWire audio redirection on an AMD Radeon GPU.

This is based on [A hardware accelerated Gnome workstation running in a container on Proxmox](#ref-pe0alx) page by PE0ALX | Alexander van der Leun, 
plus various errors & retrials and inputs by Gemini.

## Major Differences
1. The main difference is that I use the debian source code for xrdp, xorgxrdp and the piperwire plugin so I can build .deb packages for binary
redistribution and installation.
This gives the possibility of keeping the "build" and the "desktop" containers separate (for a leaner / cleaner image) and also
to "build once, deploy many".
1. Another difference is that for the desktop environment I use a minimal  XFCE4 installation (as opposed to Gnome) to keep the resource
footprint as low as possible.
1. Finally, I also add some extra flags for compilation to enable few extra performance optimizations.

## Baseline system
Debian 13.1 Trixie pve CT template.

## User & Group Creation

_The assumption is that at this stage the container has a "clean" debian install, hence a "sudo" enabled user has not been
created yet and the intial commands are run as root._

First, establish the user identities and security groups. This ensures the services run with minimal necessary privileges.

``` bash
# Install sudo command
apt-get install sudo

# Create the xrdp system user  
groupadd -r xrdp  
useradd -r -g xrdp -d /var/run/xrdp -s /sbin/nologin -c "xrdp daemon" xrdp  
  
# Create the primary user gino (if not already present)  
useradd -m -s /bin/bash -G sudo gino  
passwd gino  
```

## Package Installation

### Update debian sources:

Add ```deb-src```  and ```contrib non-free non-free-firmware``` to ```/etc/apt/sources.list.d/debian.sources```:

``` bash
Types: deb deb-src
URIs: http://deb.debian.org/debian
Suites: trixie trixie-updates
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb deb-src
URIs: http://security.debian.org
Suites: trixie-security
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg  
```

### Target Environment & Diagnostic Tools

These packages are required for the daily operation of the desktop and for verifying hardware acceleration.

``` bash
sudo apt update  
sudo apt-get install --no-install-recommends xfce4 xfce4-terminal xserver-xorg-core \
xorg-video-abi-25 pipewire-bin pipewire-audio-client-libraries pulseaudio-utils \
pipewire wireplumber pipewire-pulse pipewire-audio at-spi2-core openssl ssl-cert \
radeontop vainfo mesa-utils libpam-systemd dbus-x11 dbus-user-session \
libfdk-aac2t64 libpipewire-0.3-common xfce4-pulseaudio-plugin
```

### Compilation & Development Headers

These are the headers and compilers required specifically for building the XRDP stack from source.

``` bash
sudo apt install --no-install-recommends build-essential git autoconf libtool pkg-config nasm \
libssl-dev libpam0g-dev libturbojpeg0-dev libjpeg-dev libx11-dev libxfixes-dev \
libxrandr-dev libpulse-dev libopus-dev libfdk-aac-dev \
xserver-xorg-dev libgbm-dev libepoxy-dev libxfont-dev \
libpipewire-0.3-dev libspa-0.2-dev libva-dev
```

## Hardware & Security Permissions

Link the users to the GPU and SSL subsystems. Adding xrdp to ssl-cert allows the non-root service to read the TLS private keys.

``` bash
sudo usermod -aG video,render gino  
sudo usermod -aG video,render,ssl-cert xrdp  
```

## Compilation Using Debian Sources

Instead of using the ```apt-get source xrdp xorgxrdp pipewire-module-xrdp``` command, get the latest code from debian git repository. 

### Prerequisites

``` bash
# Install packaging tools
sudo apt install -y dpkg-dev devscripts
```

### XRDP

``` bash
# Get the latest debian package source code
git clone https://salsa.debian.org/debian-remote-team/xrdp.git
```

Edit the "debian/rules" file before building to add hw acceleartion and compilation options.
``` bash
cd xrdp-*/

# Edit the "debian/rules" file 
# locate the stanza "override_dh_auto_configure:" and replace it
# the following:

override_dh_auto_configure:
	./bootstrap
	cd librfxcodec && ./bootstrap
	cd libpainter && ./bootstrap
	./configure \
		--prefix=/usr \
		--sysconfdir=/etc \
		--localstatedir=/var \
		--enable-ipv6 \
		--enable-tjpeg \
		--enable-fuse \
		--enable-rfxcodec \
		--enable-opus \
		--enable-painter \
		--enable-pam \
		--enable-pam-config=debian \
		--enable-vsock \
		--enable-vaapi \
		--enable-accel \
		$(shell dpkg-buildflags --export=configure)
	cd librfxcodec && ./configure \
		--disable-shared \
		--enable-static \
		$(shell dpkg-buildflags --export=configure)
	cd libpainter && ./configure \
		--disable-shared \
		--enable-static \
		$(shell dpkg-buildflags --export=configure)
	find . -name Makefile -print0 | \
		xargs -0 perl -pi -e 's!XRDP_PID_PATH.*?/run!$$&/xrdp!g' --

# Build the .deb binary package:

dpkg-buildpackage -rfakeroot -b -uc -us
```

### xorgxrdp (Video Driver)

``` bash
# Get the latest debian package source code
git clone https://salsa.debian.org/debian-remote-team/xorgxrdp.git
```

Edit the "debian/rules" file before building to add hw acceleartion and compilation options.
``` bash
cd ../xorgxrdp-*/

# Edit the "debian/rules" file 
# locate the stanza "override_dh_auto_configure:" and replace it
# the following:

override_dh_auto_configure:
	./bootstrap
	./configure \
		--prefix=/usr \
		--sysconfdir=/etc \
		--localstatedir=/var \
		--enable-glamor \
		--with-simd \
		$(shell dpkg-buildflags --export=configure)
	find . -name Makefile -print0 | \
		xargs -0 perl -pi -e 's!XRDP_PID_PATH.*?/run!$$&/xrdp!g' --

# Build the .deb binary package:

dpkg-buildpackage -rfakeroot -b -uc -us
```
#### PipeWire-XRDP (Audio Bridge)

``` bash
# Get the latest debian package source code
git clone https://salsa.debian.org/debian-remote-team/pipewire-module-xrdp,git
````
No flags to change here, so just build the package:

``` bash
dpkg-buildpackage -rfakeroot -b -uc -us
```

### Install resulting packages
``` bash
cd ...
sudo apt install xorgxrdp_*.deb xrdp_*.deb libpipewire-*.deb pipewire_*.deb
```

Finally, pin the package versions so that they won't get replaced at the next ```apt-get dist-upgrade```:
``` bash
cd ...
sudo apt-mark hold xrdp xorgxrdp pipewire-module-xrdp libpipewire-0.3-modules-xrdp:amd64
```

## Compilation Using Neutrino Sources

### XRDP (Protocol Server)

Configured with high-performance flags and modern audio support.

``` bash
git clone --recursive https://github.com/neutrinolabs/xrdp.git
cd xrdp
./bootstrap
./configure --enable-opus --enable-fdk-aac --enable-ipv6 --enable-tjpeg \
--enable-pixman --enable-vsock --enable-fuse --enable-vaapi --enable-accel
make && sudo make install
```

### xorgxrdp (Video Driver)

This driver provides the bridge between Xorg and the RDP protocol, utilizing Glamor for GPU acceleration.

``` bash
git clone https://github.com/neutrinolabs/xorgxrdp.git  
cd xorgxrdp  
./bootstrap  
./configure --with-simd --enable-glamor
make && sudo make install  
```

### PipeWire-XRDP (Audio Bridge)

This module redirects sound from the container's PipeWire daemon to the RDP client.

``` bash
git clone https://github.com/neutrinolabs/pipewire-module-xrdp.git  
cd pipewire-module-xrdp  
./bootstrap && ./configure  
make && sudo make install  
```
## User Session Setup

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

## System Configuration & Security
### SSL Certificate Generation -> Required Only if Compiling from Neutrino source

Generate a self-signed certificate for TLS security and set correct permissions for the ssl-cert group.

``` bash
sudo openssl req -x509 -newkey rsa:4096 -nodes -keyout /etc/xrdp/key.pem -out /etc/xrdp/cert.pem -days 365  
sudo chown root:ssl-cert /etc/xrdp/key.pem /etc/xrdp/cert.pem  
sudo chmod 640 /etc/xrdp/key.pem  
sudo chmod 644 /etc/xrdp/cert.pem  
```

### XRDP Main Config (/etc/xrdp/xrdp.ini)

Apply cert and non-root ownership settings.

``` ini
[Globals]    
certificate=/etc/xrdp/cert.pem
key_file=/etc/xrdp/key.pem
runtime_user=xrdp
runtime_group=xrdp
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

### XRDP Session Manager Config (/etc/xrdp/sesman.ini)
Apply cert and non-root ownership settings.

``` ini
[Security]   
SessionSockdirGroup=xrdp  
```

### Xorg & Wrapper Config

#### IMPORTANT: Update `/etc/X11/xrdp/xorg.conf` to enable Glamor (hw acceleration):

``` ini
Section "Device"  
    Identifier "xrdpdev"  
    Driver "xrdpdev"  
    Option "GLAMOR" "1"  
    Option "DRMDevice" "/dev/dri/renderD128"  
EndSection  
```

#### IMPORTANT: Update `/etc/X11/Xwrapper.config` to allow non-root X sessions:

``` plaintext
allowed_users=anybody  
needs_root_rights=no  
```

#### Update /etc/xrdp/startwm.sh

When using debian source code, the shipped startwm.sh won't get XFCE4 started. Replace it with the following:

``` bash
#!/bin/sh
if [ -r /etc/default/locale ]; then
  . /etc/default/locale
  export LANG
fi

# Force the use of your custom gino session
if [ -f ~/.xsession ]; then
  exec sh ~/.xsession
fi

# Fallback if .xsession is missing
exec startxfce4
```

### Enable and Start

``` bash
sudo systemctl daemon-reload
sudo systemctl --user --now enable pipewire wireplumber pipewire-pulse
sudo systemctl enable --now xrdp-sesman xrdp
```
If the command fails due to lack of systemd scripts, use the templates in the next section.

### Systemd Service Deployment

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

## Verification

Log in via RDP and run the following commands to confirm everything is working:

  * **Audio:** `pactl list sinks short` (Look for `xrdp-sink`).
  * **GPU Rendering:** `glxinfo | grep "renderer"` (Should show **AMD Radeon**).
  * **VA-API Acceleration:** `vainfo` (Should list profiles without using sudo).
  * **GPU Load:** `radeontop` (Watch for activity during video playback).

#References
1. <a id="ref-pe0alx"></a>[A hardware accelerated Gnome workstation running in a container on Proxmox](https://pe0alx.nl/2025/04/a-hardware-accelerated-gnome-workstation-running-in-a-container-on-proxmox/)
