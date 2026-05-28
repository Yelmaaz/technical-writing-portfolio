# Unauthorised SSH Access and Data Exfiltration

**VaultPay Financial Services — Internal Security Document**

> **Disclaimer:** This document is a fictional sample created for portfolio and demonstration purposes only. VaultPay Financial Services is a fictional organisation. All incidents, individuals, systems, and data referenced in this document are entirely invented. This postmortem is intended to demonstrate technical writing competency in security operations documentation and does not represent a real breach or real organisation.

| Field              | Detail                       |
| ------------------ | ---------------------------- |
| Incident ID        | INC-2025-0847                |
| Severity           | P1 — Critical                |
| Status             | Resolved (open action items) |
| Incident Start     | 2025-09-12 02:14 UTC         |
| Incident Detected  | 2025-09-12 07:43 UTC         |
| Incident Contained | 2025-09-12 11:22 UTC         |
| Total Duration     | 9 hours 8 minutes            |
| Detection Lag      | 5 hours 29 minutes           |
| Author             | Security Operations Team     |
| Review Date        | 2025-09-19                   |
| Distribution       | Internal — Restricted        |

---

## Executive Summary

On 12 September 2025, an unauthorised actor gained access to VaultPay's transaction processing server (`txproc-prod-03`) via SSH using a compromised service account credential. The actor initially established a foothold by compromising a developer workstation through a spearphishing attack, then moved laterally across the internal network to reach the production server. The actor maintained persistent access for approximately five and a half hours before detection, during which time an estimated 218,000 customer transaction records were exfiltrated to an external host. Containment was achieved by 11:22 UTC following isolation of the affected host and credential revocation across all environments.

The root cause was a combination of a successful spearphishing attack against a developer workstation, a reused and non-rotated service account credential stored in plaintext on an internal file server, the absence of multi-factor authentication on SSH access for service accounts, and insufficient alerting on anomalous internal login activity. No ransomware or destructive payloads were deployed. The affected data set included transaction metadata, partial card numbers (last four digits only), and customer identifiers. Full card numbers and authentication credentials were not exfiltrated due to field-level encryption at rest.

This postmortem is blameless. Its purpose is to identify systemic failures and implement controls that prevent recurrence.

---

## Background

VaultPay operates a microservices-based transaction processing platform hosted across three Linux-based production servers. SSH access to production hosts is restricted by IP allowlist and requires key-based authentication for human operators. Service accounts used by internal automation pipelines were, at the time of the incident, permitted to authenticate via password over SSH from within the internal network range.

`txproc-prod-03` handles batch settlement processing and communicates with downstream banking partners via an encrypted API channel. It holds a rolling 90-day window of transaction records in a local PostgreSQL instance before archival.

Development configuration files, including environment files containing service account credentials, were stored on a shared internal file server accessible to members of the engineering team. At the time of the incident, access controls on this file server were not enforced at the individual file level.

---

## Timeline

All times in UTC.

| Time             | Event                                                                                                                                                                                                                                                                                                                                                                  |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2025-09-10 09:23 | A developer receives a spearphishing email containing a malicious attachment crafted to appear as an internal HR document. The attachment is opened, executing a staged payload that establishes a reverse shell on the developer workstation. The compromise is not detected at the time.                                                                             |
| 2025-09-10 09:31 | Actor begins internal reconnaissance from the compromised workstation. Network scanning identifies the internal file server.                                                                                                                                                                                                                                           |
| 2025-09-10 11:14 | Actor accesses the internal file server from the compromised workstation and locates development configuration files. A copy of the deployment environment file containing plaintext credentials for the `svc-settlement` service account is harvested. The credential has not been rotated in 14 months and is shared across development and production environments. |
| 2025-09-11 23:47 | Automated deployment script runs on `txproc-prod-03`. A configuration file containing plaintext service account credentials is written to `/etc/deploy/config.env` with world-readable permissions due to a misconfigured deployment step.                                                                                                                             |
| 2025-09-12 02:14 | Actor initiates SSH session to `txproc-prod-03` using the harvested `svc-settlement` credential from the compromised developer workstation. The connection originates from within the internal network range and does not trigger the IP allowlist check. Login proceeds without triggering alerts.                                                                    |
| 02:19            | Actor runs reconnaissance commands: `whoami`, `id`, `uname -a`, `ps aux`, `ss -antp`.                                                                                                                                                                                                                                                                                  |
| 02:31            | Actor accesses PostgreSQL instance using credentials stored in `/etc/deploy/config.env`. Begins querying transaction tables.                                                                                                                                                                                                                                           |
| 03:04            | First outbound data transfer detected in network logs (not alerted at time of occurrence). Destination: external host over port 443. Transfer volume: 1.2 GB.                                                                                                                                                                                                          |
| 03:04 — 07:38    | Actor conducts four additional exfiltration transfers totalling approximately 4.8 GB. SSH session remains active throughout.                                                                                                                                                                                                                                           |
| 07:38            | Actor terminates SSH session.                                                                                                                                                                                                                                                                                                                                          |
| 07:43            | On-call SOC analyst notices anomalous entry in daily login summary report during routine morning review. Service account login outside of expected automation windows flagged for review. Incident declared.                                                                                                                                                           |
| 07:51            | Incident response initiated. `txproc-prod-03` isolated from network. Compromised developer workstation identified and isolated.                                                                                                                                                                                                                                        |
| 08:15            | Credential revocation begins across all environments for `svc-settlement` and all related service accounts.                                                                                                                                                                                                                                                            |
| 08:44            | Forensic images of `txproc-prod-03` and the compromised developer workstation captured for evidence preservation.                                                                                                                                                                                                                                                      |
| 09:30            | Scope of exfiltration estimated from PostgreSQL query logs and network transfer volumes.                                                                                                                                                                                                                                                                               |
| 10:10            | Legal and compliance teams notified. Regulatory disclosure obligations assessed.                                                                                                                                                                                                                                                                                       |
| 11:22            | Containment confirmed. No further unauthorised access detected.                                                                                                                                                                                                                                                                                                        |
| 2025-09-13 09:00 | Preliminary breach notification submitted to relevant financial regulatory authority.                                                                                                                                                                                                                                                                                  |

---

## Root Cause Analysis

Three distinct failures combined to enable this incident. No single control failure was solely responsible.

### Primary Cause: Lateral Movement via Compromised Developer Workstation

On 10 September 2025, a developer workstation was compromised through a spearphishing email containing a malicious attachment. The attachment executed a staged payload that established a reverse shell, giving the actor an internal foothold within VaultPay's network. This initial compromise was not detected at the time.

From the compromised workstation, the actor accessed a shared internal file server where development configuration files were stored for team reference. Access controls on the file server were not enforced at the individual file level, allowing any authenticated internal user or process to read all files in the shared directory. Among these files was a deployment environment file containing plaintext credentials for the `svc-settlement` service account. The credential had not been rotated in 14 months and was shared across development and production environments without isolation.

Using the harvested credential, the actor moved laterally from the compromised workstation to `txproc-prod-03` over SSH. Because the connection originated from within the internal network range, it did not trigger the IP allowlist check applied to connections from external sources.

### Contributing Cause: No Multi-Factor Authentication on Service Account SSH

SSH access policy required MFA for human operator accounts but explicitly exempted service accounts on the basis that interactive MFA was operationally impractical for automated pipelines. This exemption was undocumented and had not been reviewed since the policy was originally written. It created a gap: any actor with valid service account credentials could authenticate directly over SSH without a second factor, regardless of the originating host.

### Contributing Cause: Insufficient Alerting on Anomalous Login Behaviour

Alerting rules were configured to flag failed authentication attempts and logins from IPs outside the allowlist. Because the actor connected from a compromised internal host, the login fell within the trusted IP range and produced no real-time alert. No behavioural baseline existed for service account SSH activity. An off-hours login by a service account outside of its expected automation window was logged but not flagged. The actor maintained access for over five hours before the anomaly was identified through a manual morning review of the daily login summary report.

---

## Impact Assessment

|Category|Detail|
|---|---|
|Records affected|Approximately 218,000 customer transaction records|
|Data types exfiltrated|Transaction metadata, customer identifiers, last four digits of payment cards, transaction timestamps and amounts|
|Data confirmed not exfiltrated|Full card numbers (field-level encrypted), authentication credentials, account passwords|
|Systems affected|`txproc-prod-03` (isolated); developer workstation (isolated); internal file server (access audit in progress)|
|Service disruption|Batch settlement processing delayed by 3 hours 41 minutes during containment|
|Regulatory impact|Mandatory breach notification filed; regulatory review ongoing|
|Customer impact|Notification letters issued to all 218,000 affected customers within 72 hours|

---

## What Went Well

- The forensic images of both `txproc-prod-03` and the compromised developer workstation were captured within 57 minutes of incident declaration, preserving a clean evidence trail before any remediation steps altered the system state.
- Credential revocation across all environments was completed within 33 minutes of initiation, limiting the window for further exploitation using the same credentials.
- The incident response runbook was followed without deviation. Escalation paths functioned correctly and legal and compliance teams were engaged within the response window required by internal policy.
- Field-level encryption on sensitive card data limited the impact of the exfiltration. Full card numbers were not accessible to the actor even with database access.
- Inter-team communication during the response was clear and well-documented in the incident channel. The timeline above was reconstructed with high confidence from logs preserved during the response.

---

## What Went Wrong

- The spearphishing email was not caught by existing email security controls. No security awareness training had been conducted in the preceding 12 months.
- The compromised developer workstation showed no signs of detection for over 40 hours. Endpoint detection and response tooling was not deployed on developer workstations at the time of the incident.
- Credentials were not rotated on a defined schedule. A 14-month-old credential in active use represents a systemic credential hygiene failure, not a one-off oversight.
- The same credential was shared across development and production environments. Credential isolation between environments was not enforced by policy or tooling.
- Access controls on the internal file server were not enforced at the file level. Any authenticated internal user or process could read sensitive configuration files.
- Service account SSH exemptions from MFA were undocumented and had not been reviewed since the policy was originally written.
- Real-time alerting did not account for behavioural anomalies in service account activity. A service account authenticating outside of its expected automation window produced no alert.
- Detection depended on a manual review of a daily summary report. This is not a scalable or reliable detection mechanism for a P1 incident class.

---

## Action Items

|ID|Action|Owner|Priority|Due Date|
|---|---|---|---|---|
|ACT-01|Deploy endpoint detection and response tooling across all developer workstations|Security Engineering|P1|2025-09-26|
|ACT-02|Enforce mandatory credential rotation every 90 days for all service accounts via automated policy|Identity and Access Management|P1|2025-09-26|
|ACT-03|Enforce credential isolation between development and production environments; revoke all shared credentials immediately|Identity and Access Management|P1|2025-09-26|
|ACT-04|Implement file-level access controls on the internal file server; audit current permissions across all shared directories|Security Operations|P1|2025-09-26|
|ACT-05|Implement real-time alerting for service account SSH logins outside of expected automation windows|Security Operations|P1|2025-09-26|
|ACT-06|Conduct mandatory phishing awareness training for all engineering staff|People and Security|P1|2025-10-04|
|ACT-07|Review and document all MFA exemptions for service accounts; implement certificate-based SSH authentication to replace password-based service account access|Identity and Access Management|P2|2025-10-10|
|ACT-08|Add permission validation step to all deployment pipelines to reject world-readable sensitive file outputs|DevOps|P2|2025-10-10|
|ACT-09|Conduct organisation-wide audit of configuration files for plaintext credential storage|Security Operations|P2|2025-10-10|
|ACT-10|Implement automated detection for large outbound data transfers to external hosts during off-hours|Security Engineering|P2|2025-10-17|
|ACT-11|Conduct tabletop exercise based on this incident scenario with all incident response stakeholders|Security Operations|P3|2025-11-08|

---

## Open Questions

The following questions remain unresolved pending further investigation:

- **Full scope of workstation compromise:** Forensic analysis of the compromised developer workstation is ongoing. It has not yet been confirmed whether the actor accessed additional internal systems or harvested further credentials beyond those used in this incident.
- **File server access scope:** An audit of access logs on the internal file server is in progress to determine whether the actor accessed files beyond the deployment configuration, and whether any other sensitive data was exposed.
- **Actor attribution:** No threat intelligence cluster has been definitively linked to this activity pattern. The spearphishing email origin is under analysis. Law enforcement has been notified and attribution analysis is ongoing.
- **Scope confirmation:** The 218,000 record estimate is based on PostgreSQL query logs and network transfer volumes. A full record-level audit is in progress to confirm exact scope and identify whether any records outside the transaction table were accessed.

---

## Lessons Learned

This incident was preventable at multiple points. The exfiltration succeeded not because of a sophisticated attack but because a set of individually manageable risks were left unaddressed in combination.

The initial entry point was a developer workstation with no endpoint detection tooling and a user who had not received recent security awareness training. Once inside the network, the actor encountered a file server with no file-level access controls, a credential that had not been rotated in over a year, and a production environment that accepted the same credential used in development. The lateral movement from workstation to production server was undetected because alerting was designed around external threats, not internal ones.

The five-and-a-half-hour detection lag is the most operationally significant failure in this timeline. It is not acceptable for a P1-class incident to be detected through a manual morning review. Behavioural baselining for service accounts, endpoint visibility on developer machines, and real-time alerting on anomalous internal activity are the three controls that would have shortened this lag materially. The action items above prioritise closing these gaps before addressing secondary contributing factors.

---

## Document Control

|Version|Date|Author|Notes|
|---|---|---|---|
|0.1|2025-09-14|Security Operations|Initial draft|
|0.2|2025-09-17|Security Operations|Timeline and root cause revised following forensic review of developer workstation|
|1.0|2025-09-19|Security Operations|Final version approved for internal distribution|
