---
layout: post
title:  "OpenAdmin Write Up - Hack the Box"
date:   2020-05-12 10:49:07 -0700
categories: [writeup, htb]
tags: [oscp, htb, openadmin]
---
Recently, the Hack the Box machine OpenAdmin was retired. In light of this, I've decided to refine the notes that I took while rooting this box into a write up. I hope you find it helpful :)
I start all of my Hack the Box machines with `masscan`. I find this to be a quick and dirty way to find out which ports are likely open on the box. As with all scanning tools, but especially with `masscan`, it's important to be careful and to know the environment you're in. `Masscan` can be much faster than `nmap` and it's very easy to negatively affect the network. On the Hack the Box networks, this could just be an annoyance to the other people who might be on the box; but on an actual engagement in an actual client network, being heavy-handed with scanning tools can start knocking stuff over. Concerns of stealth aside, this is obviously very bad.

{IMAGE HERE}

Once my scan has completed, I `grep`, `cut`, and `sed` the ports into a text file that I use in subsequent `nmap` scans using command substitution. 

{put commands here}

I like to add a port number that I know for certain is not open because it helps `nmap` with version detection. I begin my `nmap` scans with the `-sV`, `--reason`, and `-vv` flags. These flags give me information on the services listening on open ports, the reason why `nmap` thinks a port is in a given state, and information about the progress of the scan itself, respectively.

{IMAGE HERE}

Both tools told me that that OpenAdmin has TCP ports 22 and 80 open. Port 22 has `OpenSSH 7.6` listening, and port 80 has `Apache 2.4.29` listening. The output from the `nmap` scan also tells me that I'm looking at an ubuntu machine. Generally speaking, I don't put a lot of time into port 22 at the beginning of a test. It's good practice to check the version for any known vulnerabilities, but in my experience, these are rare. 

After ruling out port 22 as a starting point, I turned to port 80. I test a lot of web apps at work, and even on net pens, coming across web apps is pretty common, so I like to use the [FoxyProxy][foxyproxy] browser add-on to streamline things a little bit. The standard version of the add-on lets you easily toggle HTTP proxying on and off (reducing the time needed to manually change browser settings), and it lets you use reg ex's to set rules on which URLs get sent to your HTTP proxy. This is really helpful when you know the full scope in advance, and it keeps the size of the burp project file down (this is only an issue if you are using the pro version). For Hack the Box, I use `(.*)10\.10\.10\.(.*)`, which matches any 10.10.10.0/24 address over any browser-supported protocol. 

Browsing to 10.10.10.171 defaults to port 80. The server responds with the Apache 2 Ubuntu Default Page which tells me that the admin for this server might be a little lazy. It also tells me that there are likely some other pages or directories available somewhere, and that I need to find them. I tend to use `dirb` for directory brute-forcing, although I know a lot of people who use `dirbuster` or `gobuster`. I find `dirb` to be quicker to fire up and get going, but I recently discovered that `dirbuster`, and likely `gobuster`, have some other functionality that `dirb` does not - namely, they have the ability to fuzz, which is really handy if you know the name of a file being served, but aren't sure what directory it is in. For this one, I just need to know what else is being served, so `dirb` is just fine.

When the tool is finished doing its thing, I am told that there is something being served at `http://10.10.10.171/ona/`. When I browse to this, I see the title is `OpenNetAdmin::Own Your Network`; the foreshadowing makes me laugh. I poke around the portal a little bit to see what I can do. I notice that I am "authenticated" as `guest`, that the current version is 18.11, and that there is a new version of the software available. The update link is to an actual site, which tells me that I should look for known vulnerabilities before spending too much time looking at the interface. I google "OpenNetAdmin 18.11 vulnerability". The first hit is a remote code execution exploit on exploit-db. The exploit is a simple bash script and I can see that it is not doing anything shady, so I copy it into a text file on my machine.

The script takes a URL as its sole argument, so I pass `http://10.10.10.171/ona/` to it and am given a very simple shell. This is my initial foothold into the system, and I begin the next phase of enumeration. I like to think of enumeration not as a single phase at the beginning of a test, but rather as something that is done each time your position changes. It is similar to moving  room to room in a dark house, feeling for stairways, doorways, and light switches along the way. Each time my position with regards to a target machine changes, I want to know who I am impersonating, what does this account have access to, and where am I in the box itself. I also want to know what kind of services are running, what ports are open, who else might be logged on, and the general architecture of the machine. By answering each of these, I am able to pain a clearer picture of my target and what is possible.

* The shell I am using is running as `www-data` which makes sense because that is a standard, low-privileged user often used to manage web servers on linux boxes. 
    
* The web site is being served from `/opt/ona/www`. 
    
Running `netstat -antup` show me that aside from the two ports that I discovered to be listening externally, there are also three ports listening internally: port 3306, which is the default port for a mysql database (!); port 52846, which seems odd; and, port 53. The last one is listening on 127.0.0.53 is used locally to forward DNS requests upstream - I feel fairly confident to pass this one by; it is the other two ports that are interesting to me.

![netstat output]({{ site.url }}/assets/openadmin/ntst.png)

I use `find / -type f -user www-data` to list all of the files that I have access to.

So far, it looks like I'm dealing with a pretty standard LAMP stack application, but one that hasn't been configured very well. Between that and my current user, `www-data`, I have a feeling that I am looking for more things in the common web directories. 

When I was bruteforcing directories earlier, `dirb` had also discovered a few other sites being served by the web server on the machine. At the time, these gave me some insight into other possible users on the box, but not much else. While enumerating after gaining my initial foothold, the directories that the sites are served from proved to be a good series of rabbit holes that did not lead me anywhere interesting. However, I did find a database config file (`local/config/database_settings.inc.php`) in the OpenNetAdmin directory that had some hardcoded credentials for a mysql server. Pretty cool.

![MySQL creds]({{ site.url }}/assets/openadmin/database_creds.png)

While snooping around the web directories, I noticed a directory in `/var/www/` that was owned by the user jimmy and the group internal. Listing the contents of the system's home directory, `/home`, confirmed this to be one of the users on the machine (and it also disproved my earlier hypothesis that the names of the other sites being served had some correlation to user names). The other user on the box is joanna.

![home directory contents]({{ site.url }}/assets/openadmin/home_dir.png)

A lot of the stuff I've seen thus far tells me that jimmy is very likely a lazy admin, or is ignorant to security threats, so I try to use mysql password to ssh into the box as jimmy. Success! The password I found is in fact jimmy's. I check jimmy's home directory, but nothing sticks out as interesting, so I decide to look into what's going on with the web server.

    jimmy:n1nj4W4rri0R!

![internal web server directory]({{ site.url }}/assets/openadmin/internal_web_server.png)

Apache, like most web servers, allows for multiple web sites to be served from the same host. This is called Virtual Hosting, and each site has a configuration file located `/etc/apache2/sites-enabled/`. 

And, sure enough, there are two configuration files in that directory: one for the  web app served on port 80 to the internet, and one for the internal directory that I found previously. Interestingly, the internal one runs as joanna.

I navigate to the internal web app directory and  `cat main.php` first and find some interesting stuff: upon successful authentication to this internal web application, joanna's ssh key is displayed. The path to root is now clearer: I need to access this web app to gain access to joanna's account.

I continue looking through the other two php files in the directory. In `index.php`, referenced in `main.php`, I see that authentication is handled by a hard-coded password hash. This is a no-no. 

![index php code]({{ site.url }}/assets/openadmin/index_auth.png)

I copy and paste the hash into [CrackStation][crackstation] and I am rewarded with a plaintext password to use with the internal application.

    jimmy:Revealed

I know of two ways that I could use to interact with the web app that is being hosted internally: `curl` and ssh port forwarding. SSH tunneling in general is something that I know is invaluable to know how to use correctly, and also something that continues (even to this day) to confuse the hell out of me, so naturally I opted for that route. It would also mean that I could actually interact with the app instead of just reading the HTML. In fact, one of the things that I enjoyed the most about this box the first time I went through it was that it really helped me begin to understand SSH tunneling.

I used `ssh -N -L 8080:127.0.0.1:52846 jimmy@10.10.10.171` to create a proxy using ssh tunneling. I will break this down a little bit:

* `-N` tells SSH that you do not want execute any commands remotely. This flag is totally optional; leaving it out will simply cause SSH to behave otherwise normally and you will login as the user.
    
* `-L` I like to think this stands for "listen" but regardless of how the flag was decided, it tells SSH to forward connections to a local port to a remote one
    
* `8080:127.0.0.1:52846` is the argument for `-L`. From the SSH man page, the positions are: `local_socket/port:host:host_port/socket`
    
    * `local_socket/port` is the port on __your__ machine that you are going to connect to.
        
    * `host` is the IP address of the remote host. In this case, I put `127.0.0.1` because the remote service I am trying to connect to is being hosted on the remote machine's localhost.
        
    * `host_port/socket` is the remote port/socket that will receive the connection forwarded from your local port. In this case, it is `52846`, which is the listening port I discovered previously when I used `netstat`. 
    
    A quick note here: one of the things that has continued to confuse me about ssh port forwarding/tunneling is the syntax. What made it click was looking at the output from `netstat`. Listening ports are listed as `ip:port` where `ip` is the IP address of the network interface the port is listening from. 

Once the SSH proxy is set up, I go back to my browser and navigate to `127.0.0.1:8080`, which takes me to the web app. (Note: If you bind your proxy to local port 8080, then you must have burp turned off).

![internal web page]({{ site.url }}/assets/openadmin/internal_web_page.png)

Upon authenticating as jimmy with the cracked password, I am given joanna's key.

![joanna's key]({{ site.url }}/assets/openadmin/joannas_key.png)

At this point, I got a little stuck. I tried SSH'ing as joanna using this key as an identity file, but it continued to ask for a password. Eventually, I discovered that password hashes can be generated from SSH keys using a script called `ssh2john.py` which can be found in `/usr/share/john/` on Kali. From there, I was able to crack the hash using `john` and `rockyou.txt`. 

    joanna:bloodninjas

Using the key as the identity file, I was able to authenticate as joanna over SSH using the newly found password. I went through another round of enumeration to see what I can do with joanna. One of the first things I will usually check for is sudo rights. On *nix machines, sudo privileges for a user can be granted to everything, or certain binaries can be whitelisted so that the user can only user sudo with them. It goes even further: password-less sudo can be specified (meaning the user doesn't need to enter a password when asking for a sudo rights to the specific binary), and it's possible to also specify which directories a user can access while using `sudo`. Viewing a user's sudo rights is done by entering `sudo -l`. In the case of joanna, this user has no password sudo rights to use `nano` to write files to `/opt/priv`.

![joanna's sudo rights]({{ site.url }}/assets/openadmin/joanna_sudo.png)

Nano is one of those special binaries that allow you to shell-out commands while you're using it. This makes it easy to work with large amounts of text files. It also means that if you run it as sudo, you have a root shell. This is done by opening nano using sudo (in this case you have to send `/opt/priv` as the argument), and then press CTRL+R to tell nano to read a file, and then CTRL+X to tell nano you want to run a command. From there, you enter `reset; sh 1>&0 2>&0` to open a shell.

![getting a shell with nano]({{ site.url }}/assets/openadmin/nano_shell.png)
    
    * `1>&0` tells `sh` to send data from `stdout` to `stdin`.
    
    * `2>&0` tells `sh` to send error messages from `stderr` to `stdin`.
    
    This is honestly some *nix shell stuff that I don't quite understand beyond that `stdin` is the text stream that reads input from the user into the shell, `stdout` is the stream used to display text back to the shell, and `stderr` is the stream used to display error messages.

At this point, it's a very quick to get the root flag.

![i am root]({{ site.url }}/assets/openadmin/iamroot2.png)

Hope you enjoyed, and please shoot me an email if I got something wrong or if there was something I could have better explained.


[foxyproxy]: https://getfoxyproxy.org/
[crackstation]: https://crackstation.net/ 
