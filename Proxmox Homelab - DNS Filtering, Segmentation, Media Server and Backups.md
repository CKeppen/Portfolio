# Proxmox Homelab - DNS Filtering, Segmentation, Media Server and Backups
By: Cody Keppen [LinkedIn Profile](https://www.linkedin.com/in/cody-keppen-a09068355/)
Date: 09/16/2026

---
# Table of Contents
- [Preview of Exercise](#preview-of-exercise)
- [Concepts Demonstrated](#concepts-demonstrated)
- [Host and Lab Information](#host-and-lab-information)
- [Instructional Steps](#instructional-steps)
- [Verification Steps](#verification-steps)
- [Lessons Learned](#lessons-learned)
- [Summary](#summary)
- [Resources](#resources)

---

# Preview of Exercise

This is my personal homelab running on Proxmox VE. It started as a way to get household-wide ad blocking and turned into a place to practice firewall, networking and backup skills on real hardware.

The main part of this project is using a pfSense firewall VM to move Proxmox Backup Server (PBS) onto its own network segment. Only the Proxmox host can reach it. Household devices cannot.

The server also runs AdGuard Home for household DNS filtering and Jellyfin as a media server for the TVs and phones in the house.

Each part was built in stages. I didn't move on to the next stage until the current one was tested and working.

After the build was done, I ran a perimeter audit to check if anything on the network was reachable from the internet. That audit is its own write-up, [Home Network Perimeter Audit](Home%20Network%20Perimeter%20Audit.md), and a short version is included here.

IP addresses, hostnames and device identifiers are left out of this write-up on purpose.

#  Concepts Demonstrated

1. [Proxmox VE Host Setup](#proxmox-host-setup)
2. [DNS Filtering with AdGuard Home](#adguard-home-dns-filtering)
3. [pfSense Firewall on a Single NIC](#pfsense-firewall)
4. [Network Segmentation of PBS](#moving-pbs-behind-pfsense)
	1. Least privilege firewall rule for the Proxmox host only
	2. Static routing to the firewalled segment
	3. End-to-end backup test
5. [Jellyfin Media Server](#jellyfin-media-server)
	1. Unprivileged LXC container
	2. Read-only media mount
	3. Playback testing before GPU passthrough
6. [Perimeter Audit](#perimeter-audit)

---
# Host and Lab Information

- Host Hardware:
	- H110I Pro (MS-7995), Mini-ITX
	- i5-6500 (4 cores, 4 threads)
	- 16 GB RAM
	- 512 GB SSD for Proxmox and VM/container disks
	- 8 TB HDD for data
	- 2 TB HDD for the PBS datastore
	- Single onboard NIC
- Hypervisor: Proxmox VE (Debian 13 based)
- Upstream network: consumer mesh router from the ISP

| Service | Type | Placement |
|---|---|---|
| AdGuard Home | LXC | Home LAN |
| Jellyfin | LXC | Home LAN |
| pfSense CE | VM | Edge of the backup segment |
| Proxmox Backup Server | VM | Behind pfSense |

<br>

![](images/Pasted%20image%2020260916214316.png)

<br>

## Network Concept

This is what the network looks like at a high level.

- Home LAN (ISP router)
	- Proxmox host management
	- AdGuard Home, household DNS
	- Jellyfin
	- Household devices
	- pfSense WAN interface
- Backup segment (behind pfSense)
	- pfSense LAN interface, acting as the gateway
	- Proxmox Backup Server

The general flow is as follows:

- Household devices use AdGuard for DNS, set once on the ISP router
- The Proxmox host has a static route to the backup segment through the pfSense WAN interface
- pfSense has a rule allowing only the Proxmox host to reach PBS
- Everything else trying to reach the backup segment gets blocked by default

---
# Instructional Steps

These are the steps taken during this build. There will be links throughout this section to the [Lessons Learned](#lessons-learned) section to notate obstacles and decisions made along the way.

## Proxmox Host Setup

I installed Proxmox VE and switched the package repository to the no-subscription repo, so the host could get updates without a paid subscription. Then I updated the host fully before building anything on it.

The host management IP was set as a static address outside of the router's DHCP pool, so it wouldn't conflict with household devices.

<br>


![](images/Pasted%20image%2020260917103112.png)

<br>

## AdGuard Home DNS Filtering

AdGuard was the first service and the first real win for the household.

I put AdGuard on the home LAN and not behind pfSense. If the pfSense VM ever went down, I didn't want everyone in the house to lose internet because DNS stopped working.

It is also the simpler and more efficient setup. Every device in the house sends DNS queries all day, so keeping AdGuard on the same network avoids sending each one through the firewall. Putting it behind pfSense would have also meant opening a firewall rule for DNS and giving household devices a route to the segment, which my ISP router can't do.

1. Deployed AdGuard Home in an unprivileged LXC container with a static IP
2. Set the upstream resolvers to use encrypted DNS-over-HTTPS (DoH)
3. Switched the Quad9 upstream to its malware filtering endpoint, which also validates DNSSEC
4. Pointed the ISP router's DNS setting to AdGuard, so every device in the house is covered without setting each one up
5. Added the default ad and tracker lists, plus a malware and threat blocklist
6. Verified ads were being blocked on a household device

<br>

![](images/Pasted%20image%2020260916214552.png)

<br>

Really be careful going overboard with different preset DNS Filters. I go over some of the fun trial and errors I went through here, [DNS Filtering Lessons](#dns-filtering-lessons).
## pfSense Firewall

My server only has one NIC and the ISP router doesn't support VLANs. So pfSense is set up as a "router-on-a-stick" using two Proxmox bridges. I go over why I dropped the VLAN plan in [Single NIC and No VLANs](#single-nic-and-no-vlans).

| Bridge | Purpose |
|---|---|
| Physical NIC bridge | pfSense WAN, facing the home LAN |
| Internal-only bridge (no physical port, no host IP) | pfSense LAN, the isolated segment |

1. Created the internal-only bridge on the Proxmox host
2. Created the pfSense VM with one virtual NIC on each bridge
3. Assigned the WAN and LAN interfaces in pfSense
4. Verified pfSense could route out to the internet before putting anything behind it

<br>

![](images/Pasted%20image%2020260916221157.png)

<br>

## Moving PBS Behind pfSense

PBS was the first service I moved behind the firewall. It holds the backups for every VM and container, so it is worth protecting. It was also low risk to experiment with, since nothing in the house depends on it day to day.

<br>

![](images/Pasted%20image%2020260916221123.png)

<br>

The order here mattered a lot. More on that in [Order of Operations](#order-of-operations).

1. Wrote the pfSense firewall rule allowing only the Proxmox host to reach PBS
2. Added a persistent static route on the Proxmox host to reach the backup segment through pfSense
3. Changed the PBS network settings (IP, gateway, DNS) inside the VM first
4. Moved the PBS virtual NIC to the internal bridge and restarted networking
5. Ran a full backup job from the Proxmox host to PBS

The backup completed successfully through the firewall. Success!

<br>

![](images/Pasted%20image%2020260916221509.png)

<br>

## Jellyfin Media Server

Jellyfin stays on the home LAN. Every TV, phone and tablet in the house needs to reach it. Putting it behind pfSense would mean a permanent hole in the firewall, which would go against the reason for having the segment.

1. Created an unprivileged Debian 13 LXC container and installed Jellyfin directly, no Docker
2. Bind mounted the media library into the container as read-only
3. Kept Jellyfin's metadata and artwork in its own config folder on the SSD
4. Set ownership of the media folder on the host to match the container's unprivileged user mapping, so Jellyfin can read the files without extra rights. This took some work, which I go over in [UID and GID Mapping](#uid-and-gid-mapping)
5. Added a test movie to the library to confirm Direct Play worked
6. Tested playback on the phone app, a streaming stick and a web browser

The read-only mount was a security decision. Smart TVs and streaming devices run firmware that rarely gets updated. If Jellyfin was ever compromised through one of them, it still couldn't delete or encrypt the media library.

I originally had GPU passthrough planned as its own phase for hardware transcoding. After testing playback, I retired that phase. I go over why in [Testing Before GPU Passthrough](#testing-before-gpu-passthrough).

<br>

![](images/Pasted%20image%2020260916221645.png)

<br>

## Perimeter Audit

Once everything was built, I wanted to confirm nothing was exposed to the internet and that the household controls were actually working. The full details are in the [Home Network Perimeter Audit](Home%20Network%20Perimeter%20Audit.md).

- Scanned ports on my public IP with `nmap` from a cellular connection. All came back `filtered`
- Checked Shodan for any recorded services on my public IP. None found
- Tested the public IP as an open DNS resolver. No response
- Reviewed the router's port forwarding table. Empty
- Checked routes from my laptop to the backup segment. None

The audit also found one issue. A VPN client on my laptop was silently skipping the household DNS filtering. That is covered in [DNS Filtering Bypassed by a VPN](#dns-filtering-bypassed-by-a-vpn).

---
# Verification Steps

## pfSense Routing

pfSense routed to the internet before PBS was moved behind it. Ping to `9.9.9.9` and `google.com` were successful.

<br>

![](images/Pasted%20image%2020260917104915.png)

![](images/Pasted%20image%2020260917104932.png)

![](images/Pasted%20image%2020260917104536.png)

<br>

## Host to PBS

The Proxmox host reached PBS through the firewall rule, and a full backup job completed.

<br>

![](images/Pasted%20image%2020260916221509.png)

<br>

## Household Devices to PBS

Only the Proxmox host has a route and an allow rule to the backup segment. The perimeter audit confirmed my laptop on the home LAN has no route to it.

## pfSense Web GUI

The pfSense web GUI is not reachable from the home LAN. This is intended. Management is done from inside the segment or through the Proxmox console.

## DNS Filtering

Ads and known malicious domains are blocked on household devices, and queries show in the AdGuard query log.

<br>

![](images/Pasted%20image%2020260917111149.png)

<br>

## Jellyfin Read-Only Mount

Trying to create a file on the media mount from inside the container returned `Read-only file system`.

```bash
pct exec 103 -- touch /data/media/writetest
```

<br>

![](images/Pasted%20image%2020260917111722.png)

<br>

## Jellyfin Playback

The test movie confirmed Direct Play was working. The phone app played with true Direct Play, with no transcode process running on the server. The browser used Direct Stream, where the video passed through untouched and only the audio was converted.

<br>

![](images/Pasted%20image%2020260916222644.png)

<br>

## Internet Exposure

No services reachable from the internet. All scanned ports filtered.

<br>

![](images/Pasted%20image%2020260917122548.png)

<br>

---
# Lessons Learned

## DNS Filtering Lessons

When you get a new toy, it can be hard to not just turn everything on. Which in this case, taught me some early DNS filtering lessons.

First was enabling so many lists that I froze the AdGuard service as it ran out of RAM. Eventually I had to bump all the remaining RAM into the service to let it finish adding the rules. Which, AdGuard does provide in the table. A note to always check this. I settled on these two.

<br>

![](images/Pasted%20image%2020260917113913.png)

<br>

Another issue is blocking too much. Everyone hates adds on YouTube, so I thought I'd try blocking it. I found a list that advertised as a YouTube blocklist. What I didn't realize is that it does just block the ads, it blocks all of YouTube. Which resulted in a verbal ticket from my girlfriend. High priority, Critical, All hands on deck.

There might have been a second ticket later, as a virtual meeting had technical difficulties from another DNS filter.

Best advice. Take it slow. Start small.

## Single NIC and No VLANs

I originally planned a four-tier VLAN setup. I dropped it because real VLANs need a managed switch or a router that supports VLANs, and my ISP router doesn't.

Having pfSense firewall one segment was the right size for this hardware. It also keeps household internet from depending on a VM running on a 16 GB server.

I thought this would be a good way to still get segmentation while learning pfSense, which I've seen a lot in the self-hosting communities.

## Order of Operations

If I had moved PBS before pfSense was routing and the firewall rule was written, the Proxmox host would have lost contact with its backup server. The order that worked was build pfSense, verify it, write the rule, move PBS, then test with a backup.

Changing the IP inside the PBS VM first, then moving it to the new bridge, kept the switch to one step. I used the same order again for later changes.

## Remote Access Without a Subnet Router

I use Tailscale for remote access with no ports forwarded. It is only installed on the Proxmox host. None of the services have it, and I'm not using a subnet router. If I add remote access to anything else later, the plan is to install Tailscale on that machine directly.

A subnet router advertising the backup segment would create a path around pfSense. Installing it on a machine directly, instead of advertising a whole network, keeps pfSense in control of what reaches the segment.

## pfSense GUI Lockout

After I deleted the temporary allow rule I used during setup, I couldn't reach the pfSense web GUI from the home LAN anymore. That is the correct end state. I documented how to get to it from inside the segment or the Proxmox console instead of leaving the rule in place.

## Storage Without a NAS VM

I looked at TrueNAS and OpenMediaVault for storage. Both want 8 to 16 GB of RAM on their own, which doesn't work on a 16 GB host. Bind mounts with permissions set per service work fine for a single server.

## Backup Gap

PBS backs up the VMs and containers, but not folders on the host itself. Personal data on the 8 TB drive is on a single disk with no second copy right now. That is the next project.

## UID and GID Mapping

Getting file ownership right between the host and the container was a big lesson for me.

An unprivileged container doesn't use the same user IDs (UID) and group IDs (GID) as the host. Proxmox shifts them by 100000. So UID `1000` inside the container is UID `101000` on the host.

That means the media folder on the host has to be owned by `101000:101000` for it to show up as `1000:1000` inside the container.

```
chown -R 101000:101000 <media folder>
```

Anything I copied into the folder as root on the host showed up as `0:0` on the host, and as `nobody` (`65534`) inside the container. Those files needed the same `chown` before Jellyfin could see them properly.

To check from inside the container, `ls -lan` shows the raw ID numbers instead of names. If the owner shows as `65534`, the ownership change didn't take.

<br>

![](images/Pasted%20image%2020260917112144.png)

<br>

## Testing Before GPU Passthrough

I planned to pass the i5-6500's integrated GPU into the Jellyfin container for hardware transcoding. Before doing it, I tested playback on the devices we actually use.

First I had to generate a test video with:

```
/usr/lib/jellyfin-ffmpeg/ffmpeg -f lavfi -i testsrc=duration=60:size=1280x720:rate=30 \
  -f lavfi -i sine=frequency=1000:duration=60 \
  -c:v libx264 -c:a aac -shortest /tmp/test-movie.mp4
```

Two devices used Direct Play. The browser only converted the audio, and a GPU can't speed up audio conversion. On top of that, this generation of Intel GPU can't hardware encode HEVC, which is the case where a GPU would help most.

So I retired that phase and skipped the host permission changes that came with it. I'll revisit it if a device ever needs a real video transcode.

## The ffmpeg False Positive

When checking for transcoding, a plain search for "ffmpeg" in the running processes said a transcode was happening when it wasn't. The Jellyfin server process has "ffmpeg" in its own command line.

`ps aux | grep ffmpeg` found Jellyfin's server process.

<br>

![](images/Pasted%20image%2020260916222229.png)

<br>

Searching for the transcoder's full process path gave the correct answer.

<br>

![](images/Pasted%20image%2020260916221916.png)

<br>

## Read-Only Mount Library Scans

When I added files to the media folder from the host, Jellyfin didn't pick them up automatically. The read-only mount doesn't pass along the file change notifications, so a manual library scan is needed.

## DNS Filtering Bypassed by a VPN

During the perimeter audit, I found my laptop wasn't using AdGuard for about two hours. A commercial VPN client was blocking access to devices on the LAN and sending all DNS through its own servers. There was no error and browsing worked normally.

`resolvectl status` showed the VPN link claiming all DNS queries with the `~.` routing domain. AdGuard's query log had no entries from my laptop during that time.

I disconnected the VPN, turned off auto-connect and now only use it on untrusted networks. Details are in the [Home Network Perimeter Audit](Home%20Network%20Perimeter%20Audit.md).

## Power Backup

I live in Florida, and power outages can be common with the weather we see. Before this current setup, I had a Solar Battery Bank with backup power capabilities plugged in. At one point the internet went down, but the internet stayed on. It was great.

Later on, I moved the server and disconnected the Battery Bank as it was getting into hurricane season. Eventually a storm came through late in the night and knocked the power out.

When we woke up, internet was down. I had to go to the server and boot the computer on to get services back up. I didn't have the router as the backup DNS, as I didn't want it to beat out my server. Meaning AdGuard gets skipped.

Follow up projects:
- Turn on Power Restore on the Server BIOS
- Ensure AdGuard, and other services, are set to Start on Boot.
- Decide on Battery Bank use as power backup again.

---
# Summary

This started as wanting ad blocking for the house and grew into a lot more.

The pfSense and PBS part was the most important piece for me. Building the firewall, writing the rule and moving a live service behind it without losing access to backups. Taking it one step at a time and testing before moving on is what made it work.

Jellyfin was a good exercise in testing before building. I had GPU passthrough planned and it would have taken time for no real benefit.

The perimeter audit was a nice way to close it out and make sure my home network was still secure. Nothing is reachable from the internet. But it also showed that a control can be set up correctly and still not be applied on every device. I wouldn't have caught the VPN issue without checking the AdGuard logs.

RAM is my biggest limit at 16 GB. A 32 GB upgrade is cheap for this platform and would help a lot.

Next up:

- Authentik for SSO and MFA in front of my self-hosted services, which ties into my IAM studies
- Vaultwarden for password management in its own container
- A backup plan for the host data on the 8 TB drive
- Mounting the data drives by UUID so device names changing after a reboot don't cause issues

---
# Resources
- [Proxmox VE documentation](https://pve.proxmox.com/pve-docs/)
- [Proxmox Backup Server documentation](https://pbs.proxmox.com/docs/)
- [pfSense documentation](https://docs.netgate.com/pfsense/en/latest/)
- [AdGuard Home](https://github.com/AdguardTeam/AdGuardHome)
- [Jellyfin documentation](https://jellyfin.org/docs/)
- [Tailscale subnet routers](https://tailscale.com/kb/1019/subnets)
- [CIS Critical Security Controls v8](https://www.cisecurity.org/controls)
