---
title: "SwagShop"
description: "HackTheBox - Linux"
summary: "HackTheBox - Linux"
date: 2026-06-09
---

# SwagShop (Magento)

**Attack Box:** Linux

**IP:** 10.129.229.138

## Enumeration

### Rustscan & Nmap

```python
Open 10.129.229.138:22
Open 10.129.229.138:80

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.29 ((Ubuntu))
|_http-favicon: Unknown favicon MD5: 88733EE53676A47FC354A61C32516E82
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Did not follow redirect to http://swagshop.htb/
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Port 80: HTTP

`Apache httpd 2.4.29 (Ubuntu)` 

```python
Did not follow redirect to http://swagshop.htb/
```

`swagshop.htb` > `/etc/hosts` to resolve the name.

Created an account for the shop site.

![image.png](SwagShop%20(Magento)/image.png)

The site is running the `Magneto` eCommerce platform, now known as the Adobe eCommerce service. I looked around the settings and checked the source code but was unable to find a version number for the platform. Although I do have a yeas: `2014` 

![image.png](SwagShop%20(Magento)/image%201.png)

After researching the platform against exploits, I found a PoC on exploitdb relating to `Magento` from 2015. It is a remote code execution from SQLi, but from admin access. Although the exploit found creates an admin account.

[https://www.exploit-db.com/exploits/37977](https://www.exploit-db.com/exploits/37977)

[https://nvd.nist.gov/vuln/detail/CVE-2015-1397](https://nvd.nist.gov/vuln/detail/CVE-2015-1397)

### Trying the exploit

I made some changes to the code to fit the current site being used:

![image.png](SwagShop%20(Magento)/image%202.png)

![image.png](SwagShop%20(Magento)/image%203.png)

I did not modify the print statement so the credentials are different when shown. I visited the admin panel located at `/index.php/admin/` and was able to login with the credentials: `LhotseUser:LhotsePass` .

### Admin Panel Access

![image.png](SwagShop%20(Magento)/image%204.png)

I continued my research further to find out if there were any more publically available exploits to get a shell from the administrator panel. I found the following authenticated RCE which was referenced early in this post.

[https://www.exploit-db.com/exploits/37811](https://www.exploit-db.com/exploits/37811)

### Running Magescan Enumeration tool

For this exploit to run, I will need the install date for the service which can be obtained potentially using the tool `Magescan` :

[https://github.com/steverobbins/magescan/releases](https://github.com/steverobbins/magescan/releases)

```python
└─$ php magescan.phar scan:all http://swagshop.htb            
```

![image.png](SwagShop%20(Magento)/image%205.png)

![image.png](SwagShop%20(Magento)/image%206.png)

I kept receiving an error when attempting to use the `.py` script from earlier, so I found another on `github` to use:

![image.png](SwagShop%20(Magento)/image%207.png)

[https://github.com/Hackhoven/Magento-RCE](https://github.com/Hackhoven/Magento-RCE)

![image.png](SwagShop%20(Magento)/image%208.png)

This script works with `python3` and is much better, returning the command successfully. 

### Achieving RCE

![image.png](SwagShop%20(Magento)/image%209.png)

## Shell as `www-data`

Sending and catching a reverse shell:

![image.png](SwagShop%20(Magento)/image%2010.png)

In the `local.xml` file previously found, there are credentials for the database within the `db` section:

![image.png](SwagShop%20(Magento)/image%2011.png)

`root:fMVWh7bDHpgZkyfqQXreTjU9`

### Access to the MySQL Database

![image.png](SwagShop%20(Magento)/image%2012.png)

![image.png](SwagShop%20(Magento)/image%2013.png)

A new user is found under the section `admin_user` :

[haris@htbswag.net](mailto:haris@htbswag.net)

`8512c803ecf70d315b7a43a1c8918522:lBHk0AOG0ux8Ac4tcM1sSb1iD5BNnRJp`

![image.png](SwagShop%20(Magento)/image%2014.png)

Unfortunately the hash was not crackable against hashcat and `rockyou.txt` :

### user.txt

```python
www-data@swagshop:/home/haris$ cat user.txt
5c51e945c213a2cdb3afb215bc7a0276
```

## Privilege Escalation

Checking `sudo -l` reveals the usage of `vim` as the `root` user is possible:

```python
www-data@swagshop:/home/haris$ sudo -l
Matching Defaults entries for www-data on swagshop:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on swagshop:
    (root) NOPASSWD: /usr/bin/vi /var/www/html/*
```

Reading this blog, I found it was very easy to privilege escalate using `vim` :

[https://hoop.dev/blog/privilege-escalation-in-vim-a-simple-path-to-root/](https://hoop.dev/blog/privilege-escalation-in-vim-a-simple-path-to-root/)

I created a file in `/var/www/html/`  called `esc` . I opened it using `vim` as `sudo` and ran `:!bash` , which put me in a root shell after hitting enter.

## Shell as `root`

![image.png](SwagShop%20(Magento)/image%2015.png)

```python
root@swagshop:/var/www/html# id
uid=0(root) gid=0(root) groups=0(root)
```

### root.txt

```python
root@swagshop:~# cat root.txt
653110b3bf7df65d35266155d5dfded4
```
