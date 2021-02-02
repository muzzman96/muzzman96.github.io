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

We can see that the box is listening on **FTP (21)** and **HTTP (80)**.

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
	

Few items to note here:

**1.HTTP:**
- Running **IIS version 7.5**, this has been information has been pulled from server headers
- Allows the use of the **HTTP TRACE** verb; not useful to us in this scenario but worth noting if this was a pen test
- **HTTP-TITLE is IIS7** (potentially a **default test page**)

**1.FTP**
- **Anonymous login acess allowed**
- A listing of the directory when accessed anonymously

I like to normally go for HTTP first so let's do that


### 3. Further enumeration - HTTP

Accessing the web page we notice that this is indeed a default test page when configuring IIS:

![default_test_page]({{site.baseurl}}/_posts/2021-02-02 15_57_40-Window.png)

If we look at the sourcecode we can see the welcome.png file is hosted locally on the server:

![2021-02-02 16_03_57-Window.png]({{site.baseurl}}/_posts/2021-02-02 16_03_57-Window.png)


### 4. Further enumeration - FTP

As shown earlier FTP allows anonymous access: that is where **username=anaonymous** and **password=anything**. The following demonstrates how to do this:

	ftp $IP
    
    Connected to 10.129.90.59.
	220 Microsoft FTP Service
    Name (10.129.90.59:muzz): anonymous
    331 Anonymous access allowed, send identity (e-mail name) as password.
    Password:
    230 User logged in.
    Remote system type is Windows_NT.
    ftp>



Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.
