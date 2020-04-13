# DC9- Vulnhub

This is my walkthrough of Vulnhub machine [DC-9](https://www.vulnhub.com/?q=DC-9).  

Once the VM is up, we need to find the IP address of it. I setup my kali VM and DC9 VM in the same NAT network. So, we can use `netdiscover` or `arp-scan`. Both these tools send arp messages to find all the IPs in a given subnet.
```
netdiscover -r 10.0.2.0/24
arp-scan --interface eth0 10.0.2.0/24
```
**1-arpscan.png**

Looks like 10.0.2.19 is the DC9 machine
Lets run a quick nmap on this
`nmap -sA -sV -oA nmap/dc9 10.0.2.19`

**2-nmap.png**

port 22 is filtered, which means there is some kind of firewall rule blocking the port. iptables is generally used to configure such rules and reject packets reaching to the port.

Moving on to the website, it has 4 php pages. display.php has all the staff records with user id, usernames, phone numbers and email addresses. The search page can be used to search the staff details using first or last name. The manage page needs login.

Lets start by running `gobuster` on the site to find all the pages
```
gobuster dir -u http://10.0.2.19 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php 2>&1 | tee gobuster.txt
```
**3-gobuster**

There are some interesting pages */server-status.php, /config.php, /session.php*
Looks like the session.php page automatically logs you in as admin.
The page says *You are already logged in as admin* and it gives extra menu options to add record and logout
It also says *File does not exist*(This might be hinting that we need to give a valid file name somewhere in the request.)
Wierdly, the *add record* option doesn't have any functionality.

Lets go back to the search page and see if we can inject anything there.
*mary'* gives 0 results, but *mary'-- -* still gives Mary's details. This means it is vulnerable to SQL injection

lets open the request in burp

**4-burp.png**

We can copy the request to a file and then use the file in `sqlmap`
This looks like a UNION SQL injection, so lets use sqlmap with union technique
```
sqlmap -r req --technique=U --dump
sqlmap -r req --dbs
sqlmap -r req -D users --dump
```
**5-tables.png**
**6-users.png**

Looks like it is using MySQL
We can find the credentials of all the users and these these credentials work for logging in into the site.
There is a hash for admin from the users table, we can use hashcat to decrypt the MD5 hash. We can also search if it is matching any known hashes in hashkiller.io
The hash matches with *transorbital1*
Lets login the with these admin credentials

Now, we can add new records. I've tried for any command injection through the new records but nothing seems to work.

The page still shows *File does not exist*. So, lets try some file inclusion strings like
 `?file=../../../../../etc/passwd`

It still show file does not exist but also prints out /etc/passwd
We can do a fuzz to find what files are present and enumerate and find any ways to get a reverse shell

At this point, I was stuck and got some help from other walkthroughs.
Apparently, there is a `knockd` process which provides port knocking service (a port is opened only if the port is scanned in a specific way)
The conf file is in /etc/knockd.conf

**7-knockd-conf.png**

According to the conf file, ports 7469, 8475, 9842 must be scanned for port 22 to be open. And they have to be scanned in reverse order to close port 22
We can quickly run nmap for just these ports in this order.
Or we can run nmap on all ports without randomization of port numbers (which is how it does by defaut). We can do this since the ports we need are in ascending order
```
nmap -p- -r 10.0.2.19
nmap -p 22 10.0.2.19
```
now, port 22 is open

**9-sshopen.png**

Now, since we have admin and other users' credentials, we can try ssh'ing
ssh was failing for admin and marym users. We can use `medusa` to see which users are able to login. we can create a creds file from the sqlmap output and use it with medusa.
```
cat UserDetails.csv | awk '{split($0,a,","); print a[2] ":" a[5]}' > creds
for i in $(cat creds); echo 10.0.2.19:$i; done > credsCombo
medusa -C credsCombo -M ssh
```
Only chandlerb, joeyt and janitor can login. Lets login and enumerate using linenum.sh or linpeas.sh

I didn't find anything interesting in joeyt and chandlerb. But janitor has some passwords

**9-janitor.png**

We can try these passwords again with the rest of the users and see if they work. This time, we don't know which password works for which user, so lets create users file with all usernames and passwords file with these newly found passwords. `medusa` can check all combinations to see if any password works with any user
```
medusa -h 10.0.2.19 -U users -P passwords -M ssh
```
The user fredf (Password: B4-Tru3-001) is able to successfully login. So, lets login with these creds and enumerate again. This time we can see that fredf has some permissions in sudoers file

We can run `sudo -l` to see the permissions
fredf has permissions to use `/opt/devstuff/dist/test`
When I try to run, it says  `Usage: python test.py read append`
going through test.py, it looks like the script takes two files as arguments, reads the first file and appends it to the second file

**10-test.py**

Since we use sudo while running the test binary, we can use it to edit sudoers rules

Lets create a temporary file with the permissions to append to sudoers
```
echo 'fredf	   ALL=(ALL:ALL) ALL' > sudoAdd
sudo /opt/devstuff/dist/test/test sudoAdd /etc/sudoers
sudo bash
```
**11-root.png**

In the root folder there is a file 'theflag.txt' and this is the flag

**12-flag.png**
