# Cybersecurity Home Lab Creation
Author: Cody Keppen

Date: 11/02/2025

## Project Goal
To create a home lab to experiment and learn different tools for cybersecurity. I had my old computer sitting around that I decided to use for this project. This is an in progress project that I started on 11/02/2025.

Main skillsets to build on:
* Linux
* Creating VM's
* SIEMs
* Sandboxing
* Attacking vulnerable systems

## Host Machine Setup
* Hardware:
  *  H110I Pro (MS-7995)
  *  i5-6500
  *  Radeon RX 470
  *  16GB Ram
  *  8TB HHD
  *  2TB SSD
  *  TP-Link Archer TX2OU NANO
* Host OS: Ubuntu 24.04 LTS

## Actions setting up Home Lab

### Installing Linux
* Downloaded Ubuntu 24.04 LTS ISO on different device.
* Created a bootable USB drive using Balena Etcher.
* Installed Ubuntu on home lab, wiping the disk for a clean install.

### Wi-Fi Issue
At installation, no internet device was found when prompted to setup internet. There was at a step to use internet connection to download updates. I was using a Wi-Fi adapter and realized that it was not working with Linux.
* Issue to resolve: TP-Link Wi-Fi adapter wasn't recognizeable
  * Consulted AI as to why new adapter wasn't working.
  * Issue discovered: TP-Link Wi-Fi Realtek drivers not included in Linux kernal.
  * Solution: Tether mobile phone to get internet, then download RealTek driver.
  * Steps taken:
    *  Used `lsusb` to list USB devices and find the chip used for my Wi-Fi adapter.
    *  Once I discovered the ID number for the device, I used AI to discover the chipset.
    *  Ran `sudo apt update && sudo apt install git dkms build-essential linux-headers-$(uname -r)`, to prepare for driver installation.
    *  Ran `wget https://github.com/morrownr/rtl8852bu-20240418/archive/refs/heads/main.zip` to download the driver.
    *  Extracted the zip file with, `unzip main.zip`.
    *  After switching to the downloadeded file `cd rtl8852bu-20240418-main`, ran the installation script, `sudo ./install-driver.sh`.
    *  Ran into an issue in which I need to install the wireless utility iw, `sudo apt install iw`.
    *  Then ran installer again, `sudo ./install-driver.sh`, this time successfully.
    *  Rebooted computer and had the ability to connect to my router.
*  Issue resolved: Internet connection established.

### Finish Full System Update and Enable Firewall

With internet established without tethering my phone, the next step is to finish the system update that didn't occur at installation. Ran `sudo apt update && sudo apt upgrade`. Then I enabled the firewall with `sudo ufw enable`.
  
### Downloading the pre-built Kali VM (Attacker)

I go to the www.kali.org website and find the `Download` button.  Looking for the option for the virtual machines to download the VirtualBox pre-built images.

Once the zip file is downloaded, unzip the file. You will see a `.vbox` file. This is what we will upload into your VirtualBox.

Open you VirtualBox and click the `Add` button or go to Machine  > Add.  Select the kali-linux file until you find the `.vbox`. Select this file to add the Kali VM to your VirtualBox.

### Downloading Metasploitable2 (Victim)

Go to the https://sourceforge.net/projects/metasploitable/ website and click the `download` button to download  file for the VM. 

This download is a little different than with Kali. For Metasploitable 2, you need to click the `New` button. Give the VM he name Metasploitable 2 and change the type to Linux with the version as Oracle Linux (64-bit)

At the Hardware section, set your base memory based on your needs. The minimum can be around 512 MB.  Use one processor. Click `Next`.

Select the radio button for `Use an Existing Virtual Hard Disk File` and click the file browser button to the side.

If you don't see the `Metasploitable.vmdk` click `Add` at the top and select the file where you downloaded it. Then choose this file.

Review the Summary page and click `Finish`.

### Network Configuration Isolation

This is the process to isolate the VM from your home network.

In the VirtualBox, go to Tools or Network at the top . In Host-only Networks, Click `Create`. Take note of the name of this network.

This needs to be done for each. Go to your VM's and open their settings. Find the Network section. Under the `Adapter 1` go to `Attached to:` and select `Host-Only Adapter.` Go to Name and find your newly created network. This is now the network that your VM's will use and communicate to each other on. 

### Setting the IPs
At this point you need to set the IPs of your machines to differentiate them.

This is done with the following: 
```Bash
sudo ifconfig eth0 192.168.56.101 netmask 255.255.255.0 up
```
I used the following:
Kali IP: `10.10.10.10`

Metasploitable2 IP: `10.10.10.30`

## Completion

This is a very simple quick guide to get a Kali and Metasploitable2 Cyberlab


