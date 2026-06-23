# CloudOps Helpdesk + Security Lab

End-to-end IT Support and Security Operations lab deployed on AWS Free Tier. Built to demonstrate practical skills in identity management, helpdesk operations, cloud infrastructure, and security incident detection — targeting Jr. IT Support Specialist and SOC Analyst Trainee roles.

**Total cost**: USD $0.00 (AWS Free Tier)  
**Duration**: 14 days  
**Status**: ✅ Complete

---

## What this project demonstrates

| Skill Area | What I built |
|---|---|
| **Cloud Infrastructure** | VPC, subnets, IGW, security groups, EC2 instances on AWS |
| **Identity & Access Management** | AWS IAM Identity Center with users, groups, MFA, and permission sets |
| **IT Support Operations** | osTicket helpdesk — SLAs, departments, custom forms, ticket lifecycle |
| **Email Integration** | Amazon SES configuration, SMTP verification, email delivery testing |
| **Security Monitoring** | CloudTrail audit logging, CloudWatch Agent, log aggregation |
| **Threat Detection** | CloudWatch Logs Insights queries for brute force, privilege escalation, credential access |
| **Incident Response** | 3 complete incident reports with MITRE ATT&CK mapping, IoCs, and remediation |
| **Documentation** | Technical writeups, architecture decisions, lessons learned |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   AWS Account (Free Tier)                   │
│                      us-east-1                              │
│                                                             │
│  ┌──────────────────┐         ┌──────────────────────┐     │
│  │  Identity Layer  │         │   Helpdesk App        │     │
│  │  IAM Identity    │────────▶│   osTicket v1.18.1    │     │
│  │  Center          │         │   EC2 t2.micro        │     │
│  │  3 users, groups │         │   Ubuntu 24.04        │     │
│  │  MFA enforced    │         │   Apache + PHP + MariaDB    │
│  └──────────────────┘         └──────────────────────┘     │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐                        │
│  │  Endpoint A  │  │  Endpoint B  │                        │
│  │  EC2 t2.micro│  │  EC2 t2.micro│                        │
│  │  CW Agent    │  │  CW Agent    │                        │
│  │  Attack host │  │  Target host │                        │
│  └──────┬───────┘  └──────┬───────┘                        │
│         │                 │                                 │
│  ┌──────▼─────────────────▼──────────────────────────┐     │
│  │              Security & Monitoring                  │     │
│  │  CloudTrail · CloudWatch Logs · Alarms · SNS       │     │
│  └────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

**Network**: Custom VPC (10.0.0.0/16) · 2 public subnets · Internet Gateway  
**Security**: Security Groups with least-privilege rules · SSH key-based auth only

---

## Tech Stack

**Cloud**: AWS (EC2, VPC, IAM Identity Center, CloudTrail, CloudWatch, SES, S3, SNS)  
**OS**: Ubuntu Server 24.04 LTS  
**Helpdesk**: osTicket v1.18.1  
**Web Stack**: Apache 2 · PHP 8.5 · MariaDB  
**Monitoring**: Amazon CloudWatch Agent · CloudWatch Logs Insights  
**Security Tools**: Hydra (attack simulation) · swaks (SMTP testing)  
**Documentation**: Markdown · GitHub

---

## Project Structure

```
cloudops-helpdesk-security-lab/
├── README.md
├── docs/
│   ├── 01-aws-fundamentals.md        # Phase 1: Account setup, VPC, IAM Identity Center
│   ├── 02-helpdesk-deployment.md     # Phase 2: osTicket deploy, config, email, operations
│   └── 03-security-operations.md    # Phase 3: CloudTrail, monitoring, incidents
│       ├── IR-001-ssh-brute-force
│       ├── IR-002-unauthorized-user-creation
│       └── IR-003-sensitive-file-access
└── PROJECT_ROADMAP.md                # Full project plan and scope
```

---

## Phases

### Phase 1 — AWS Fundamentals (Days 1-3)
- Secured AWS account with MFA on root and IAM admin user
- Configured billing alerts (Budgets + CloudWatch Alarms) — total cost $0
- Built custom VPC with public subnets, Internet Gateway, and security groups
- Deployed AWS IAM Identity Center with 3 users, 3 groups, permission sets, and MFA

**Key decision**: Chose IAM Identity Center over traditional Active Directory — cloud-native, no servers to maintain, reflects the real stack of US remote-first companies.

📄 [Phase 1 Documentation](docs/01-aws-fundamentals.md)

---

### Phase 2 — Helpdesk Deployment (Days 4-7)
- Manually deployed osTicket on EC2: Ubuntu + Apache + PHP + MariaDB
- Configured departments (IT Support, Security, HR), SLAs (Critical 2hr to Low 72hr), Help Topics with auto-routing, and custom Access Request form
- Verified Amazon SES integration via SMTP — email delivery confirmed via swaks
- Simulated and resolved 10 diverse tickets across all categories

**10 tickets resolved, covering**:
- 2 Security Incidents (phishing, unauthorized login)
- 2 Account Lockouts
- 2 Access Requests (with manager approval verification)
- 2 Hardware Issues (remote diagnosis + replacement request)
- 2 General Inquiries (VPN setup, license upgrade)

📄 [Phase 2 Documentation](docs/02-helpdesk-deployment.md)

---

### Phase 3 — Security Operations (Days 8-11)
- Enabled CloudTrail with multi-region logging to S3 + CloudWatch
- Deployed CloudWatch Agent on 2 endpoints — collecting auth.log and syslog
- Simulated 3 security incidents using Hydra and manual privilege escalation
- Wrote CloudWatch Logs Insights detection queries for each incident type
- Configured 3 CloudWatch Alarms with SNS notifications
- Produced 3 professional incident reports with MITRE ATT&CK mapping

**Incidents detected and documented**:

| ID | Incident | Severity | MITRE ATT&CK |
|---|---|---|---|
| IR-001 | SSH Brute Force Attack | HIGH | T1110.001 |
| IR-002 | Unauthorized User Creation & Privilege Escalation | CRITICAL | T1136.001, T1548 |
| IR-003 | Sensitive File Access & Credential Harvesting | CRITICAL | T1003.008, T1552.004 |

📄 [Phase 3 Documentation + Incident Reports](docs/03-security-operations.md)

---

## Key Learnings

**Technical**
- AWS Free Tier is sufficient for a production-grade lab environment when resources are managed carefully (stopped when not in use)
- CloudWatch Logs Insights can replicate basic SIEM functionality without the cost — detection queries for brute force, privilege escalation, and credential access are achievable with standard log patterns
- The `[preauth]` SSH log pattern is a reliable brute force indicator; `useradd` log entries provide clean forensic evidence for persistence detection

**Operational**
- Documenting *why* a decision was made matters as much as *what* was done — architecture decisions with justifications make documentation useful, not just descriptive
- When a UI blocks a configuration (SES-osTicket integration), verifying functionality at the infrastructure layer (swaks) and documenting the limitation honestly demonstrates more maturity than forcing a workaround
- MITRE ATT&CK provides a shared vocabulary that makes incident reports readable across any security team

**Security**
- Password authentication on SSH should always be disabled — key-based auth is the baseline
- Storing `.pem` files on servers is a critical misconfiguration — private keys should live in secrets management systems
- Three incidents from a single compromised account (brute force → persistence → credential harvest) illustrate why lateral movement is rapid once initial access is achieved

---

## Known Limitations


osTicket v1.18.1 — SMTP UI bug: the Outgoing SMTP configuration panel has a cross-validation issue with the Remote Mailbox section that prevents entering credentials through the UI. Email delivery was verified at the infrastructure layer using swaks (Swiss Army Knife for SMTP) directly against Amazon SES — authentication and delivery confirmed successful. Tickets were created manually through the web UI as a workaround.
php-imap unavailable on Ubuntu 24.04: the php-imap package was removed from the main Ubuntu 24.04 repository. This means automatic ticket creation via email (IMAP polling) is not implemented. This does not affect the core helpdesk functionality demonstrated in this lab.
AWS Billing note: AWS Billing showed a calculated usage value of $3.32 during the project. This amount is fully covered by the Free Tier and resulted in $0.00 actual charges. It reflects the billing calculation methodology AWS uses for metered services even within free tier limits.



Technical Deep Dives

For detailed explanations of what each command does and why — written to be useful in interview preparation:


📄 Day 4 — LAMP Stack Explained: step-by-step breakdown of every command run during the osTicket deployment, including what Apache, PHP, and MariaDB are doing under the hood, and how the request-response cycle works.

---

## Cost Analysis

| Service | Usage | Cost |
|---|---|---|
| EC2 (3x t2.micro) | Stopped when not in use — ~4 hrs/day | $0.00 |
| S3 (CloudTrail logs) | < 5 GB | $0.00 |
| CloudWatch Logs | < 5 GB ingestion | $0.00 |
| CloudTrail | First trail free | $0.00 |
| IAM Identity Center | Free | $0.00 |
| SES | < 200 emails/day in sandbox | $0.00 |
| **Total** | | **$0.00** |

---

## About

Built by **Martín Salvo** — Systems Analyst (ORT Argentina, 2025), CompTIA Security+ certified (May 2026), ISC2 CC certified (May 2026).

Currently seeking Jr. IT Support Specialist and Security-adjacent roles in Buenos Aires and remote international positions.

📧 martin.salvo616@gmail.com  
🔗 [LinkedIn](https://www.linkedin.com/in/martinsalvo616)
