# Fase 3 — Security Operations

**Estado**: ✅ Completado
**Días**: 8, 9, 10, 11

---

## Infraestructura de monitoreo desplegada

| Componente           | Recurso                          | Propósito                                    |
| -------------------- | -------------------------------- | -------------------------------------------- |
| CloudTrail           | cloudops-lab-trail               | Audit log de toda actividad en la cuenta AWS |
| S3                   | cloudops-lab-logs-martin         | Almacenamiento de logs de CloudTrail         |
| CloudWatch Log Group | /cloudtrail/cloudops-lab         | Logs de CloudTrail en CloudWatch             |
| CloudWatch Log Group | /cloudops-lab/endpoint-a/auth    | auth.log de Endpoint A                       |
| CloudWatch Log Group | /cloudops-lab/endpoint-a/syslog  | syslog de Endpoint A                         |
| CloudWatch Log Group | /cloudops-lab/endpoint-b/auth    | auth.log de Endpoint B                       |
| CloudWatch Log Group | /cloudops-lab/endpoint-b/syslog  | syslog de Endpoint B                         |
| CloudWatch Agent     | Endpoint A (i-0f7d66edc3fddea09) | Envío de logs del sistema                    |
| CloudWatch Agent     | Endpoint B (i-00f37d0d18857a0d9) | Envío de logs del sistema                    |

---

## Alarms de detección configuradas

| Alarm                               | Metric Filter                                | Threshold    | Log Group                     |
| ----------------------------------- | -------------------------------------------- | ------------ | ----------------------------- |
| ssh-brute-force-detected            | "Disconnected from authenticating" "preauth" | > 3 en 5 min | /cloudops-lab/endpoint-b/auth |
| unauthorized-user-creation-detected | "new user" "useradd"                         | > 0 en 5 min | /cloudops-lab/endpoint-a/auth |
| sensitive-file-access-detected      | "COMMAND" "/etc/shadow"                      | > 0 en 5 min | /cloudops-lab/endpoint-a/auth |

---

## Detection Queries — CloudWatch Logs Insights

### Query 1 — SSH Brute Force Detection
```
fields @timestamp, @message
| filter @message like "Disconnected from authenticating"
| filter @message like "preauth"
| stats count(*) as attempts by @logStream
| sort attempts desc
```
**Resultado**: 6 intentos detectados en Endpoint B (i-00f37d0d18857a0d9)

### Query 2 — Unauthorized User Creation
```
fields @timestamp, @message
| filter @message like "useradd"
| filter @message like "new user"
| sort @timestamp asc
```
**Resultado**: 1 evento detectado — creación de backdoor_user en Endpoint A

### Query 3 — Sensitive File Access
```
fields @timestamp, @message
| filter @message like "COMMAND"
| filter @message like "/etc/shadow" or @message like "/etc/sudoers" or @message like ".pem"
| sort @timestamp asc
```
**Resultado**: 3 eventos detectados — acceso a /etc/shadow, find *.pem, /etc/sudoers

---

# Incident Report 001 — SSH Brute Force Attack

**Incident ID**: IR-001-20260610
**Date**: June 10, 2026
**Severity**: HIGH
**Status**: Resolved
**Analyst**: Martin Salvo

---

## Executive Summary

A SSH brute force attack was detected against Endpoint B (ip-10-0-14-33) originating from Endpoint A (ip-10-0-15-138) within the lab VPC. The attack attempted 10 password combinations using the Hydra tool across 5 consecutive runs, generating 15 connection attempts between 10:53 and 10:59 UTC. No successful unauthorized access occurred — the target system rejected all attempts at the pre-authentication stage because password authentication is disabled and only key-based authentication is accepted.

---

## Timeline

| Time (UTC) | Event                                                                                  |
| ---------- | -------------------------------------------------------------------------------------- |
| 10:50:10   | Endpoint B (lab-endpoint-b) started, SSH daemon listening on port 22                   |
| 10:50:28   | Hydra installed on Endpoint A via apt                                                  |
| 10:53:45   | First Hydra execution initiated against 10.0.14.33:22                                  |
| 10:56:39   | First disconnect recorded: `Received disconnect from 10.0.15.138 port 45196 [preauth]` |
| 10:56:39   | `Disconnected from authenticating user ubuntu 10.0.15.138 port 45196 [preauth]`        |
| 10:58:49   | Second wave of attempts — 5 rapid disconnects recorded                                 |
| 10:58:50   | Attempts from ports 53520, 53524, 53540, 53544, 53552                                  |
| 10:59:09   | Final verification query from Endpoint B — 15 total log entries confirmed              |

---

## Detection

**Method**: CloudWatch Logs Insights query against `/cloudops-lab/endpoint-b/auth`

**Query used**:
```
fields @timestamp, @message
| filter @message like "Disconnected from authenticating"
| filter @message like "preauth"
| stats count(*) as attempts by @logStream
| sort attempts desc
```

**Result**: 6 matching events detected in log stream `i-00f37d0d18857a0d9`

**Key log entries**:
```
2026-06-10T10:56:39 ip-10-0-14-33 sshd[25509]: Received disconnect from 10.0.15.138 port 45196:11: Bye Bye [preauth]
2026-06-10T10:56:39 ip-10-0-14-33 sshd[25509]: Disconnected from authenticating user ubuntu 10.0.15.138 port 45196 [preauth]
2026-06-10T10:58:49 ip-10-0-14-33 sshd[25688]: Received disconnect from 10.0.15.138 port 53520:11: Bye Bye [preauth]
2026-06-10T10:58:49 ip-10-0-14-33 sshd[25694]: Received disconnect from 10.0.15.138 port 53544:11: Bye Bye [preauth]
```

---

## Indicators of Compromise (IoCs)

| Type        | Value                          | Description                                     |
| ----------- | ------------------------------ | ----------------------------------------------- |
| Source IP   | 10.0.15.138                    | Endpoint A — internal attacker host             |
| Target IP   | 10.0.14.33                     | Endpoint B — target of brute force              |
| Target Port | 22 (SSH)                       | Standard SSH port                               |
| Pattern     | Multiple [preauth] disconnects | Characteristic brute force signature            |
| Tool        | Hydra v9.5                     | Brute force tool identified via apt install log |
| Username    | ubuntu                         | Default Ubuntu username targeted                |
| Wordlist    | 10 common passwords            | password, 123456, admin, root, ubuntu, etc.     |

---

## Root Cause Analysis

The attack originated from Endpoint A (10.0.15.138), a host within the same VPC subnet. The attacker had prior access to the internal network and used Hydra to attempt password-based SSH authentication against Endpoint B.

The attack failed because:
1. Endpoint B has password authentication disabled in sshd_config
2. Only RSA key-based authentication is accepted
3. Hydra received an immediate rejection at the protocol negotiation stage

The Security Group rule allowing SSH from 10.0.0.0/16 (internal VPC) enabled the connection attempts. In a production environment, this rule would be restricted to specific management IPs or a bastion host.

---

## Response Actions

| Action                                      | Status | Details                                      |
| ------------------------------------------- | ------ | -------------------------------------------- |
| Confirmed no successful unauthorized access | Done   | All attempts rejected at [preauth] stage     |
| Identified source of attack                 | Done   | IP 10.0.15.138 — Endpoint A                  |
| Reviewed auth.log for full scope            | Done   | 15 total entries confirmed                   |
| CloudWatch Alarm configured                 | Done   | Triggers on > 3 preauth disconnects in 5 min |
| Security Group review                       | Done   | SSH from VPC range justified for lab only    |

---

## Recommendations

1. **Restrict SSH access**: In production, SSH should only be accessible from a dedicated bastion host or specific management IPs, never from the entire VPC range.
2. **Implement fail2ban**: Automatically block IPs after N failed attempts. Reduces noise in logs and adds a layer of automated defense.
3. **Change default SSH port**: Reduces automated scanning noise (security through obscurity — not a primary control but useful as a supplement).
4. **Enable SSH key rotation policy**: Regularly rotate key pairs, especially in environments with multiple administrators.
5. **Alert on Hydra installation**: Monitor for installation of penetration testing tools (hydra, nmap, nikto) via package manager logs.

---

## Lessons Learned

- The `[preauth]` pattern in SSH logs is a reliable indicator of brute force attempts — each failed connection before authentication completes generates this entry.
- CloudWatch Logs Insights allows near-real-time detection of attack patterns across multiple log streams without a dedicated SIEM.
- Password authentication should always be disabled on SSH servers — key-based authentication is the baseline security control.

---

# Incident Report 002 — Unauthorized User Creation & Privilege Escalation

**Incident ID**: IR-002-20260610
**Date**: June 10, 2026
**Severity**: CRITICAL
**Status**: Resolved
**Analyst**: Martin Salvo

---

## Executive Summary

A privilege escalation incident was detected on Endpoint A (ip-10-0-15-138). An authenticated user with sudo privileges created an unauthorized local user account named `backdoor_user`, set a password for it, and added it to the `sudo` group — effectively creating a persistent privileged backdoor account. The incident was detected via CloudWatch Logs Insights monitoring of `/etc/passwd` modification events in auth.log. The backdoor account was subsequently deleted as part of incident response.

---

## Timeline

| Time (UTC) | Event                                                            |
| ---------- | ---------------------------------------------------------------- |
| 10:59:30   | `sudo useradd -m -s /bin/bash backdoor_user` executed            |
| 10:59:31   | New user created: `backdoor_user, UID=1001, GID=1001`            |
| 11:00:26   | `sudo passwd backdoor_user` — password set for backdoor account  |
| 11:00:35   | `sudo usermod -aG sudo backdoor_user` — added to sudo group      |
| 11:00:35   | `usermod: add 'backdoor_user' to group 'sudo'` confirmed in logs |
| 11:00:35   | `usermod: add 'backdoor_user' to shadow group 'sudo'` confirmed  |
| 11:03:22   | IR response: `sudo userdel -r backdoor_user` executed            |
| 11:03:22   | User deleted, removed from sudo and shadow groups                |

---

## Detection

**Method**: CloudWatch Logs Insights query against `/cloudops-lab/endpoint-a/auth`

**Query used**:
```
fields @timestamp, @message
| filter @message like "useradd"
| filter @message like "new user"
| sort @timestamp asc
```

**Result**: 1 event detected — unauthorized user creation at 10:59:31 UTC

**Key log entries**:
```
2026-06-10T10:59:30 ip-10-0-15-138 sudo: ubuntu: COMMAND=/usr/sbin/useradd -m -s /bin/bash backdoor_user
2026-06-10T10:59:31 ip-10-0-15-138 useradd[25424]: new user: name=backdoor_user, UID=1001, GID=1001, home=/home/backdoor_user, shell=/bin/bash
2026-06-10T11:00:35 ip-10-0-15-138 usermod[25442]: add 'backdoor_user' to group 'sudo'
2026-06-10T11:00:35 ip-10-0-15-138 usermod[25442]: add 'backdoor_user' to shadow group 'sudo'
```

---

## Indicators of Compromise (IoCs)

| Type             | Value               | Description                                 |
| ---------------- | ------------------- | ------------------------------------------- |
| Backdoor account | backdoor_user       | Unauthorized local user — UID 1001          |
| Privileged group | sudo                | Backdoor user added to sudo                 |
| Home directory   | /home/backdoor_user | Created with -m flag                        |
| Shell            | /bin/bash           | Full interactive shell assigned             |
| Source user      | ubuntu              | Legitimate account used for privilege abuse |
| Timestamp        | 10:59:30 UTC        | Creation time for forensic correlation      |

---

## Root Cause Analysis

The `ubuntu` user, which has unrestricted sudo access, was used to create a backdoor account. This represents an insider threat scenario or a compromised legitimate account being used to establish persistence.

The attack followed a classic persistence pattern:
1. **Initial access**: Already had shell access as `ubuntu`
2. **Persistence**: Created new account (`backdoor_user`) to survive password changes
3. **Privilege escalation**: Added account to `sudo` group for root access
4. **Lateral movement potential**: New account could be used to access any system accepting `ubuntu`'s key or the backdoor password

---

## Response Actions

| Action                           | Status | Details                                        |
| -------------------------------- | ------ | ---------------------------------------------- |
| Identified backdoor account      | Done   | backdoor_user, UID=1001, sudo group            |
| Deleted backdoor account         | Done   | `userdel -r backdoor_user` at 11:03:22 UTC     |
| Verified deletion                | Done   | `grep backdoor /etc/passwd` returned empty     |
| Confirmed removal from sudo      | Done   | Log entry: removed from sudo and shadow groups |
| CloudWatch Alarm configured      | Done   | Triggers on any useradd new user event         |
| Reviewed for additional accounts | Done   | No other unauthorized accounts found           |

---

## Recommendations

1. **Implement sudoers auditing**: Configure auditd to alert on any `useradd`, `usermod`, or `userdel` commands in real time, not just via log review.
2. **Restrict sudo access**: The `ubuntu` user having unrestricted sudo is a security risk. Apply specific command whitelisting in sudoers.
3. **User account baseline**: Maintain a baseline of authorized user accounts and alert on any deviation.
4. **Implement PAM controls**: Use PAM (Pluggable Authentication Modules) to restrict which users can log in and from where.
5. **Regular account audits**: Schedule weekly automated review of `/etc/passwd` and `/etc/group` against an authorized baseline.

---

## Lessons Learned

- Local user creation is fully logged in auth.log via `useradd` entries — this makes CloudWatch monitoring effective for detecting persistence mechanisms.
- Adding a user to the sudo group generates two separate log entries (group and shadow group) — both are detectable.
- The `userdel -r` command generates deletion audit entries, confirming successful remediation in the log trail.
- In a real incident, the first response would be to lock the compromised account (`passwd -l ubuntu`) before investigating, to prevent further actions.

---

# Incident Report 003 — Sensitive File Access & Credential Harvesting Attempt

**Incident ID**: IR-003-20260610
**Date**: June 10, 2026
**Severity**: CRITICAL
**Status**: Resolved
**Analyst**: Martin Salvo

---

## Executive Summary

Following the unauthorized user creation (IR-002), the same threat actor performed a series of sensitive file access operations on Endpoint A (ip-10-0-15-138). The actor accessed `/etc/shadow` (hashed password database), performed a system-wide search for `.pem` files (SSH private keys), and accessed `/etc/sudoers` (privilege configuration). These actions are characteristic of a post-exploitation credential harvesting phase, where an attacker with initial access attempts to gather credentials for lateral movement or persistence.

---

## Timeline

| Time (UTC) | Event                                                    |
| ---------- | -------------------------------------------------------- |
| 11:00:40   | `sudo cat /etc/passwd` — user enumeration                |
| 11:01:21   | `sudo cat /etc/shadow` — password hash access            |
| 11:01:47   | `sudo find / -name *.pem` — SSH private key search       |
| 11:02:03   | `sudo cat /etc/sudoers` — privilege configuration review |

**Note**: All four actions occurred within a 2-minute window (11:00:40 — 11:02:03 UTC), indicating a systematic and deliberate credential harvesting sequence.

---

## Detection

**Method**: CloudWatch Logs Insights query against `/cloudops-lab/endpoint-a/auth`

**Query used**:
```
fields @timestamp, @message
| filter @message like "COMMAND"
| filter @message like "/etc/shadow" or @message like "/etc/sudoers" or @message like ".pem"
| sort @timestamp asc
```

**Result**: 3 events detected

**Key log entries**:
```
2026-06-10T11:01:21 ip-10-0-15-138 sudo: ubuntu: COMMAND=/usr/bin/cat /etc/shadow
2026-06-10T11:01:47 ip-10-0-15-138 sudo: ubuntu: COMMAND=/usr/bin/find / -name *.pem
2026-06-10T11:02:03 ip-10-0-15-138 sudo: ubuntu: COMMAND=/usr/bin/cat /etc/sudoers
```

---

## Indicators of Compromise (IoCs)

| Type           | Value                       | Description                                    |
| -------------- | --------------------------- | ---------------------------------------------- |
| File accessed  | /etc/shadow                 | Contains hashed passwords for all system users |
| File accessed  | /etc/sudoers                | Contains sudo privilege configuration          |
| Search pattern | *.pem                       | Targeting SSH private key files                |
| Actor          | ubuntu (via sudo)           | Legitimate account used for malicious actions  |
| Time window    | 11:00:40 — 11:02:03 UTC     | 2-minute systematic credential harvest         |
| Instance       | ip-10-0-15-138 (Endpoint A) | Source host of all actions                     |

---

## Impact Assessment

| File         | Sensitivity | Potential Impact                                                |
| ------------ | ----------- | --------------------------------------------------------------- |
| /etc/shadow  | CRITICAL    | Password hashes extractable for offline cracking                |
| /etc/sudoers | HIGH        | Reveals privilege structure, identifies targets for escalation  |
| *.pem files  | CRITICAL    | SSH private keys enable passwordless access to other systems    |
| /etc/passwd  | MEDIUM      | User enumeration — low sensitivity but enables targeted attacks |

In this lab environment, the actual risk is contained. In a production environment, access to `/etc/shadow` combined with offline password cracking tools (hashcat, John the Ripper) could compromise all system accounts. Discovery of `.pem` files could enable lateral movement to any system those keys authenticate against.

---

## Root Cause Analysis

This incident is a continuation of IR-002. After establishing a backdoor account, the threat actor performed standard post-exploitation reconnaissance:

1. **User enumeration** (`/etc/passwd`): identify all accounts on the system
2. **Credential harvesting** (`/etc/shadow`): extract password hashes for offline cracking
3. **Key discovery** (`find *.pem`): locate SSH private keys for lateral movement
4. **Privilege mapping** (`/etc/sudoers`): understand the privilege landscape

This sequence follows the MITRE ATT&CK framework:
- **T1003.008** — OS Credential Dumping: /etc/passwd and /etc/shadow
- **T1552.004** — Unsecured Credentials: Private Keys
- **T1069.001** — Permission Groups Discovery: Local Groups

---

## Response Actions

| Action                                  | Status | Details                                         |
| --------------------------------------- | ------ | ----------------------------------------------- |
| Confirmed access via audit logs         | Done   | sudo log entries with exact commands            |
| Assessed data exposure                  | Done   | Shadow file viewed but no exfiltration detected |
| Reviewed for .pem files present         | Done   | lab-key-pair.pem present on system              |
| CloudWatch Alarm configured             | Done   | Triggers on any /etc/shadow access via sudo     |
| Rotated potentially exposed credentials | Done   | Lab key pair noted for rotation                 |
| Backdoor account removed                | Done   | Addressed in IR-002                             |

---

## Recommendations

1. **Restrict /etc/shadow access**: Only root should access shadow file. Implement mandatory access controls (AppArmor/SELinux) to alert or block unauthorized shadow reads.
2. **Never store .pem files on servers**: Private keys should never be present on the systems they authenticate. Use AWS Secrets Manager or Systems Manager Parameter Store for key management.
3. **Implement file integrity monitoring (FIM)**: Tools like AIDE or auditd file watches can detect access to sensitive files in real time rather than requiring log analysis.
4. **Harden sudoers configuration**: Replace `NOPASSWD: ALL` with specific command whitelisting. The ubuntu user should not have unrestricted sudo access in production.
5. **Enable auditd for enhanced forensics**: The Linux audit daemon provides more granular file access monitoring than auth.log alone, including read-only file access that doesn't generate sudo entries.
6. **Network segmentation**: Endpoints should not have unrestricted intra-VPC SSH access. Use a bastion host with jump server configuration and log all sessions.

---

## Correlation with Other Incidents

This incident is directly correlated with IR-002 (Unauthorized User Creation):
- Same source instance (ip-10-0-15-138 / Endpoint A)
- Sequential timeline — credential harvesting followed persistence
- Same actor (ubuntu user with sudo)
- Combined, these incidents represent a complete attack chain: **Initial Access → Persistence → Credential Access → Potential Lateral Movement**

---

## Lessons Learned

- Sensitive file access via sudo generates clean, parseable log entries — `COMMAND=/usr/bin/cat /etc/shadow` is unambiguous in forensic context.
- A 2-minute window of high-privilege file accesses is a strong behavioral indicator of post-exploitation activity, even without specific file content being logged.
- CloudWatch Logs Insights allows correlation across multiple log entries to reconstruct an attack timeline without a dedicated SIEM.
- The MITRE ATT&CK framework provides a structured vocabulary for describing attacker behavior — using it in incident reports makes them readable and actionable for any security team.

---

# Fase 3 — Estado final

## Resumen de incidentes detectados

| Incident                            | Severity | Detection Method         | Response                                |
| ----------------------------------- | -------- | ------------------------ | --------------------------------------- |
| IR-001 — SSH Brute Force            | HIGH     | CloudWatch query + Alarm | Confirmed no breach, alarm configured   |
| IR-002 — Unauthorized User Creation | CRITICAL | CloudWatch query + Alarm | Backdoor account deleted                |
| IR-003 — Sensitive File Access      | CRITICAL | CloudWatch query + Alarm | Exposure assessed, controls recommended |

## Costo acumulado

**Valor calculado por AWS Billing**: USD $3.32  
**Costo real abonado**: USD $0.00

El monto de $3.32 corresponde al valor calculado por AWS para el uso de recursos durante el proyecto (principalmente transferencia de datos y métricas de CloudWatch). Este monto queda completamente cubierto por el AWS Free Tier y no genera ningún cargo en la tarjeta asociada a la cuenta. AWS muestra el valor calculado en el dashboard de billing como referencia, pero el cargo efectivo es $0.

Esto confirma que el proyecto puede ejecutarse íntegramente dentro del Free Tier si se siguen las restricciones documentadas: instancias apagadas cuando no se trabaja activamente, sin NAT Gateway, y sin Elastic IPs huérfanas.

## Capacidades demostradas

- ✅ Deployment y configuración de CloudTrail para audit logging
- ✅ Instalación y configuración de CloudWatch Agent en múltiples endpoints
- ✅ Escritura de detection queries en CloudWatch Logs Insights
- ✅ Configuración de Metric Filters y Alarms para detección automatizada
- ✅ Producción de incident reports profesionales con timeline, IoCs, análisis de causa raíz y recomendaciones
- ✅ Aplicación del framework MITRE ATT&CK para clasificación de técnicas
- ✅ Incident response: contención, erradicación y recomendaciones de mejora