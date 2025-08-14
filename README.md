# wifi7_ubuntu_setup
Early Testing of WiFi-7 on Ubuntu for Robotic Applications

The software ecosystem supporting the new 802.11be standard is still a working progress. Although the Intel driver supports WiFi7, the released and integrated wpa_supplicant, Linux network manager, and GUI don't support these new features.

This repository provides a brief guide on how to install and configure the system to test the new hardware on Linux.


## Hardware:
 - WiFi 7 cards: Intel BE200/BE201/BE202
 - Access point: UniFi E7 with [experimental firmware](https://help.ui.com/hc/en-us/articles/25656226682775-Multi-Link-Operation-MLO-in-UniFi-Network) that supports MLO.

Check the hardware model:
```bash
sudo journalctl -k | grep -iE "iwlwifi.*Detected Intel"
```
You will see a line like:
```
iwlwifi ...: Detected Intel(R) Wi-Fi 7 BE201 320MHz
```



## Software:
 - Ubuntu 24.04: Upgrade the kernel to 6.16.
 - Hardware driver: Core96 from Intel
 - wpa_supplicant: Compile [version 2.11](https://www.linuxfromscratch.org/blfs/view/svn/basicnet/wpa_supplicant.html) from source to enable 802.11be.
 - Disable the default network manager.


## Step 1: Upgrade the kernel

1. Install the mainline tool (This tool is not officially supported for using in production)
```bash
sudo apt install -y wget gpg
wget -qO - https://keyserver.ubuntu.com/pks/lookup?op=get&search=0xF6B0FC61 | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/canonical-kernel.gpg >/dev/null
sudo add-apt-repository ppa:cappelikan/ppa
sudo apt update
sudo apt install mainline
```
2. check the avaiable builds and update
```bash
mainline --list | grep 6.16
sudo mainline --install 6.16  # CHECK THE VERSION!!!

# Reboot and check the kernel
uname -r
```
If error occures in reboot, disable the Secure Boot for testing purpose

## Step 2: Install the firmware

The latest firmware from Intel, the [Core96](https://www.intel.com/content/www/us/en/download/824804/intel-wireless-wi-fi-drivers-for-linux.html) is contained in the [linux firmware](https://launchpad.net/ubuntu/+source/linux-firmware) release.

```bash
# Download the debian package and install
sudo dpkg -i linux-firmware_20250807.gitb6b0b152-0ubuntu1_all.deb
```

Check the firmware version:
```bash
sudo dmesg | grep iwlwifi
# Expected output: iwlwifi 0000:...: loaded firmware version 96.<...> for Intel(R) Wi-Fi 7 BE202
# Check the BE20x-specific .ucode:
cd /lib/firmware/
# you will find the firmware bloc like: iwlwifi-gl-c0-fm-c0-96.ucode (often compressed as .ucode.zst)
```

Moreover, the firmare alone isn't enough, Linux kernel/mac80211 and iwlwifi driver are essential:
 - expose MLO to cfg80211
 - pass them through to wpa_supplicant.

## Step 3: Compile the wpa_supplicant

It needs to be built with CONFIG_IEEE80211BE=y (v2.11 or newer): [Reference](https://www.linuxfromscratch.org/blfs/view/svn/basicnet/wpa_supplicant.html)

Install the dependencies:
```bash
sudo apt install -y libnl-3-dev libnl-genl-3-dev libssl-dev pkg-config \
                    libdbus-1-dev

# Optional: Rebuild with P2P enabled
sudo apt install -y libnl-3-dev libnl-genl-3-dev libssl-dev libdbus-1-dev pkg-config


# Download: https://w1.fi/releases/wpa_supplicant-2.11.tar.gz
# add a config file to overwrite the default configs
cd wpa_supplicant/
touch .config

# Build
make -j"$(nproc)"
# Move the local bin
sudo cp wpa_cli wpa_supplicant wpa_passphrase /usr/local/bin
```

Check the version:
```bash
wpa_supplicant -v
```

Important config:
 - nl80211 backend enabled (required for modern Linux Wi-Fi)
 - Wi-Fi 7 + MLO (CONFIG_IEEE80211BE)
 - WPA3/SAE with PMF
 - Control socket support so ctrl_interface=/run/wpa_supplicant works
 - DBus support so NetworkManager can talk to it later

## Step 4: Connecting

Add the wifi configuration in /etc/wpa_supplicant/
```bash
ctrl_interface=/run/wpa_supplicant
update_config=1
ap_scan=1
# p2p_disabled=1            # keeps P2P code out of the way
# MLO is ON by default; only add disable_mlo=1 if you ever need to turn it off
# disable_mlo=1

network={
    ssid="WLAN_NAME"
    scan_ssid=1
    key_mgmt=SAE           # WPA3-Personal
    ieee80211w=2           # PMF required
    psk="PASSWORD"
}
```

Connecting:
```bash
# deactivate the network manager
sudo systemctl stop NetworkManager
# set link up
sudo ip link set wlp87s0f0  up
# wpa
sudo /usr/local/bin/wpa_supplicant -i wlp87s0f0 -c /etc/wpa_supplicant/wifi7.conf
```

After successful connection, check the link
```bash
sudo iw dev wlp87s0f0 link
sudo iw dev wlp87s0f0 info
iw dev wlp87s0f0 station dump | grep -E 'tx|rx|bitrate'
```

Expected output with MLD:
```bash
Connected to 1c:0b:8b:00:43:01 (on wlp87s0f0)
	SSID: W7-MLO
	Link 1 BSSID 26:0b:8b:00:43:04
		freq: 5220.0
	Link 2 BSSID 2a:0b:8b:00:43:05
		freq: 6135.0
	Link 0 BSSID 26:0b:8b:00:43:03
		freq: 2437.0
MLD 1c:0b:8b:00:43:01 stats:
	RX: 93393057 bytes (99057 packets)
	TX: 15695713 bytes (46006 packets)
	signal: -46 dBm
	tx bitrate: 576.4 MBit/s 160MHz EHT-MCS 3 EHT-NSS 2 EHT-GI 0
	bss flags: 
	dtim period: 0
	beacon int: 0
```

Run DHCP client to get the IP
```bash
sudo dhcpcd wlp87s0f0
```
