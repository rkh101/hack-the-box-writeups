# Hack The Box Write-ups

Structured write-ups for retired Hack The Box machines, covering reconnaissance,
initial access, privilege escalation, remediation, and lessons learned.

> Only retired and authorized training content is published here.
>
## Windows Machines

| Machine | Difficulty | Initial Access | Privilege Escalation | Write-up |
|---|---|---|---|---|
| Access | Easy | Anonymous FTP → Credential Discovery → Telnet | `runas /savecred` with stored Administrator credentials | [View Write-up](retired-machines/windows/access/README.md) |
| Jeeves | Medium | Jenkins Script Console RCE | Meterpreter / SYSTEM escalation | [View Write-up](retired-machines/windows/jeeves/README.md) |
