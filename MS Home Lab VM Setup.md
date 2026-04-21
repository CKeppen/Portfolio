# MS Home Lab VM Setup

Author: Cody Keppen [LinkedIn Profile](https://www.linkedin.com/in/cody-keppen-a09068355/)

Date: 02/04/2026

# Table of Contents
- [Preview of Exercise](#preview-of-exercise)
- [Concepts Demonstrated](#concepts-demonstrated)
- [Instructional Steps](#instructional-steps)
- [Verification Steps](#verification-steps)
- [Lessons Learned](#lessons-learned)
- [Summary](#summary)
- [Resources](#resources)

---

# Preview of Exercise

This is the creation of a Microsoft enterprise work environment using VirtualBox (VB) to manage multiple Virtual Machines (VM).

VB will host a Windows 2022 Server VM ("Server") acting as the Domain Controller (DC)  with Active Directory Domain Services (AD DS) and DNS. 

Along with a Windows Enterprise Client ("Client") VM running Windows 10 acting as an employee workstation. 

There will also be a Kali Linux VM acting as a threat actor for future cybersecurity exercises.

This will be the first step in creating a corporate Microsoft work environment with generated organization units and employees. Governed by Group Policies and other admin enforced security controls, to be documented separately.

##  Concepts Demonstrated

1. VirtualBox VM creation and configuration
2. Snapshot creation and implementation
3. Multi-home network configuration using the Server as DNS for VMs
4. Server AD DS configuration
5. Successful pings between Server, Client and Kali
6. Kali probe demonstration

---

# Instructional Steps

These are the steps taken during this demonstration. There will be links throughout this section to the [Lessons Learned](#lessons-learned) section to notate obstacles and solutions during the demonstration.

## Host Machine
- Ubuntu 24.04 LTS
- H110I Pro (MS-7995)
- i5-6500 - 4 cores
- Radeon RX 470 4 GB
- 16 GB RAM
- 8 TB HDD
- 2 TB SSD

## Virtual Machine Creation

The three main Virtual Machines to be used are a Windows 2022 Server, a Windows Enterprise Client and a Kali Linux VM managed on VirtualBox.

These are the `.iso` VM images used.

- Windows 2022 Server: [Server_Eval_x64FRE_en-us.iso](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022)
- Windows 10 Enterprise Client: [21h2_CLIENTENTERPRISEEVAL_x64FRE_en-us.iso](https://archive.org/details/19044.1288.211006-0501.21h-2-release-svc-refresh-cliententerpriseeval-oemret-fre-en-us)
- Kali Linux 2026.1: [kali-linux-2026.1-virtualbox-amd64.7z](https://www.kali.org/get-kali/#kali-virtual-machines)

The Windows and Kali images are setup in different ways. I only have the `.iso` files the two Windows VMs I'll setup. Kali provides all the needed files (`.vbox`, `.vdi`) which makes it a much quicker setup.

(See [Virtual Machine Files](#virtual-machine-files) below for my new knowledge on the files related to VM's).

## Lab Network Configuration

This is the network diagram concept to be created for this lab. The Internal Network in layer 3 will not be exposed to the internet. Instead, it will rely on Layer 2 being the DNS and Gateway to handle those requests, being the bridge to the internet for Layer 3 by communication to Layer 1 for it.

- Layer 1 (External): Host Ubuntu Home Lab
	- Home Router WAN to Internet
- Layer 2 (Bridge): Windows Server as the Virtual Gateway of the VMs
	- External Interface (NAT)
	- Internal Interface (LAN)
- Layer 3 (Internal): Windows Client as the Private Lab
	- Default Gateway: Windows Server
	- DNS: Windows Server

![Lab Network Diagram](images/Pasted%20image%2020260419184046.png)

## VM File Management

For the file management, I decide on the below file structure. That way if I need to do a clean install of the `.iso` I have them on hand. Plus I keep the  `.vbox` and `.vdi` files together with the VM with their snapshots. (Learning the `tree` command was great here to help with the below.)

``` tree --charset=ascii

└── Lab Storage
    ├── ISO_Images
        ├── Kali
           └── kali-linux-2025.4-virtualbox-amd64.7z
        └── Windows
            ├── Windows Client
                └── 21h2_CLIENTENTERPRISEEVAL_x64FRE_en-us.iso
            └── Windows Server 2022
                └── SERVER_EVAL_x64FRE_en-us.iso
    └── VM files
        ├── Kali
           ├──Snapshots
           ├── kali-linux-2025.4-virtualbox-amd64.vbox
           └── kali-linux-2025.4-virtualbox-amd64.vdi
        └── Windows
            ├── Windows-10-Client
                 ├──Snapshots
                 ├── Windows-10-Client.vbox
                 └── Windows-10-Client.vdi
            └── Windows-Server-DC
                ├──Snapshots
                ├── Windows-Server-DC.vbox
                └── Windows-Server-DC.vdi            


```

(I did discover an issue in which I accidently installed the OS on the HDD and not the SDD, discovered here, [HDD/SSD Mounting Discovery](#hddssd-mounting-discovery))

## Windows VM Creation

In VirtualBox, click the "New" icon at the top. This will bring up a VM wizard to help with the installation of the two Windows machines.

![VirtualBox New VM button](images/Pasted%20image%2020260411165745.png)

In the wizard, a name is given and the VM folder is selected for the `.vbox` and `.vdi` files.

I leave the ISO Image blank for now to keep things simple and not have VirtualBox do too much setup here to avoid any conflicts. OS and OS Version are set to the future `.iso` file image I connect later.

![VM Wizard - Name and OS](images/Pasted%20image%2020260411170242.png)

The next page creates the `.vbox` settings to allocate resources to the VM. As an example for the server, 4 GB of memory and 2 CPUs are allocated with the default of 50 GB of Disk space. EFI is selected for the UEFI secure boot.

![VM Wizard - Hardware](images/Pasted%20image%2020260411171655.png)

Afterwards, a summary page for review to finish the creation of the `.vdi` and `.vbox` files.

![VM Wizard - Summary](images/Pasted%20image%2020260411171733.png)

I did this for both the Server and Client, shown below.

![Server and Client VMs created](images/Pasted%20image%2020260411172143.png)

Windows-Server-DC: 
- Windows Server 2022 64-bit
- 4 GB RAM
- 2 Cores
- 50 GB Storage

Windows-10-Client:
- Windows 10 64-bit
- 4 GB RAM
- 1 Core
- 50 GB Storage

## Network Adapter Configuration

Now that I have these two machines created, I'm going to setup the network before I connect the `.iso` file to the machine.

Going back to the [Lab Network Configuration](#lab-network-configuration) concept, I'm going to create these settings for the Windows Server in the Network settings.

One way you can get to the settings by right clicking the machine, then settings. Then scrolling down to "Network" to select and see the adapters. 

![Network Settings menu](images/Pasted%20image%2020260411174126.png)

Below are the settings for the Windows Server. Adapter 1 being the NAT network to communicate outside the VirtualBox network to the internet, while not allowing internet traffic to originate into the Lab Network.

Server Adapter 1:
![Server Adapter 1 - NAT](images/Pasted%20image%2020260411173728.png)

Adapter 2 is an Internal Network setting. This allows the Server to communicate with any other machines on the Lab Network.

Server Adapter 2: 
![Server Adapter 2 - Internal Network](images/Pasted%20image%2020260411173843.png)

The client only gets the Internal Network Lab Network setting. Meaning it can only see other machines on this network. It cannot see outside the VirtualBox. Relying on the Server to communicate outside the Lab Network, as the Gateway for the network. This creates the private Lab Network.

Client Adapter 1: 
![Client Adapter 1 - Internal Network](images/Pasted%20image%2020260411174401.png)

## .ISO Connection

With the network adapter configuration now setup for these machines, I will connect the `.iso` images to the machine.

Another way to adjust the settings of a machine is to select the machine, then scroll to a setting you want to adjust in the "Details" tab on the right.

Here I need to go to the "Storage" settings and connect the `.iso` for the initial install. You can see the `.vdi` already connected acting as a hard drive with 50 GB.

![Storage settings with .vdi](images/Pasted%20image%2020260411173404.png)

From here, depending on the machine you're setting up, you'll click the "Empty" optical drive on the left. On the right side will be the same optical drive symbol to click. Here you'll locate the `.iso` file for this specific machine.  As in, the Server VM will use the Server `.iso` shown below. Think of this like inserting a CD into the disc drive of your computer.

![Connecting the .iso to the optical drive](images/Pasted%20image%2020260411175630.png)

This is done for both the Server and Client.

## Launching the VM's

Now is the moment of truth to launch these one at a time to let the initial boot installation take place. Which I won't go into detail to stay on topic for the scope of this demonstration. Normal installation setup is done for both the Server and Client.

One main difference is that once I get to the Server login, a complex password is required, which is entered. 

Both VM's launch. Success!

![Both Windows VMs running](images/Pasted%20image%2020260411180936.png)

## VirtualBox Guest Additions

There can be some lag when running a VM, so I'm going to download something to help with the graphics for better mouse movement and resizing. This is from the Virtual Box Guest Additions image, which I install on the Host computer with the below.

``` Bash
sudo apt update && sudo apt install virtualbox-guest-additions-iso -y
```

First I'm going to shutdown the VM by clicking the "x" on the top right. Then select "Send the shutdown signal", which is the same as selecting "Shutdown" on your PC. "Save the machine state" saves the VM in it's current running state. "Power off the machine" is like a hard shutdown when holding down the power button of the device.

![VM shutdown options](images/Pasted%20image%2020260411181557.png)

With the same concept to connect the initial install `.iso`, I'll now choose the VBox Guest Additions for installation once the machine is down. Again, like inserting a VBox Guest Additions CD into the disc drive to install.

![Connecting VBox Guest Additions .iso](images/Pasted%20image%2020260411181422.png)

When I'm back in the VM, I can see a disc drive for the VBox Guest Additions in the File Manager.

![VBox Guest Additions disc in File Manager](images/Pasted%20image%2020260411181706.png)

I choose the VBox WindowsAddition-amd64 application to run and install. Then reboot for a smoother experience.

![VBoxWindowsAdditions installer](images/Pasted%20image%2020260411181752.png)

## VirtualBox Snapshot

Now that I have the two Windows machines up and running and with VBox Guest Additions, I'm going to take my first baseline snapshot of these two machines.

With the VM selected, in the details window on the right is a second tab with a camera icon called "Snapshots". On the top left will be a "Take" button that once clicked will create a current snapshot of your VM. You'll give the snapshot a name, and a short description. After that, they'll be in a list format to choose from.

![Snapshot tab in VirtualBox](images/Pasted%20image%2020260411182906.png)

This is perfect for experimenting with your machines and should be taken advantage of when learning something new. Even in this documentation, I've gone back and forth to better help put down my actions in this writing when I felt my information dump of notes didn't help with the documentation. Or going through the network setup multiple times for practice or experiments of doing something better.

## Kali VM Creation

Luckily, Kali is incredibly easy to install. Going to Kali site for the [pre-built VM](https://www.kali.org/get-kali/#kali-virtual-machines), download the `.7z` zip file for VirtualBox.

![Kali pre-built VM download page](images/Pasted%20image%2020260411184752.png)

In the Host i5-6500 terminal, I run the following command to unzip file for install.

``` Bash
7z x kali-linux-2026.1-virtualbox-amd64.7z
```

This will provide the pre-built `.vdi` and `.vbox` files for Kali.

In VirtualBox, click the "Add" button and select the Kali `.vbox` that was just downloaded. 

![VirtualBox Add button](images/Pasted%20image%2020260411165745.png)

![Kali .vbox added to VirtualBox](images/Pasted%20image%2020260411192051.png)

Go to Network settings and choose Adapter 1 to be the Internal Network, Lab Network like the other VMs. Again, the concept is to isolate the lab from the internet using the Internal Network, while the Server handles requests outside the Lab. 

![Kali Adapter 1 - Internal Network](images/Pasted%20image%2020260411192111.png)

Run the Kali VM, enter the default login. Then you're in. Make a base Snapshot and this is now setup.

![Kali VM running](images/Pasted%20image%2020260411185459.png)

The VirtualBox now has a Windows Server, Windows Client and Kali system up and running.

![All three VMs in VirtualBox](images/Pasted%20image%2020260411185719.png)

Now that the machines are up and running, I need to get the network setup. Then the Domain Controller with Active Directory and DNS.

## Server Network and AD DS Setup

In the Windows Server, I need to discover the NAT and Internal Network cards. Going to the Network Connections menu, I see two network adapters, "Ethernet" and "Ethernet 2".

![Network Connections - two adapters](images/Pasted%20image%2020260416171511.png)

To discover which is the NAT and Internal Network, I check the status of these adapters.

Left side is Ethernet and right side is Ethernet 2. 

![Ethernet adapter status comparison](images/Pasted%20image%2020260416172849.png)

Ethernet 2 has a `10.0.2.15` with a gateway, DHCP and DNS server, while Ethernet has a `169.254.x.x`, which is an APIPA (Automatic Private IP Addressing) IP. Meaning the device couldn't find a DHCP server to give it an IP. Ethernet 2 got it's IP from the VirtualBox DHCP, meaning it has outside connection, which points to this being the NAT.

Now that I can identify which adapter is which, I rename them to better setup the network.

"Ethernet 2" > "Internet" (NAT)
"Ethernet" > "Internal" (Internal Network: Lab Network)

![Renamed network adapters](images/Pasted%20image%2020260416172750.png)

Going to the Internal card properties, I'm going to give it the static IP of `192.168.10.1`, subnet mask `255.255.255.0` and a preferred DNS of `127.0.0.1` to resolve DNS itself. Default Gateway is left blank, since this is for the Internal network.

![Internal adapter static IP configuration](images/Pasted%20image%2020260416173238.png)

Verification of the Internal card with the new IP and subnet, `192.168.10.0/24` for the Lab Network.

![Internal adapter IP verification](images/Pasted%20image%2020260416173608.png)

(I did end up mixing the Internal Network and NAT cards with the names - Internet and Internal. I knew which was which, but the names are so similar and I typed the wrong one for each. I caught it, but I somehow created a second Internal card during this process. Which leads to an issue later that I have to solve. Mentioned and documented the section [Remote Access Setup](#remote-access-setup).

With the Internal Network setup, I'll use Server Manager to setup the Domain Controller. In Server Manager, I'll select "Add Roles and Features" to begin the process.

![Server Manager - Add Roles and Features](images/Pasted%20image%2020260416173932.png)

In the Wizard menu, I'll leave the Installation Type on "Role-based or feature-based installation".

At server selection, I can confirm the IP Address of the Windows Server and OS.

![Server selection confirmation](images/Pasted%20image%2020260416174207.png)

In the next screen is the server role selection. This is where Active Directory Domain Services is selected. (AD DS). Clicking Add Features to move forward with the decision.

![AD DS role selection](images/Pasted%20image%2020260416174347.png)

Going through the next screens to confirm the selection, then hitting "Install".

![AD DS installation confirmation](images/Pasted%20image%2020260416174518.png)

After the installation, the Server Manager Dashboard has a yellow warning label to ensure configuration of the AD DS.

![Server Manager yellow warning for AD DS config](images/Pasted%20image%2020260416174946.png)

In the Deployment Configuration menu, I select "Add a new forest" and choose the Root domain name, `cyberlab.local`.

![AD DS Deployment Configuration - new forest](images/Pasted%20image%2020260416181338.png)

I create a Directory Services Restore Mode (DSRM) password when prompted.

I move through the rest of the configuration wizard menu using the defaults, then installing. Once finished, the machine will reboot.

![AD DS installation progress](images/Pasted%20image%2020260416181944.png)

When the server reboots, this is now the login screen with the CYBERLAB domain. Success!

![CYBERLAB domain login screen](images/Pasted%20image%2020260416182323.png)

Now Server Manager has additional roles. AD DS and DNS. Success!

![Server Manager with AD DS and DNS roles](images/Pasted%20image%2020260418152655.png)

## Client Network Setup

I'm going to go to the Client now to get it ready for a connection.

Going to the Ethernet adapter, I give the Client a static IP of `192.168.10.2` with the subnet `255.255.255.0` to be on the `192.168.10.0/24` Internal Lab Network.

Then I use the Server IP of `192.168.10.1` so that the client uses the Server as the DNS Server.

![Client static IP configuration](images/Pasted%20image%2020260416184830.png)

Now, for the final test. Does the Server and Client reflect this information? 

Client IP confirmed `192.168.10.2`

![Client IP confirmation](images/Pasted%20image%2020260416184945.png)

Server, confirmed NAT address of `10.0.2.15` and Internal address of `192.168.10.1`

![Server IP confirmation](images/Pasted%20image%2020260416185444.png)

## Domain Controller Setup

Back on the Server, I get to the "Active Directory Users and Computers" window through the Server Manager "Tools". I can see `cyberlab.local` domain! Success!

![Active Directory Users and Computers - cyberlab.local](images/Pasted%20image%2020260418172138.png)

Now, back on the Client I need to join it to the domain.

In the "About your PC" menu", I find the "Rename this PC (advance)" option. Select "Change..." for the option to change the domain.

In the "Computer Name/Domain Changes" window, I select "Domain" and enter the `cyberlab.local` I just created.

![Domain join dialog](images/Pasted%20image%2020260418172526.png)

An administrator login with popup for permissions. Using the admin account setup on the server, permission is granted.

![Admin credentials prompt for domain join](images/Pasted%20image%2020260418173025.png)

(See [Admin name](#admin-name) section for the hurdle of the nuances of the new admin name and domain join.)

Once entered correctly, there is a "Welcome" message, followed by a restart prompt that must occur, which I then restart the Client VM.

![Domain join welcome message](images/Pasted%20image%2020260418174125.png)
![Restart prompt](images/Pasted%20image%2020260418174153.png)

Now at the new login screen of the Client, I'm going to test logging in as the Domain Admin by clicking "Other Users" at the bottom. Then logging in as `CYBERLAB\Administrator`. Though seeing "CYBERLAB" is already a good indicator.

![Client login screen showing CYBERLAB domain](images/Pasted%20image%2020260418174432.png)

Success!

![Logged in as CYBERLAB Administrator](images/Pasted%20image%2020260418174512.png)

Once the Client is loaded up while logged in as the CYBERLAB admin, it's time to check the Computers in the Active Directory on the Server for further verification.

On the Server in "Active Directory Users and Computers", Computer `DESKTOP-MBV8D69` is shown.

![Active Directory Computers showing client](images/Pasted%20image%2020260418175101.png)

Going back to the Client "About your PC" page, it is confirmed! Even the "Full Device Name" shows the domain, `DESKTOP-MBV8D69.cyberlab.local`. Nice!

![Client About PC showing domain](images/Pasted%20image%2020260418175133.png)

## Creating an IT Admin User

Now that the Client is on the domain, I'm going to make an IT admin user for my regular use, so I'm not always logged in as the Domain Administrator. This is a security practice in case that user is compromised, it does not have Admin roles or permissions.

In the "Active Directory Users and Computers" window, I go to Users > New > User.

![New User option in AD](images/Pasted%20image%2020260418180435.png)

I'm going to make an admin named Cody Admin. Using the middle initials as a way for me to easily identify their department, as I make more users.

![New user - Cody Admin](images/Pasted%20image%2020260418181338.png)

Next window is to create a password. Since this will be my main account, I'm going to uncheck "User must change password at next logon", but will keep it for future Users.

![Password options for new user](images/Pasted%20image%2020260418181542.png)

Then hit finish after reviewing for any typos.

![New user summary](images/Pasted%20image%2020260418181605.png)

I did however have second thoughts about the "Password Never Expires" since this is a lab and not real environment, and added that.

![Password Never Expires setting](images/Pasted%20image%2020260418181759.png)

Back to the Client, I log out of the Domain Administrator, and now the new Admin user. 

Success!

![Logged in as cody_admin](images/Pasted%20image%2020260418181923.png)

## Remote Access Setup

Now I need to set the Server up to allow the Client to access the internet securely using the Server for routing. In the "Server Manager" window, I go to Manage > Add Roles and Features.

![Server Manager - Add Roles and Features](images/Pasted%20image%2020260418182415.png)

I'll go through the Wizard till I get to Server Roles and find Remote Access.

![Remote Access role selection](images/Pasted%20image%2020260418182557.png)

In "Role Services", I check "Routing" and "Add Features", which will also toggle on "DirectAccess and VPN(RAS)"

![Routing and RAS role services](images/Pasted%20image%2020260418182720.png)

At the end of the Wizard, I review, then Install.

![Remote Access installation confirmation](images/Pasted%20image%2020260418182844.png)

Once that is finished, on the Server still I'll go to Tools then Routing and Remote Access.

![Tools - Routing and Remote Access](images/Pasted%20image%2020260418183226.png)

In the "Routing and Remote Access" window, I see the Server and a red down error. I'll right click and select "Configure and Enable Routing and Remote Access" to bring up the Remote Access Server Setup Wizard.

![Routing and Remote Access - configure option](images/Pasted%20image%2020260418183401.png)

I'm going to select NAT, then find the Internet adapter with the IP address of `10.0.2.15`, that was confirmed earlier. Then select "Finish". (I did have to exit and go back in for a refresh to see these.)

![NAT configuration - selecting Internet adapter](images/Pasted%20image%2020260418184027.png)

(I did end up running into some trouble getting a connection, as I ended up with two internal networks in my NAT. I'm guessing this occurred earlier when I mixed the Ethernet adapters. I go over that here [Google DNS Ping issue](#google-dns-ping-issue).)

Now, going back to the Client while as `cody_admin` I'm going to change the Default Gateway of the Internal adapter to the Server, `192.168.10.1`.  So that any requests to the internet, go to the Server to be passed on with the NAT adapter. When the request is answered, the Server will route to the device that made the original request using the NAT Table records.

This change requires permission by using the Domain Administrator login credentials.

![Client Default Gateway set to Server IP](images/Pasted%20image%2020260418184922.png)

With the network setup for the VM's, the objectives for the demonstration are completed and need to be verified.

---

# Verification Steps

## VirtualBox VM Creation and configuration.

Success.

Three functioning VM's, a Windows Server, Windows 10 Client and Kali Linux

![All three VMs running in VirtualBox](images/Pasted%20image%2020260418191036.png)

## Snapshot creation and Implementation

Success.

A base and network snapshot created for easy rollback.

![Snapshots list for VMs](images/Pasted%20image%2020260418191146.png)

## Multi-home Network Configuration Using the Server as DNS for VMs.

Success.

Client using the Server as a Gateway has successful pings to Google DNS.

![Client pinging Google DNS successfully](images/Pasted%20image%2020260418191805.png)

## Server AD DS Configuration

Success

Server has AD DS and DNS up and running, with the `cyberlab.local` domain.

![AD DS running on Server](images/Pasted%20image%2020260418201557.png)

![DNS running on Server](images/Pasted%20image%2020260418201730.png)

## Successful pings between Server, Client and Kali

Success.

Server Ping to Client Test: Success

First test is against the client. Which failed. I then turned off the domain firewall, and got successful pings. I go over that here [Firewall ICMP Block](#firewall-icmp-block).

![Server ping to Client - success after firewall change](images/Pasted%20image%2020260418194901.png)

Server Ping to Kali Test: Success

![Server ping to Kali](images/Pasted%20image%2020260418201347.png)

Client to Server Ping Test: Success

![Client ping to Server](images/Pasted%20image%2020260418201258.png)

Client to Kali Ping Test: Success

![Client ping to Kali](images/Pasted%20image%2020260418201331.png)

Before running the Kali tests, I setup the Kali IP, `192.168.10.3`, as it's the first time getting on the network. Which is a quick and easy process.

![Kali IP configuration](images/Pasted%20image%2020260418200126.png)

Kali Ping to Server Test: Success

![Kali ping to Server](images/Pasted%20image%2020260418200202.png)

## Kali Probe Demonstration

Kali Ping to Client Test: Windows Firewall block Success ([Firewall ICMP Block](#firewall-icmp-block))

![Kali ping to Client - blocked by firewall](images/Pasted%20image%2020260418200249.png)

Kali Nmap scan to Client to Bypass Firewall Ping Test: Success

![Kali Nmap scan results on Client](images/Pasted%20image%2020260418200838.png)

Not only a success, but the first vulnerability discovered with the open RPC Port: 135! I also added Server as the DNS for Kali with for now.

``` bash
echo "nameserver 192.168.10.1" > /etc/resolv.conf
```

All items are a success!

---
# Lessons Learned

The below were linked throughout the demonstration.

## Virtual Machine Files

I had to dig deeper to understand the [Virtual Machine Creation](#virtual-machine-creation) files as there were multiple files for each VM and I wanted to be organized. Below is a high level view of what these three files do and how they relate to the VM's.

- `.iso` (Optical Disc Image) is the downloaded copy of the virtual machine for installation. This is the initial install of the VM for setup, like what used to be on installation CD's when getting a new OS.
- `.vbox` (VirtualBox Settings) is the resource settings from VirtualBox when you setup the VM in VirtualBox. Like how many cores, RAM allocation and network settings for the `.vdi` when you open it.
- `.vdi` (Virtual Disk Image) is the actual OS and storage of the VM itself and what gets launched from VirtualBox.

To setup a Virtual Machine, find the `.iso` for the system you want to run and download it. This is basically the "disc" for the VM. Then use the `.iso`  in VirtualBox to create a `.vbox` with the resource allocation and the `.vdi` that gets launched.

## HDD/SSD Mounting Discovery

 It was at this point when looking to install the VM's I realized I made a mistake when installing Linux for the first time on the Host device. Before 2026, I've never used Linux. I'm using my old gaming PC that had been laying around and it had an 8 TB HDD and a 2 TB SSD with an i5-6500 CPU.

Initially when installing the OS for this machine, I thought I had chosen the SSD for the OS as I wanted a faster boot. This is before even getting into the thought of using VM's.

A set back at first, but then I settled on the positive that my VM's will be on the faster SSD and the Host will primarily have all the storage.

So I pivoted to using the SSD solely for the VM files. Which meant I needed to learn how to mount a storage device on Linux. I've only done this on Windows.

In the Disk app on the Host Server, it is confirmed that the 2 TB SSD has no partitions and the 8 TB has a partition for the BIOS Boot and another for the file system.

To mount the SSD:
- Click on the box inside the SSD disk
- Look at the bottom for `+` sign and click it
- At the "Create Partition" menu, adjust your partition size and click "Next"
- Enter a name in the Volume Name field like, "Lab Storage"
- Select the file system "Ext4" (Linux format, unlike Windows NTFS)
- Click "Create"
- Confirm the 2 TB Disk and "Lab Storage" is now in the File Manager

![SSD mounted as Lab Storage in File Manager](images/Pasted%20image%2020260410231527.png)

Once confirmed, I got back to the [Virtual Machine Creation](#virtual-machine-creation) steps. This is something I will address later and document fixing. As I might be able to move drives around for better efficiency between my other devices. As that is a good SSD, that I could put into my T14, which only has 512 GB.

## Admin name

I had quite the trouble getting the admin name right. I kept going between Administrator and Admin with username errors. When I looked it up, it was the domain admin name that was created I needed to now use, "CYBERLAB\Administrator".

It's not the local admin of the computer that needed to give permission, but the admin of the actual new domain.

Back to [Domain Controller Setup](#domain-controller-setup).

## Google DNS Ping Issue

I was having issue after setting up the Default Gateway to actually get any pings with Google. 

![Google DNS ping failing](images/Pasted%20image%2020260418190529.png)

Eventually, I went into the Router and Remote Access window to find that I had two Internal Network cards. Which was causing confusion with the system. This probably was created when I realized I swapped the adapters. 

![Two Internal Network cards in Routing and Remote Access](images/Pasted%20image%2020260418190618.png)

Once I deleted the extra one, I was able to get the Google DNS pings and continue on with the [Remote Access Setup](#remote-access-setup).

## Firewall ICMP Block

When doing the ping tests, I came across the situation in which the Client could ping the Server, but the Server was not able to ping the Client. Which was odd to me, as both could ping Google DNS `8.8.8.8`. The below are the CMD of the Server on left and Client on right.

![Server-Client ping asymmetry](images/Pasted%20image%2020260418232603.png)

Both were confirmed to be on the same subnet and the Client pinging the Server further provided they should be able to ping each other.

I ran `traceroute` and found the subnet `192.168.10.0/24` does go out the internal card using `192.168.10.1`

![Traceroute output](images/Pasted%20image%2020260418233551.png)

I then went to check if it was truly the Firewall that was possibly blocking the ICMP echo requests to the Client. So I turned off the Domain Firewall, which prompted the Domain Admin credentials.

![Domain Firewall toggle - admin credentials prompt](images/Pasted%20image%2020260418233848.png)

Here is a side by side of the ping on the left from the Server to the Client, which on the right shows the Domain Firewall is off. Showing, a successful connection.

![Side-by-side ping success with firewall off](images/Pasted%20image%2020260418233935.png)

Once confirmed, I turned the Firewall back on. After some research, Windows Firewall does block ICMP echo requests.

After confirmation that all devices can communicate with each other, I proceed with the [Successful pings between Server, Client and Kali](#successful-pings-between-server-client-and-kali) test.

---
# Summary

This was quite the project for me. I've had some small iterations of this demonstration in smaller chunks as I learned about VirtualBox and networking. 

Originally I just started with a Kali and Metasploitable machine on a Ubuntu host. The setup was very basic, but I wanted a Windows Environment with some networking practice. Though that was my first taste of using Linux.

The networking setup and troubleshooting required a lot of research. This was the most involved networking project I've taken. I have been watching videos for eventually taking my Network+ and I would refer to Professor Messer, AI and Google for networking concepts.

I've been in a Windows Enterprise business for almost a decade, which migrated to an Azure hybrid setup in my later years there. Being a Manager and Director of areas, I feel a bit comfortable on the IT side of things for Windows, as I've been in it for so long. Even personally.

It was fun getting the network connections and seeing the communication between the devices. Logging in to different accounts. Especially exciting seeing the Kali Nmap scan already get a hit.

Though the next step would be to create a whole fake organization. Which will be the next documentation piece I put out.

---
# Resources
- VirtualBox - https://www.virtualbox.org/
- Windows 2022 Server: [Server_Eval_x64FRE_en-us.iso](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022)
- Windows 10 Enterprise Client: [21h2_CLIENTENTERPRISEEVAL_x64FRE_en-us.iso](https://archive.org/details/19044.1288.211006-0501.21h-2-release-svc-refresh-cliententerpriseeval-oemret-fre-en-us)
- Kali - https://www.kali.org/
	- Kali Linux: [kali-linux-2026.1-virtualbox-amd64.7z](https://www.kali.org/get-kali/#kali-virtual-machines)
- Guides/Help
	- https://bluecapesecurity.com/build-your-lab/medium-lab/
	- https://www.blackhillsinfosec.com/build-a-home-lab-equipment-tools-and-tips/
	- https://www.blackhillsinfosec.com/what-to-do-with-your-first-home-lab/
	- https://www.youtube.com/professormesser
