---
title: "HTB: Shoppy"
date: 2026-06-30
tags: [hackthebox, htb-shoppy, nosql-injection, mattermost, ghidra, docker, reverse-engineering, easy]
---

Shoppy is an Easy Linux box. The path runs: vhost discovery → NoSQL
authentication bypass → dumping users from a search field → cracking an MD5
hash → into Mattermost → SSH creds → reverse-engineering a binary for the next
user → docker group privesc to root.

## Recon

An nmap scan turns up three TCP ports: SSH (22), HTTP (80), and a Go service on
9093.

```bash
nmap -sC -sV -p- 10.10.11.180 -oN shoppy.txt
```

Port 80 redirects to `shoppy.htb`, so I add it to `/etc/hosts`. The use of a
hostname is a hint to fuzz for virtual hosts.

### Virtual host fuzzing

A vhost fuzz against the IP, varying the `Host` header, surfaces a second site:

```bash
gobuster vhost -u http://shoppy.htb \
  -w /usr/share/seclists/Discovery/DNS/namelist.txt --append-domain
```

This finds `mattermost.shoppy.htb`. (Note: the default subdomains wordlist does
**not** contain `mattermost` — always check that your candidate is in the list
before blaming your filters.)

## Foothold — NoSQL injection

The main site's `/login` isn't SQL-backed. A single `'` makes it hang for 60
seconds (an nginx upstream timeout), and `$ne` operator injection hangs the
same way — a dead end on this box. What works is a JavaScript
string-interpolation payload in the username field:

```
admin' || '1'=='1
```

That bypasses the login and lands on the admin panel. The "Search for users"
function is injectable too: an always-true condition dumps every user, leaking
`josh` and an MD5 hash.

## Cracking the hash

The hash is 32 hex characters — MD5. An online lookup cracks it instantly. The
password gets me into **Mattermost** as `josh`, where a private channel leaks
SSH credentials for `jaeger`.

## Reverse engineering the password manager

`sudo -l` shows `jaeger` can run a `password-manager` binary as the `deploy`
user. It asks for a master password I don't have. `strings` doesn't reveal it —
because the password is built character by character in the C++ source. Opening
the binary in **Ghidra** and reading `main`, the decompiled code appends
`S`, `a`, `m`, `p`, `l`, `e` to a string and compares it to my input. The
password is `Sample`.

(Shortcut: `strings -el` catches it too — the password is stored as a wide /
16-bit string, which default `strings` skips.)

Entering `Sample` prints `deploy`'s credentials.

## Privesc to root — docker group

`deploy` is in the `docker` group, which is effectively root. There's an
`alpine` image present. I launch a container with the host filesystem mounted
in, where I'm root, and read the flag:

```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
cat /root/root.txt
```

Rooted.

## Takeaways

- Fingerprint the backend before forcing one injection type — SQL, NoSQL
  operator, and NoSQL string-interpolation are all different doors.
- A consistent 60-second timeout means "wrong vector, pivot."
- When a secret isn't in `strings`, it may be assembled in code (read it in a
  decompiler) or stored as a wide string (`strings -el`).
- After every foothold: `id` and `sudo -l` first. The docker group was the
  whole privesc.
