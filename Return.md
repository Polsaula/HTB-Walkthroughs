# Return

![image](https://user-images.githubusercontent.com/82520079/185070740-9a0d85e9-162f-459c-a3db-c4ea176e8b1c.png)

## Scanning

The first thing we are going to do is run an NMAP scan to find all open ports. We are going to use "-p-" to scan all the 65.535 ports, "-sS" to perform 
a TCP SYN scan, "--min-rate 5000" to increase scan speed, "--open" to display only open ports, "-Pn" to skip host discovery.

`````
# sudo nmap -p- -sS --min-rate 5000 --open -n -Pn -oG allPorts 10.10.11.108
[...] 
PORT      STATE SERVICE
53/tcp    open  domain
80/tcp    open  http
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
9389/tcp  open  adws
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49671/tcp open  unknown
49674/tcp open  unknown
49675/tcp open  unknown
49679/tcp open  unknown
49682/tcp open  unknown
49694/tcp open  unknown
54360/tcp open  unknown
`````
Now that we know all the open ports of the remote machine, let's have a deeper look at each one of them:

`````
# nmap -sC -sV -p53,80,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001 -oN targeted 10.10.11.108

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: HTB Printer Admin Panel
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2022-07-28 16:08:26Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: PRINTER; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 18m34s
| smb2-time: 
|   date: 2022-07-28T16:08:34
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.53 seconds
`````
## Understanding the remote machine

First things first, check if the remote machine is up and running. Let’s send a ICMP request packet and wait for the response: 

`````
# ping 10.10.11.108 -c 1
PING 10.10.11.108 (10.10.11.108) 56(84) bytes of data.
64 bytes from 10.10.11.108: icmp_seq=1 ttl=127 time=42.2 ms

--- 10.10.11.108 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
`````

As we’ve received the ICMP response packet, now we know the machine is active. I also like starting with this command because we can check the remote OS that is running with the TTL (time to live) value. Remember that for TTL=128 we are dealing with a Windows machine and for TTL=64, a Linux machine. Looking at this output, the value is closest to 128, so we know we are working with a Windows machine. 

This machine has the following services active: Port 53 (Simple DNS Plus), port 80 (Microsoft IIS httpd 10.0), port 88 (Microsoft Windows Kerberos), port 135 (Microsoft Windows RPC), port 139 (Microsoft Windows netbios-ssn), port 389 (Microsoft Windows Active Directory LDAP) and port 5985 (Microsoft Windows Remote Management).

## Port 80

Navigating through the web page we find the Settings screen. Where there is a form where we can insert credentials and IP and port. 

<img width="100%" alt="image" src="https://user-images.githubusercontent.com/82520079/185071459-02edd176-f113-44ec-aba9-f5e80be56078.png">
 
We see there is a field with a password written in it. So, our first thought is that maybe we can see this password on plain text. But if we Inspect the web page and look for the input tag of the password, we’ll see the type isn’t “password”, it is “text”. Meaning that the input of this field is literally “*” characters. To find the actual password we can assume that behind this request a real password is being sent. So we are going to start a listener on our computer with Netcat and change the Server Address of the web page so that the request is sent to our machine. 
 
<img width="100%" alt="image" src="https://user-images.githubusercontent.com/82520079/185071502-709726a2-0666-4e67-a9af-4aacf1d64426.png">

And if we click the “Update” button we are going to get “1edFg43012!!”, which looks like the password for the “svc-printer” user.

<img width="100%" alt="image" src="https://user-images.githubusercontent.com/82520079/185071537-6a2b594e-fa48-4822-8ff4-a11a3f8e5c80.png">

Now that we have credentials, we can use a tool called “Crackmapexec” to verify if these credentials are valid. 

<img width="100%" alt="image" src="https://user-images.githubusercontent.com/82520079/185071578-d1ffd8a2-3b41-41d1-82a3-9f96aec807e6.png">

For SMB service we know these credentials are valid. Before using these credentials for tools like “smbmap” or “smbclient”, and as we know that port 5985 is open (Windows Remote Management) we are going to check if we can connect to the Windows Remote Management service using “winrm”. To do so, the user must be inside the group “Remote Management Users”. 
 
<img width="100%" alt="image" src="https://user-images.githubusercontent.com/82520079/185071664-8850d2c7-c3dc-4c01-add1-407e4215d894.png"> 
 
If the output displays “Pwn3d!” it means that the user is inside this group and we can connect to the service using these credentials.

## Port 5985

We are going to use the tool “evil-winrm” (https://github.com/Hackplayers/evil-winrm) to generate a shell with the Windows Remote Management service.

<img width="100%" alt="image" src="https://user-images.githubusercontent.com/82520079/185071710-5aaa47ca-0599-470d-b32d-3c3fb9e7c5f7.png">

And we can easly find the user’s flag:

<img width="100%" alt="image" src="https://user-images.githubusercontent.com/82520079/185071749-1829f92f-a2d5-4a0d-b20b-a1e383ee79cd.png">

Next step will be to find and be able to display the root’s flag. Typically, this is going to be on the Administrator directory, where we shouldn’t be able to access without privileges. Although in this machine we can access it without any trouble, but we cannot display the flag. 

<img width="100%" alt="image" src="https://user-images.githubusercontent.com/82520079/185071779-dc967a40-298f-4090-991d-97c99fc24316.png">

Ideally, what we want to do is become NT Authority\System to be able to read the root’s flag. First we are going to check all the privileges that the user “svc-printer” has: 

<img width="100%" alt="image" src="https://user-images.githubusercontent.com/82520079/185071809-ea495a30-6db0-42ad-afb0-b1adf0d7af2d.png">
 
To get more information of this user, we are going to use the command “net user”, and we are going to look for the groups this user belongs to:

<img width="100%" alt="image" src="https://user-images.githubusercontent.com/82520079/185071865-df606ae8-4282-4988-9b7a-38dc26902c14.png">

When we are in this situation where we don’t know what can or cannot do a user, it is recommended to search it on the internet. Thanks to this link (https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-groups#bkmk-serveroperators) we know that our user can start and stop services, among others.

<img width="100%" alt="image" src="https://user-images.githubusercontent.com/82520079/185071903-9df073b5-dd6a-4a53-b599-184def38b672.png">

The privilege scalation for this machine is going to be done using services.
1)	The first step is to upload Netcat to the victim’s machine. Fisrt we are going to locate the tool, copy it to our current directory and upload it.

<img width="100%" alt="image" src="https://user-images.githubusercontent.com/82520079/185072049-02f0def4-4b90-487a-838a-b1c51afb5b1d.png">
<img width="100%" alt="image" src="https://user-images.githubusercontent.com/82520079/185072096-2c4c90bf-b601-41fb-973a-2e4f5a8180d9.png">

 
2)	Now that we have Netcat on the victim’s machine, we are going to create/modify a service to be able to execute some commands. With “sc.exe” we can create a service and with the “binPath” parameter we can specify what’s going to happen once the service is started. This command is going to stablish a connection to our machine though port 443 once the service is started:

<img width="100%" alt="image" src="https://user-images.githubusercontent.com/82520079/185072132-b90343fa-42c8-4f43-9d90-7653244cd652.png">

But it looks like we don’t have permission to create new services. So, instead of creating one, we are going to try to modify one. To list all services running: 

<img width="100%" alt="image" src="https://user-images.githubusercontent.com/82520079/185072183-c51558ac-59f5-4152-8751-ee0c168022de.png">

Now we want to choose one and modify the “binPath” parameter.

<img width="100%" alt="image" src="https://user-images.githubusercontent.com/82520079/185072228-730b44e9-f27b-4a78-a08c-e1fd0757ebe0.png">

3)	Once this is done, we are going to start a listener on our machine on port 443:

<img width="60%" alt="image" src="https://user-images.githubusercontent.com/82520079/185072247-85ddd3f1-25de-497a-b16b-69521b0ddbc8.png">

And when we stop and start the service we just modified:

<img width="100%" alt="image" src="https://user-images.githubusercontent.com/82520079/185072268-3097c152-4841-46c2-a5d4-71102d19ed0c.png">

We have a shell with Administrator privileges.

<img width="100%" alt="image" src="https://user-images.githubusercontent.com/82520079/185072301-11055d4f-ba5f-41a5-b91c-e82408546399.png">






