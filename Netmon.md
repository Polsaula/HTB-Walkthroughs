# Netmon

![Netmon](https://user-images.githubusercontent.com/82520079/178553810-4292990b-b856-40c1-90d0-be8709afbde8.png)

## Scanning

The first thing we are going to do is run an NMAP scan to find all open ports. We are going to use "-p-" to scan all the 65.535 ports, "-sS" to perform 
a TCP SYN scan, "--min-rate 5000" to increase scan speed, "--open" to display only open ports, "-Pn" to skip host discovery.

````
# sudo nmap -p- -sS --min-rate 5000 --open -n -Pn 10.10.10.152

PORT      STATE SERVICE
21/tcp    open  ftp
80/tcp    open  http
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
5985/tcp  open  wsman
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown
`````

Now that we know all the open ports of the remote machine, let's have a deeper look at each one of them:

````
# nmap -sC -sV -p21,80,135,139,445,5985,47001 -oN targeted 10.10.10.152

PORT      STATE SERVICE      VERSION
21/tcp    open  ftp          Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 02-03-19  12:18AM                 1024 .rnd
| 02-25-19  10:15PM       <DIR>          inetpub
| 07-16-16  09:18AM       <DIR>          PerfLogs
| 02-25-19  10:56PM       <DIR>          Program Files
| 02-03-19  12:28AM       <DIR>          Program Files (x86)
| 02-03-19  08:08AM       <DIR>          Users
|_02-25-19  11:49PM       <DIR>          Windows
80/tcp    open  http         Indy httpd 18.1.37.13946 (Paessler PRTG bandwidth monitor)
|_http-trane-info: Problem with XML parsing of /evox/about
| http-title: Welcome | PRTG Network Monitor (NETMON)
|_Requested resource was /index.htm
|_http-server-header: PRTG/18.1.37.13946
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2022-07-12T08:27:44
|_  start_date: 2022-07-11T23:28:58
````
## Understanding the remote machine

First things first, check if the remote machine is up and running. Let’s send a ICMP request packet and wait for the response:
````
# ping 10.10.10.152 -c 1
PING 10.10.10.152 (10.10.10.152) 56(84) bytes of data.
64 bytes from 10.10.10.152: icmp_seq=1 ttl=127 time=42.0 ms

--- 10.10.10.152 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
````
As we’ve received the ICMP response packet, now we know the machine is active. I also like starting with this command because we can check the remote OS that is running with the TTL (time to live) value. Remember that for TTL=128 we are dealing with a Windows machine and for TTL=64, a Linux machine. Looking at this output, the value is closest to 128, so we know we are working with a Windows machine. 

This machine has FTP, HTTP, RPC and SMB services active. Let’s check each port. First thing we notice for port 21 is that anonymous login is allowed. So, this is going to be one of the first things to check. Port 80 is running HTTP (Indy httpd 18.1.37.13946) and we also see that is running PRTG Network Monitor, which is an agentless network monitoring software. Then there are ports 135 (Microsoft Windows RPC), 139 (Microsoft Windows netbios-ssn) and port 445 (Microsoft Windows Server 2008 R2). We also find uncommon ports running HTTP service.

### Port 21

As previously told, we are going to start scanning port 21 with the anonymous FTP login. Once logged in we can run commands like “dir” or “ls” to list the directory files.
````
ftp> dir
229 Entering Extended Passive Mode (|||57518|)
150 Opening ASCII mode data connection.
02-03-19  12:18AM                 1024 .rnd
02-25-19  10:15PM       <DIR>          inetpub
07-16-16  09:18AM       <DIR>          PerfLogs
02-25-19  10:56PM       <DIR>          Program Files
02-03-19  12:28AM       <DIR>          Program Files (x86)
02-03-19  08:08AM       <DIR>          Users
02-25-19  11:49PM       <DIR>          Windows
226 Transfer complete.
````

After very short research we can find the user.txt file on the path /Users/Public. We can use the command “get” to download the file into our own machine and be able to read it. 

````
ftp> ls
229 Entering Extended Passive Mode (|||57617|)
150 Opening ASCII mode data connection.
02-03-19  08:05AM       <DIR>          Documents
07-16-16  09:18AM       <DIR>          Downloads
07-16-16  09:18AM       <DIR>          Music
07-16-16  09:18AM       <DIR>          Pictures
07-11-22  07:29PM                   34 user.txt
07-16-16  09:18AM       <DIR>          Videos
226 Transfer complete.
ftp> get user.txt
local: user.txt remote: user.txt
229 Entering Extended Passive Mode (|||57619|)
150 Opening ASCII mode data connection.
100% |***********************************************************************|    34        0.76 KiB/s    00:00 ETA
226 Transfer complete.
34 bytes received in 00:00 (0.76 KiB/s)
````

### Port 21

There is nothing else interesting on the FTP service. Now let’s move to the HTTP. Before anything else I like running the “whatweb” command to list all services and versions this web is running.

<img width="100%" alt="image" src="https://user-images.githubusercontent.com/82520079/178553641-9498f209-3769-427a-918a-fc55bb875762.png">

When opening the web browser we are redirected to a login screen for the network monitor. 

<img width="100%" alt="image" src="https://user-images.githubusercontent.com/82520079/178554022-613edc74-4f89-485f-bc2c-cef05aa956b5.png">

First thing we want to do is check if there are some default credentials to try to login. We find that “prtgadmin” is the default username and password, but it seems that credentials have been changed. Let’s try to discover more about PRTG and see if we can find something interesting. After some research we find that some configuration and data files are stored locally on the remote machine and using FTP we can access them accessing the following path “%programdata%\Paessler\PRTG Network Monitor”.
 
````
ftp> ls
229 Entering Extended Passive Mode (|||58542|)
150 Opening ASCII mode data connection.
12-15-21  08:23AM       <DIR>          Configuration Auto-Backups
07-11-22  08:00PM       <DIR>          Log Database
02-03-19  12:18AM       <DIR>          Logs (Debug)
02-03-19  12:18AM       <DIR>          Logs (Sensors)
02-03-19  12:18AM       <DIR>          Logs (System)
07-12-22  12:00AM       <DIR>          Logs (Web Server)
07-11-22  08:00PM       <DIR>          Monitoring Database
02-25-19  10:54PM              1189697 PRTG Configuration.dat
02-25-19  10:54PM              1189697 PRTG Configuration.old
07-14-18  03:13AM              1153755 PRTG Configuration.old.bak
07-12-22  05:58AM              1733386 PRTG Graph Data Cache.dat
02-25-19  11:00PM       <DIR>          Report PDFs
02-03-19  12:18AM       <DIR>          System Information Database
02-03-19  12:40AM       <DIR>          Ticket Database
02-03-19  12:18AM       <DIR>          ToDo Database
226 Transfer complete.
````

And we are going to download .dat, .old and .old.bak files. If we go through the three of them, we are going to find that on the .old.bak file there are credentials stored: “prtgadmin:PrTg@dmin2018”.

<img width="50%" alt="image" src="https://user-images.githubusercontent.com/82520079/178554261-4eaac73a-5e3a-4474-9005-f75c38ea09d6.png">

But if we try these credentials, we can see that these are incorrect. Let’s remember that these credentials have been obtained from a backup file, and if we use some logic, we realise that the password has the year 2018 written in it. So maybe this was the password used for that year and we can suppose that this password has been changed. Password “PrTg@dmin2019” does work. And we can access to main screen.

<img width="100%" alt="image" src="https://user-images.githubusercontent.com/82520079/178554352-d5b8c2e7-14d0-4ac7-822b-e3315ef1749c.png">

Now that we are logged in, it’s time for us to explore this webpage, but soon we’ll realise that there’s nothing to do in here. Now that we have correct credentials, we can look for exploits on the internet that can give us RCE. There is this post PRTG Network Monitor Remote Code Execution that talks about a Metasploit module that exploits a vulnerability found on this network monitoring software. We are going to start Metasploit and search for this exploit.

<img width="100%" alt="image" src="https://user-images.githubusercontent.com/82520079/178554454-2589724f-a461-4fd0-b101-57b118d49bcb.png">

We are going to change the ADMIN_PASSWORD for “PrTg@dmin2019”, the RHOSTS to 10.10.10.152 and the LHOST to the ip address of our own machine.

<img width="100%" alt="image" src="https://user-images.githubusercontent.com/82520079/178554481-5726ae5d-8d25-423c-9f26-dd56cf61e315.png">

Nice! We have been able to open a Meterpreter session! The only thing left to do is search the root.txt flag, which can be found on the Administrator directory!

<img width="100%" alt="image" src="https://user-images.githubusercontent.com/82520079/178554654-64654e4e-9ffd-4efd-a1f5-6104e13f2f52.png">



