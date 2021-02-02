---
published: false
---
## #1. OSCP Prep - HTB Series :: Devel Writeup (without metasploit)

Hi, this is my write up for the easy box **Devel** on Hackthebox. This is a retired windows machine and can be easily exploited in a matter of minutes is Metasploit. However, as this is for OSCP preparation, no metasploit will be used.

![devel_graphi]({{site.baseurl}}/_posts/devel_graphic.PNG)

If you enjoyed this write up feel free to follow me on twitter. Let's get started!! :)

___

### 1. Enumeration

#### nmap

Quick nmap scan on all ports to find the services that the machine has open:

	nmap -p- $IP -oN nmap.txt
    
    PORT   STATE SERVICE
	21 tcp open  ftp
	80 tcp open  http

We can see that the box is listening on FTP (21) and HTTP (80).

Next aggressive scan to perform some service enumeration and default scripts against returned listening ports:

	nmap -p 21,80 -A $IP
	
    PORT   STATE SERVICE VERSION
    21 tcp open  ftp     Microsoft ftpd
    | ftp-anon: Anonymous FTP login allowed (FTP code 230)
    | 03-18-17  01:06AM       DIR          aspnet_client
    | 03-17-17  04:37PM                  689 iisstart.htm
    |_03-17-17  04:37PM               184946 welcome.png
    | ftp-syst: 
    |_  SYST: Windows_NT
    80 tcp open  http    Microsoft IIS httpd 7.5
    | http-methods: 
    |_  Potentially risky methods: TRACE
    |_http-server-header: Microsoft-IIS 7.5
    |_http-title: IIS7
	



Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.
