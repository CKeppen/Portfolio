Author: Cody Keppen [LinkedIn Profile](https://www.linkedin.com/in/cody-keppen-a09068355/)

Date: 04/22/2026

---
# Table of Contents
- [Preview of Exercise](#preview-of-exercise)
- [Concepts Demonstrated](#concepts-demonstrated) 
- [Instructional Steps](#instructional-steps)
- [Case Notes](#case-notes)
* [Lessons Learned](#lessons-learned)
- [Summary](#summary)
- [Resources](#resources)

---

# Preview of Exercise

These are some brief demonstrations using tcpdump and Wireshark. There are multiple `.pcap` files available to use. In which I'll filter, parse and analyze the data. Eventually finding malicious software.

#  Concepts Demonstrated

1. **tcpdump** Understanding
2. Filtering Data
3. Parsing Data
4. Various Flag Uses
5. **WireShark** Use
6. **CyberChef** Use
7. Malware findings

---
# Instructional Steps

While in Kali as sudo, I'm going to analyze the pcap file called `magnitude_1hr.pcap`

![pcap file](images/Pasted%20image%20 20260422190145.png)

## tcpdump Specific IP Host and Port Filter

In this instance, I need to look at the IP Host `192.168.99.52`

I use the following to get these first 5 lines from the `.pcap`.

```Bash
tcpdump -n -r magnitude_hr1.pcap host 192.168.99.52 -c 5
```

![](images/Pasted%20image%20 20260422190534.png)

I used the `-c 5` flag to keep the screenshot clean, only showing the first 5 results.

The `-n` flag is used to not resolve the addresses, like returning Google for `8.8.8.8` or port `443` being HTTPS.

The `-r` flag along with the `.pcap` is to read the file.

`host 192.168.99.52` declares this to be the IP to filter for.

Looking at the results, you can see a timestamp, protocol, source IP and Port to destination IP and Port, different flags and data size.

![](images/Pasted%20image%20 20260422191355.png)

Let's say I now need to narrow this down to a specific port, like port `80` for HTTP, I add `and port 80`.

```Bash
tcpdump -n -r magnitude_hr1.pcap host 192.168.99.52 and port 80 -c 5
```

![](images/Pasted%20image%20 20260422191620.png)

I see a `GET` request, so let's run a the ASCII decode flag `-A` to get a better idea of what is in the payload.

```Bash
tcpdump -n -r magnitude_hr1.pcap host 192.168.99.52 and port 80 -c 5 -A
```

![](images/Pasted%20image%20 20260422192118.png)

## Analysis

I would say seeing a `hxxp[://]www[.]bankofbotswana[.]bw/` and host `wilfredcostume[.]bamoon[.]com` is enough to look more into this.

![](images/Pasted%20image%20 20260422193602.png)

Here I run the previous command but with `and tcp` to get some more GET requests and expanded the count to 10, `-c 10`.

```Bash
tcpdump -n -r magnitude_hr1.pcap host 192.168.99.52 and port 80 and tcp -A -c 10
```

The concern is seeing `FromBase64String` and the `IO.MemoryStream` object.

I look up what `IO.MemoryStream` is and it looks to be a way to process data in memory. Which gives me the idea something doesn't want to be seen executing on disk.

![](images/Pasted%20image%20 20260422195804.png)

## CyberChef and WireShark Use

First I tried "From Base64" in CyberChef and eventually find the file to be compressed with Gzip using "Detect File Type". (When I went back and reviewed the tcpdump of the GET packet, it did mention gzip encoding.)

![](images/Pasted%20image%20 20260422195329.png)

But when I went to unzip the string, I got an error. Which I assumed was that I didn't have the full string, causing a cutoff of the string.

![](images/Pasted%20image%20 20260422201323.png)

In a way to quickly get the string, I go to WireShark.

Using the filter, ``ip.addr==192.168.99.52 && ip.addr==68.183.138.51`` I find a `GET` packet. Then right click and use, Follow > TCP Stream to get the Request and Response to find the full string.

![](images/Pasted%20image%20 20260422200933.png)

Now I can see a script and another Base64 string embedded. Which I'm going to assume is a payload.

![](images/Pasted%20image%20 20260422200622.png)

I tried to decode this one, but couldn't. Generated hashes and checked VirusTotal, found nothing.

![](images/Pasted%20image%20 20260422201822.png)

I did run the decoded script through ChatGPT for quick analysis. Which confirmed the malware traits of "PowerShell in memory exection". Avoiding detection with `IO.MemoryStream`, calling functions from memory, using the Base64String in the code as a hidden payload. I talk about this more in the [Lessons Learned](#lessons-learned) section.

Additionally, the below from ChatGPT gives me information I can use to decode the second string as it mentions an XOR obfuscation key.

![](images/Pasted%20image%20 20260422204832.png)

Going back to CyberChef, I use From Base64, XOR with the key of 35, then look for strings and get plenty of hits of strings.

![](images/Pasted%20image%20 20260422204742.png)

I ended up at the Minimum length of 16 just to get rid of the random "WATAUAVAH" strings.

![](images/Pasted%20image%20 20260422205034.png)

I found a [light hearted site](https://www.hexacorn.com/blog/2013/05/16/uvwatauavawh-meet-the-pushy-string/) that explains these as opcodes.

![](images/Pasted%20image%20 20260422205210.png)

Of concern is with the default minimum length of 4, revealing `MZARUH`. Which is an IOC for Cobalt Strike.

![](images/Pasted%20image%20 20260422210406.png)

I found a [DFIR Report execution section](https://thedfirreport.com/2024/04/29/from-icedid-to-dagon-locker-ransomware-in-29-days/#execution) on Cobalt Strike that provides some more IOC's of Cobalt Strike that I found in the initial decoding. I go in more details in the [Lessons Learned](#lessons-learned) section.

![](images/Pasted%20image%20 20260423124545.png)
# Case Notes

Here are my quick case notes for this demonstration.

- Using `magnitude_hr1.pcap` suspicious activity found with IP `68.183.138.51` to `192.168.99.52`
- With tcpdump ASCII decode, a GET request is found with following:
	- HTTP: `hxxp[://]www[.]bankofbotswana[.]bw/`
	- Host: `wilfredcostume[.]bamoon[.]com`	  
	- Found `FromBase64String` in GET request with `IO.MemoryStream` object creation	
	![](images/Pasted%20image%20 20260422193602.png)
-  Using Wireshark TCP Stream, found the full string and decoded on CyberChef using FromBase64 and Gunzip. (In a real setting, I would provide a full attachment.)
- String included `func_get_proc_address` and `func_get_delegate_type`, supporting "PowerShell In Memory Execution" tactics to hide execution in memory, avoiding disk writing
- Found another Base64String inside the script, assume to be payload
- First script included Base64 and XOR-obfuscation key of `35` 
- Using CyberChef with Base64 and XOR-obfuscation for embedded encoded script, found `MZARUH`, known Cobalt Strike string (Would include file attachment.)
- **Escalation: IOC of Cobalt Strike found**
  ![](images/Pasted%20image%20 20260422212230.png)

---
# Lessons Learned

## ChatGPT can be a quick way to decode along side CyberChef

Having ChatGPT on the side to throw the Base64 encoding into for quick analysis was interesting. It broke down the encoding methods for me to decode using CyberChef. While giving me a general idea of what the script is trying to do.

I was having a hard time on that second Base64 encoding, but that fact that it identified Base64 decoding and a XOR obfuscation key allowed me to use CyberChef for the second decoding.

## PowerShell In Memory Execution

This is the process of running PowerShell scripts in RAM as to not have anything written to disk evade detection. In this example `IO.MemoryStream` was used with an Base64 encoded string. This becomes a fileless malware that is harder to detect for antivirus software.

##  "MZARUH" and other IOCs for Cobalt Strike

In the [DFIR Report](https://thedfirreport.com/2024/04/29/from-icedid-to-dagon-locker-ransomware-in-29-days/#analysts), the script in the report is almost identical. The two functions, `func_get_proc_address` and `func_get_delegate_type` are used to load and execute the code in memory. The encoded Base64 string in the script is the stageless beacon, which is also encoded in Base64 but is also XOR encoded with the decimal key 35.

With `GetModuleHandleA` and `GetProcAddress` memory is allocated and the now decoded shellcode is injected.

This is where the "MZARUH" decoded string shows up, as the two headers for Cobalt Strike, `magic_mz_x86` and `magic_mz_x64`, result in that decoded string.

## "UVWATAUAVAWH" is a common string to find as it is opcodes (expand)

From the post on "UVWATAUAVAH" this is a very common string to find in a Windows environment. These are a bunch of `PUSH` opcodes that create a large number of variations of this strings in HEX.

Nothing that should be alarming, but something to be aware of when hunting for strings with HEX, as they can take up a lot of the screen. In my case, I moved the minimum character limit higher to filter these off, but if I didn't notice the "MZARUH" string in the beginning with the lower setting, I could have missed that Cobalt Strike IOC.

Meaning multiple pass throughs should occur when adjusting the minimum character limit as to not miss anything.
# Summary

Using tcpdump, I found odd websites and hosts that led me to dig deeper with tcpdump. Once I found a packet worth following with FromBase64 strings, I moved to WireShark for the TCP Stream function.

I got the string, and started using CyberChef to try decoding what the string was intending to do.

Which led me to finding a Cobalt Strike payload attempting to execute inside memory with PowerShell in memory execution.

tcpdump, WireShark and CyberChef are a fun combo to have. Using AI to find key characteristics of a script is a great tool for speeding up your understanding of the code.

---

# Resources
- https://github.com/strandjs/IntroLabs/blob/master/IntroClassFiles/Tools/IntroClass/TCPDump/TCPDump.md
- https://danielmiessler.com/blog/tcpdump
- https://thedfirreport.com/2024/04/29/from-icedid-to-dagon-locker-ransomware-in-29-days/#analysts
- https://www.hexacorn.com/blog/2013/05/16/uvwatauavawh-meet-the-pushy-string/

