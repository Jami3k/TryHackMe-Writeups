# Lookup

Hey guys! This is a writeup for the **Lookup** room on TryHackMe, marked as easy. 
In this writeup I will not only outline the correct path, but every step that I took, even if it proved to be wrong.
This is my first writeup, so I'd love some feedback from you all!

## Tasks

### Enumeration

First of all, I added the ip address of the machine to my /etc/hosts file, and called it "machine" for convenience. I then enumerated the running services using **nmap**:

``` markdown
nmap machine
```
and in the background a more thorough scan:

``` markdown
nmap machine -p-
```
which doesn't yield us any additional results.

The first scan shows that ports 22 and 80 are open, so I did a more aggressive scan on those ports using nmap's -A flag:

 ``` markdown
nmap machine -p 22,80 -A
```

Results:

``` bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.9 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 10:dd:24:f7:be:ec:16:90:e2:f2:e5:c0:bc:ae:82:91 (RSA)
|   256 b9:78:83:f4:07:39:d7:97:ff:ec:fe:96:68:3a:d7:62 (ECDSA)
|_  256 10:86:ed:55:e6:69:75:c6:84:f1:6d:6d:f3:37:e1:cb (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://lookup.thm
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.29 seconds

```

In the results, we can see that it says "Did not follow redirect to http://lookup.thm, and when we try to visit the webpage in the browser, we get an error. 

We add "lookup.thm" behind out initial /etc/hosts entry for machine, and when reloading the tab we can now visit the website. 

<img width="1917" height="959" alt="login_page_lookup_thm" src="https://github.com/user-attachments/assets/f58e846d-5ac6-4ddd-acc1-167418a4faf7" />

### Initial Access

We are greeted with a login page. As always, I first check the source code, where I find nothing of interest. Next, I try the default credentials admin:admin. We are send to another site which tells us that we used a wrong password, and returns us shortly after. 

I noticed that when we entered a random string as a username and password, it didn't say "wrong password", but "wrong username and password", which means we can enumerate users. 
Since we already found the username "admin" by chance, we'll run hydra on it: 
```bash
hydra -l admin -P Schreibtisch/Hacking/Wordlists/rockyou.txt lookup.thm http-post-form "/login.php:username=^USER^&password=^PASS^:F=Wrong password" -vV
```
After a couple seconds, it returns success for the password "password123", but when we enter the credentials admin:password123, we are greeted by "Wrong username or password" instead of logging in or a "wrong password" message. This must mean that the password is valid, but not for the admin user. 

Using the wordlist from https://github.com/jeanphorn/wordlist, I used the tool **ffuf** to enumerate valid usernames. Initially, I wanted to use the *BurpSuite Intruder* tool, but the attack proved too slow as they were being time-throttled since I only own the community edition.

While setting up my ffuf command, I remembered to do a directory scan, for which I used **gobuster**:

```markdown
gobuster dir -u http://lookup.thm -w big.txt -x php,html,txt -t 100
```
This revealed no additional information.

Using ffuf with the command 
``` markdown
ffuf -w Wordlists/usernames.txt -X POST -d "username=FUZZ&password=boguspassword" -H "Content-Type: application/x-www-form-urlencoded" -u http://lookup.thm/login.php -mr "Wrong password"
```
eventually gives us another username, jose. We try to login with out previously found password, and we are greeted with another connection error for *files.lookup.thm*. So in our /etc/hosts file, we change lookup.thm to files.lookup.thm, reload the tab, and we're in!

<img width="1918" height="951" alt="elfinder_lookup" src="https://github.com/user-attachments/assets/c0e8badf-bfd7-4cea-8591-57cf0f3ad85d" />

We are in the file manager elFinder, and we see some juicy looking files! 
After looking around a little bit, I tried sshing in as root trying out the passwords in root.txt, but it doesn't work.

While musing about how old this file manager looks, I remembered to search for the version to check https://exploit-db.com for elFinder. We can find the version number unter the help tab (looks like a question mark). Looking at exploit-db.com, we find an RCE vulnerability! 

After downloading it, we run it using 
```markdown
python elfinder_RCE.py http://files.lookup.thm
```
and encounter an error. It seems like we need to have a file named "SecSignal.jpg" in out directory. We just cp a random .jpg to our directory, and then run it again.

Now it doesn't work. I re-check the url, and see that I need to append /elFinder. Run it again, and now we get a shell as www-data!

<img width="572" height="143" alt="Initial_access_lookup" src="https://github.com/user-attachments/assets/a5f9eb7b-81b9-40ed-bc0f-449aa1050326" />

### Privilege Escalation
