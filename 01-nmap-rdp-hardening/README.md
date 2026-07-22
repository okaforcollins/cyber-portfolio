# Attack Surface Assessment & RDP Hardening — Azure Windows VM

**Project 01 · Cloud Security Lab**
*Service enumeration on a self-built Azure VM, followed by a least-privilege network control to reduce exposure.*

---

## Summary

I built a Windows virtual machine in Microsoft Azure, enumerated its exposed network services with Nmap, interpreted the results from a defender's perspective, and remediated the highest-risk finding by restricting Remote Desktop access to a single source IP.

**Key outcome:** RDP (3389) was reachable from the entire internet. I reduced that exposure to one known source address, applying the principle of least privilege to network access.

---

## Environment

| Component | Detail |
|---|---|
| Platform | Microsoft Azure |
| VM name | `win-lab` |
| Resource group | `rg-soc-lab` |
| Size | Standard B2ls v2 (2 vCPU, 4 GB RAM) |
| OS | Windows |
| Region | Sweden Central |
| Tooling | Nmap 7.99 |

All systems in this exercise were built and owned by me. No third-party or production systems were scanned at any point.

---

## Objective

1. Determine which network services the VM exposes.
2. Assess each service against what a defender would expect and accept.
3. Remediate any unnecessary internet-facing exposure.

---

## Method

Service and version detection was run against the host:

```bash
nmap -sV 127.0.0.1
```

Output was written to file for evidence:

```bash
nmap -sV 127.0.0.1 -oN nmap-scan.txt
```

---

## Results

```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-16 13:48 +0000
Nmap scan report for localhost (127.0.0.1)
Host is up (0.0011s latency).
Not shown: 997 closed tcp ports (reset)
PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
445/tcp  open  microsoft-ds?
3389/tcp open  ms-wbt-server
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port3389-TCP:V=7.99%I=7%D=7/16%Time=6A58E142%P=i686-pc-windows-windows%
SF:r(TerminalServerCookie,13,"\x03\0\0\x13\x0e\xd0\0\0\x124\0\x02/\x08\0\x
SF:02\0\0\0");
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.18 seconds
```

**997 of 1,000 common TCP ports were closed**, indicating a modest default attack surface. Three services were listening.

---

## Analysis

| Port | Service | Assessment |
|---|---|---|
| **135/tcp** | MSRPC | Expected on Windows. Used for inter-process communication over the network. Should never be internet-facing — internal use only. |
| **445/tcp** | SMB | Expected on Windows (file/print sharing). Historically high-risk: this is the service exploited by EternalBlue and used by WannaCry in 2017, which significantly disrupted NHS services. Must not be exposed publicly. |
| **3389/tcp** | RDP | Expected — this is my management access path. However, it had been configured as reachable from **any** source address during VM creation. Internet-exposed RDP is routinely targeted by automated credential-stuffing and brute-force campaigns, and is a common ransomware entry vector. **This was the finding I prioritised.** |

Nmap could not fingerprint the exact RDP version but correctly identified the host OS as Windows via its response signature.

### Risk prioritisation

RDP was treated as the primary finding rather than SMB or RPC, because:

- it was confirmed **publicly reachable** by design (an inbound allow rule from `Any`), whereas 135 and 445 were not intentionally published;
- it is a **management plane** service — compromise yields interactive control of the host;
- it is among the most actively scanned ports on the internet.

---

## Remediation

The exposure was addressed at the network layer rather than on the host, so that unwanted traffic is dropped before it reaches the VM.

**Control applied:** Network Security Group inbound rule for port 3389 amended — `Source` changed from `Any` to a single permitted IP address.

**Steps:**
1. Azure Portal → `win-lab` → **Networking** → **Network settings**
2. Selected the inbound rule for port **3389** (`RDP`)
3. Changed **Source** from `Any` to **My IP address**
4. Saved and re-tested the RDP session to confirm legitimate access was retained

**Result:** RDP is now reachable only from an approved source. All other internet traffic to 3389 is denied at the NSG.

**Verification:** Connectivity was re-tested post-change and the session established successfully, confirming the rule was correctly scoped and did not break intended access.

---

## Outcome

| | Before | After |
|---|---|---|
| RDP (3389) source | Any (0.0.0.0/0) | Single approved IP |
| Internet-facing management ports | 1 | 0 |

---

## What I learned

- **Reading a scan is the skill, not running one.** The output takes seconds; deciding which of three "normal" services actually represents risk is the analytical work.
- **Context determines severity.** All three ports are expected on Windows. What made 3389 the finding was not the service itself but its *exposure* — the same port is unremarkable when scoped correctly.
- **Least privilege applies to networks, not just identities.** Restricting source addresses is the network equivalent of granting only the access required.
- **Defaults are permissive for convenience.** The VM creation wizard offered internet-wide RDP as a default path and warned against it. Secure configuration is an active decision, not an outcome of accepting defaults.
- **Precision matters in tooling.** An initial attempt using `-sv` rather than `-sV` returned the help output instead of a scan — a reminder that security tools are case-sensitive and that verifying you got the output you expected is part of the process.

---

## Next steps

- Deploy Microsoft Sentinel and ingest this VM's security event logs
- Build a detection rule for failed RDP authentication attempts against this host
- Consider Azure Bastion or a just-in-time access policy as a stronger alternative to a source-restricted NSG rule

---

## Ethics & scope

All activity was conducted against infrastructure I personally provisioned and own, within my own Azure subscription. No third-party, employer, or production systems were scanned. Unauthorised scanning of systems is both a violation of Azure's terms of service and, in the UK, an offence under the Computer Misuse Act 1990.

---

*Part of my transition into cloud security and SOC analysis. Background: healthcare support worker with NHS Scotland, handling sensitive data under GDPR and the Caldicott Principles. Certified: CompTIA Security+, ISC2 CC.*
