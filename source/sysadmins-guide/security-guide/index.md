---
layout: default
title: Security guide
github_link: sysadmins-guide/security-guide/index.md
shopware_version: 5.0.0
tags:
  - security

indexed: true
---

## Introduction

Online shopping is only possible in a secure environment, where customers and shop owners can successfully exchange goods in a reliable and secure way. Online hacking has many purposes, from data theft to web defacing with no real motivation. As such, Shopware implements the most modern security features, ensuring both customers and shop owners can rely on the data they are given access to. However, web security is a much broader topic than just implementing application level hacking protection. In this article, we will cover some of the basic aspects you need to keep in mind when deploying your web server.

<div class="toc-list"></div>

<div class="alert alert-warning">
<strong>Disclaimer:</strong> This guide covers basic and common best practices when deploying a LAMP application. Using it will help you improve your server's security but it is not, by any means, exhaustive or 100% fail safe.
</div>

## Setup

Before we can discuss any potential security issues, we need a working server with a running Shopware installation. In these tests, we use one of the most commonly used server setups online, which also fulfills Shopware's requirements:

- Ubuntu 14.04 LTS 64bits
- Apache 2.4.7
- MySQL 5.5.46
- PHP 5.5.9
- mod_rewrite
- php5_gd
- Shopware 5.1.1

No changes were made besides the ones strictly necessary to make our server run Shopware properly. Now that our test server setup is running, we can start (or try) hacking it.

## Gather information

Now that we have our server running, it's time to try and hack it. To do that, we need to look at our server from an attacker's point of view. An attacker will, at first, only know our shop's URL, which isn't much. So his initial steps will be to try and find out more about our shop, the server in which it is installed, and what else might be hosted on that server. In it's current state, itis quite easy to find out more and useful information about our server:
 
```
curl --head http://localhost/
```

The above command simulates a typical HTTP request to our web server, and prints only the response headers. The output is something like:

```
HTTP/1.1 200 OK
Date: Tue, 27 Oct 2015 14:09:53 GMT
Server: Apache/2.4.7 (Ubuntu)
X-Powered-By: PHP/5.5.9-1ubuntu4.13
Set-Cookie: session-1=3bc888cdd4af450408717f5df68db8bece6510dc; path=/
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate, post-check=0, pre-check=0
Pragma: no-cache
Cache-Control: nocache, private
Content-Type: text/html; charset=UTF-8
```

With a single one line command, the attacker already knows that we are using Ubuntu with Apache 2.4.7 and PHP 5.5.9. He can then look for known vulnerabilities in this specific PHP or Apache version and exploit them. As he also knows that we are using Ubuntu, he can deduct the exact Ubuntu version we are using (each Ubuntu release has a specific PHP and Apache2 version). With that, he can also deduct that, if we are using MySQL (which we are), we will probably be using 5.5.46. He can do the same deduction for all other system packages (however, he cannot know, just yet, if they are installed or not). While most of these steps are deductions that may not be true, it is impressive the amount of information an attacker can gather with just a single, simple page request. So we need to hide this information and we will be a lot safer, right? Well, the answer is not so simple.

### The bigger issue - Security through obscurity 

At this point it's important to take a moment to analyse the above issue in depth. By knowing which software versions we are using, the attacker can look for existing known flaws to try and gain access to our server. These flaws are the real problem with our server. Disclosing software versions, in itself, cannot be considered a true security flaw, as it does not grant access to any confidential information. However, it does make an attacker's life a lot easier. As it is our job to make attackers lifes harder, we should hide this information. 

We can hide our PHP version by changing a simple configuration in our PHP configuration file (located in `/etc/php5/apache2/php.ini`)

```
expose_php = Off
```


To hide the Apache version, you need to edit the Apache configuration file, located in `/etc/apache2/conf-enabled/security.conf`:

```
ServerTokens Prod
```

Restart Apache and try the command again. The `X-Powered-By` header is completely gone, and the `Server` header now says only `Apache`. Best case scenario, our attacker now needs to try harder to access our server.

Hiding potential security flaws from attackers (or hiding information that might lead an attacker to find out a way to compromise our system) is called **security through obscurity**, a topic which is often revisited in discussions about online security practices. While these discussions are not relevant to us, their conclusions are: relying solely on **security through obscurity** is a bad security practice, but using it in conjunction with other, more effective security measures, can prevent or, at least, mitigate or delay attacks. 

We can easily see this using our test system as an example. Suppose that our Apache version has a known security flaw. By explicitly informing our attacker about the Apache version we are using, he can quickly exploit that flaw and compromise our system. If we, instead, hide the version number, we are not fixing the flaw. We are just making it harder for the attacker to find and exploit it. So, as a golden rule of thumb, **do not hide your security vulnerabilities, fix them** (and, after that, hide everything you can)

## Updates and upgrades

Following the previous example, if our Apache (or any other package) has a known issue, it's time to update. Keeping our server software up to date is the easiest and more efficient way to keep our server secure. This includes all layers of software, from the system kernel, to Shopware itself, to any 3rd party Shopware plugins you might be using. Keep in mind that, at any moment, it might only take a single flaw at any point on your server to potentially compromise your whole infrastructure. Attackers always search for the weakest link, so it's your job to ensure that there are none.
  
In our example, we are using an Ubuntu installation. As installing updates is a boring and repetitive task, you probably don't want to do it by hand (especially if you are managing more than one server). You can easily configure an Ubuntu installation to [automatically install security updates](https://help.ubuntu.com/community/AutomaticSecurityUpdates). Other OS and distributions also support this feature, and Google can help you figure out how to enable it.

Keep in mind that different operating systems and different linux distributions have different approaches on how to handle updates. For example, by Ubuntu's policy, when you update your packages for security problems, you will most likely not get any new features or performance improvements when you update your system. Other linux flavours have different approaches, meaning you can end up with a new major version of certain software libraries when you perform a routine system update.

Each approach has its advantages and disadvantages, and while it's not part of the scope of this article to discuss them, you should find out exactly how your operating system handles these. Specifically, if you perform a version upgrade (something other than a security and bugfix release), you should test your system for stability and keep an eye out for new features that you might need to configure in order to keep your system secure.

## Hardening your system

By default, most systems come configured with default settings that make them as easy to install and start using as possible. More often than not, these are not the ideal settings for production environments. As we saw before, both Apache and PHP disclose by default their versions, which is useful when you are in a development or test environment, but should be changed when you use them in production. Production configuration has many goals, but in here we will focus specifically on [hardening](https://en.wikipedia.org/wiki/Hardening_(computing)) certain aspects of your system.

### Remove what you (are sure you) don't need

The following command will give you an overview of the listening services you have on your current system:

```
sudo netstat -tulpn | grep LISTEN
```

You will get something like this:

```
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      980/mysqld           
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN      340/cupsd           
tcp        0      0 127.0.0.1:53            0.0.0.0:*               LISTEN      4874/dnsmasq           
tcp6       0      0 :::80                   :::*                    LISTEN      559/apache2           
tcp6       0      0 ::1:631                 :::*                    LISTEN      340/cupsd           
tcp6       0      0 :::443                  :::*                    LISTEN      559/apache2           
```

Looking at the last column, we can see we have 4 unique services here:
- apache2
- mysqld
- dnsmasq
- cupsd

Apache2 and MySQL (mysqld stands for MySQL daemon) are known to us. Dnsmasq and cupsd are not. The first is a network related service. Cups is a printing server. As we installed our system as a desktop installation, printing is a common task, so it's only natural that we might, at some point, want to print something. But, in reality, our system is a web server. It will most likely not print anything anytime soon, so cups is not really needed. Still, it is running, and it has an open connection. The table above also shows us that, while it is listening for incoming connections on port 631, it is bound only to 127.0.0.1, meaning that it's not accessible from outside of our machine (you can test this by opening `localhost:631` on this machine vs opening `<your server address>:631` from a different host). So, while in reality cups is not accessible from the outside of our server, it's also useless, and nothing ensures us that, in the future, a flaw in it will not allow an attacker to bypass this restriction and access our server through it. So we should consider uninstalling cups altogether.

The above service list might be larger on your system. If you have installed other services like an SSH server or an FTP server, they will also show up there. Be sure to double check that you don't need something before removing it. For example, should you remove the sshd service, you will lose remote SSH access to your server, and will have to have physical access to the server machine to restore it. Like always, Google can be very helpful when determining what can and can't be removed.

### Access hardening

Now that we removed unnecessary services, it's time to improve the security of the necessary ones. The most basic attack on a server is a brute force/dictionary attack, whereby an attacker tries as many different passwords as he cans against your available points of entry (typically SSH and FTP, but can be used pretty much against any authentication mechanism), hoping you used a weak password somewhere. There are multiple ways to protect against this type of attack

#### Fail2ban

`Fail2ban` is a software package available on Ubuntu (and other distributions too) that monitors multiple login sources (for example, SSH) and tries to detect brute force attacks by finding patterns in failed login attempts. If an attack attempt is detected, the sources of those attacks are banned during a specific time window, making these attacks much much slower to perform and virtually useless.

`Fail2ban` supports different protocols and policies, so be sure to check out [the project's homepage](http://www.fail2ban.org/wiki/index.php/Main_Page) for more information on how you can adapt it to your needs.

#### Disable SSH password authentication

Server access is mostly done using SSH connections, which, by default, rely on a password. Passwords, even the unique, good ones, are susceptible to dictionary and brute force attacks. That

### Network layer hardening

Now that your server has no unnecessary services, it's time to start increasing the security of what is still there. 

#### Iptables and ufw

`iptables` is a software firewall that is available on most linux distributions. In Ubuntu, it's available in a package with the same name, and installed by default:

```
sudo iptables -L
sudo ip6tables -L
```

These commands will print the current rule set, respectively, for IPv4 and IPv6. In a fresh install, both will print the same result: 

```
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

As you can imagine, this means that there are no rules defined, and traffic can flow freely in and out of your system. To help us configure `iptables`, we will use another application: `ufw`. `ufw` helps you configure `iptables` rules in a much easier way. By default, it is installed but inactive:

```
sudo ufw status
```

```
Status: inactive
```

You can find the `ufw` configuration file at `/etc/default/ufw`, where you can read the documentation for each option. You can also configure `ufw` by defining rules in the command line:

```
sudo ufw allow ssh
```

The above command allows SSH connections. It is, in reality, a shorthand for the native ufw syntax:

```
sudo ufw allow 22/tcp
```

As you plan on running a web server on you server machine, the following two rules are a must:

```
sudo ufw allow http
sudo ufw allow https
```

Once your configuration is set, you can enable your firewall:

```
sudo ufw enable
```

Keep in mind that, as they are, these rules are lost on reboot. There are ways to persist them, which your favourite search engine will help you find. 


#### Sysctl

`sysctl` is a linux kernel configuration utility that allows you to control the kernel's behaviour during runtime. As you might have guessed by now, it's a very powerful tool, so **be extra careful when changing the following parameters**.

Some default kernel settings regarding the IP protocol can be improved in order to make your system more resilient to disruption of service attacks. In order to learn more about fine tuning your kernel configuration settings, please refer to [this guide](http://www.cyberciti.biz/faq/linux-kernel-etcsysctl-conf-security-hardening/). As always, read about each option before changing it, so that you are sure that your changes will not have any negative side effect on your server.


### Hardening MySQL

#### Securing your connection to MySQL

Going back to [the table above](#remove-what-you-are-sure-you-don), you can see that MySQL is configured to only listen on 127.0.0.1. This is a default setting, and while it prevents attackers from even connecting to your MySQL instance, it also prevents you from legitimately connecting to it to, for example, get an up to date copy of the database, for developing or testing purposes. You might be tempted, at this point, to change that setting. Even if he can connect, without a valid username and password, an attacker cannot log in, and your data is still safe, right? And your life as a sys admin would be a lot easier.
  
While this assumption is true, changing this setting is not a good idea. By changing it, you are effectively removing an obstacle from your attacker's path. Like we said before, it's our job to make an attacker's life as complicated as possible, and that means adding barriers to its path in every way we can. So, unless you have strong reasons to do otherwise, do not change this setting.

This, of course, still leaves you with the problem of connecting to your database in a secure way, without creating shortcuts for your attackers. The easiest and safest way you can do so is by using [SSH tunnel](https://en.wikipedia.org/wiki/Tunneling_protocol#Secure_Shell_tunneling). SSH tunneling requires SSH communication between your server and client, and works like this:

- On your server machine, the SSH server connects to the local MySQL port (3306). As MySQL is configured to accepts connections on localhost, this is not a problem
- The traffic from that MySQL connection is encapsulated and routed through a secure SSH connection to your client machine
- On your client machine, the SSH client decapsulates and exposes that traffic on a specified local port.
- The whole process abstracts itself, meaning you can use any MySQL client to connect to the local port provided by the SSH tunnel, as if it were a normal MySQL server.

The whole process has, of course, many more details which we will not discuss here. You can read more online about the exact command you can use to create a SSH tunnel. SSH tunnels can be created from both the server or the client, and are not exclusive to MySQL connections: they can be used with virtually any type of connections. The important point here is that, using this method, your data is transmitted using a secure connection from your server, while keeping your server safe from potential attackers

#### Use dedicated MySQL users

Another way in which you can improve your MySQL security is by creating individual access accounts to it. This is specially useful if you are hosting multiple services or sites on the same server, with different MySQL databases. As a rule of thumb, you should use an individual MySQL account for each profile that accesses your database. So, for example, you should create a "shopware" (or any other name) user on your database, and grant it access exclusively to the database that will hold Shopware's data. You can also limit each MySQL user account access by host. Again, for your "shopware" user, access from any other host besides localhost is not needed, so go ahead and add that limitation too. Remember that every obstacle you can create for an attacker is a potential advantage you have when defending against an intrusion attempt.


### Hardening Apache2

Apache2 is probably the trickiest part of your system to harden, due to the fact that, unlike all other system components, it is supposed to accept connections from the outside. This greatly broadens the potential interactions between it and your attacker, and as such grants him many more potential attack vectors that you need to defend against.

#### Remove unnecessary modules

Previously, we recommended that you [Remove packages that you are sure you don't need](#remove-what-you-are-sure-you-don). The same principle applies to Apache2 modules. Run the following command:

```
apache2ctl -M
```

This will list all the currently enable Apache2 modules. Some of these modules are absolutely necessary (`core_module` speaks for itself), others are required by Shopware (`rewrite_module` is very very recommended), while other provide useful development features that are not needed in production environments (`autoindex_module` is an example of this). Which modules you might require and which you can remove will depend on your system configuration and on which other web applications you might be running. In any case, be sure to remove modules you suspect you might not need one by one, and properly test your system between each change. You can also find online resources covering each module, which will help you determine what you need and what you don't.

#### mod_evasive

`mod_evasive` is primarily aimed at preventing [DoS and DDoS attacks](https://en.wikipedia.org/wiki/Denial-of-service_attack) against your system. While in Ubuntu 14.04 mod_evasive is readily available from the package repository (`libapache2-mod-evasive`), this might not be true for all OS's. After installing and enabling it, you should take a look at the configuration file (`/etc/apache2/mods-enabled/evasive.conf` on Ubuntu, actual content may vary):
 
```
<IfModule mod_evasive20.c>
    #DOSHashTableSize 3097
    #DOSPageCount 2
    #DOSSiteCount 50
    #DOSPageInterval 1
    #DOSSiteInterval 1
    #DOSBlockingPeriod 10
    #DOSEmailNotify <someone@somewhere.com>
</IfModule>
```

First, notice that every single option is commented out, so don't forget to uncomment them if you change their values. The module offers more options, which you can explore with the help of the existing online resources. As an example, the above constellation will block for 10 seconds every user (based on the IP) that does 2 page hits on the same URL in 1 second, or 50 site hits every 1 second. It will also send an email to the configured email address when an IP address is blacklisted. As you can imagine, this setting constellation is not ideal for production environments, so be sure to configure it to your needs before enabling the module in production.

It is worth noting at this point that DoS and DDoS can be detected and stopped at a much lower level, with reduced system performance impact. It is common for hardware and software firewalls to provide features to mitigate/prevent this kind of attacks, so be sure to use those in conjunction with (or even instead of) `mod_evasive` for better results.

#### mod_security

`mod_security` detects, logs and blocks certain suspicious requests to your web application. It does so by using certain rules (which, in turn, use regular expressions) to inspect certain aspects of incoming requests and try to detect attack attempts. `mod_security` is available in Ubuntu in the `libapache2-mod-security2` package, and you can find the configuration file at  `/etc/apache2/mods-enabled/security2.conf` which, in turn, imports all config files in `/etc/modsecurity` (again, keep in mind that this might not be true for your OS). This folder, in turn, has 2 files. `modsecurity.conf-recommended` has the configuration options for our module (the file itself is well documented) and some basic rules.
 
More extensive rules can be found [here](https://github.com/SpiderLabs/owasp-modsecurity-crs), in a project maintained by [OWASP](https://www.owasp.org/index.php/About_OWASP). These rules are carefully and extensively documented. As a rule of thumb, it is never a good idea to deploy any rules right into production. As detailed in the README file of the above project, these rules aim at detecting generic attacks against any web application. While this might sound like a great feature, it is also often the cause of unexpected behaviour in web applications, in scenarios where the rules block out legitimate request. As such, you should start by implementing a set of rules that you find adequate for your server (again, keep in mind that these rules are applied to all applications and sites served by Apache, and not just Shopware), and changing `mod_security`'s `SecRuleEngine` to `DetectionOnly`. This will cause `mod_security` to detect but not prevent suspicious requests. You can then use this to test you rules, and see if any legitimate access triggers any rules it shouldn't. After using this method to fine tune your rule set, you can change `SecRuleEngine` to `On` to have `mod_security` block out suspicious requests to Apache.











































