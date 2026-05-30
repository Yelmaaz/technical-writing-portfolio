
**Company:** Northlane  
**Document owner:** Engineering Operations / Site Reliability  
**Version:** 2.1  
**Last updated:** May 2026  
**Applies to:** All on-call engineers, incident commanders, and CS leads

## Table of Contents

1. [Purpose and Scope](#purpose-and-scope)
2. [Incident Severity Definitions](#incident-severity-definitions)
3. [Tool Stack](#tool-stack)
4. [Detection and Initial Response](#detection-and-initial-response)
5. [Incident Roles](#incident-roles)
6. [Escalation Matrix](#escalation-matrix)
7. [Communication Protocols](#communication-protocols)
8. [Resolution and Recovery](#resolution-and-recovery)
9. [Client Communication Templates](#client-communication-templates)
10. [Postmortem Process](#postmortem-process)
11. [Postmortem Template](#postmortem-template)
12. [Lessons Learned Workflow](#lessons-learned-workflow)

## Purpose and Scope
This playbook defines Northlane’s incident response procedures for production events affecting payout routing, disbursement scheduling, and transaction reconciliation services.
Northlane is a fintech infrastructure platform that manages payout routing, disbursement scheduling, and transaction reconciliation for marketplace businesses operating across multiple payment providers and regional banking systems. Production incidents at Northlane have direct financial consequences for marketplace operators and their end users. This playbook exists to ensure incidents are detected quickly, contained effectively, communicated clearly, and resolved without recurrence.
**What this playbook covers:**
- Incident severity classification
- Detection, triage, and escalation procedures
- Role definitions and responsibilities during active incidents
- Internal and external communication protocols
- Resolution and recovery workflows
- Postmortem process and documentation requirements
**What this playbook does not cover:**
- Routine operational alerts that do not meet SEV-3 threshold (see: Alerting Runbook)
- Planned maintenance windows (see: Maintenance Window Procedure)
- Security incident response (see: Security Incident Response Playbook)
- Vendor outage communication (see: Third-Party Provider Escalation Guide)

## Incident Severity Definitions
All production events are classified by severity at the time of declaration. Severity determines escalation requirements, communication cadence, and postmortem obligations.

| Severity | Description                                                    | Example                                                        | Postmortem required         |
| -------- | -------------------------------------------------------------- | -------------------------------------------------------------- | --------------------------- |
| SEV-1    | Major platform outage with direct financial impact             | Disbursement queue stalled globally, payouts not processing    | Yes, within 48 hours        |
| SEV-2    | Partial degradation affecting a subset of clients or functions | Reconciliation reporting delayed, single-region payout latency | Yes, within 5 business days |
| SEV-3    | Minor operational issue with no direct client financial impact | Dashboard latency spike, non-critical API slowness             | Optional, at IC discretion  |
| SEV-4    | Informational alert, no user impact                            | Elevated error rate below threshold, infrastructure warning    | No                          |
### Severity escalation
If the incident's impact expands, immediately upgrade the severity and notify the next escalation contact per the matrix. Never reduce severity during the active incident. Reclassify only after full review during postmortem. If uncertain, ask the Engineering Manager or VP Engineering for guidance.
When unsure, escalate to a higher severity immediately following the matrix. Overclassifying allows for rapid response, while underclassifying can delay action. Always err on the side of escalation, and communicate your decision to the appropriate contacts.

## Tool Stack

| Tool               | Purpose                                             | On-call access      |
| ------------------ | --------------------------------------------------- | ------------------- |
| PagerDuty          | Alerting, on-call rotation, incident escalation     | Required            |
| Slack              | Incident coordination, real-time communication      | Required            |
| Datadog            | Infrastructure monitoring, log analysis, dashboards | Required            |
| Linear             | Task tracking, action item assignment               | Required            |
| Statuspage         | External incident communication to clients          | IC only             |
| Runbook Repository | Internal procedure documentation                    | Read access for all |
| Zoom               | Incident bridge calls for SEV-1 and SEV-2           | Required            |
All on-call engineers must have PagerDuty and Datadog mobile access configured before their first on-call rotation. Access issues must be reported to Engineering Operations before rotation begins.

## Detection and Initial Response
### How incidents are detected
Northlane incidents are typically detected through one of the following channels:
- **Automated alert:** PagerDuty triggers from Datadog monitoring rules
- **Client report:** CS team receives client-reported issue and escalates via `#incidents`
- **Internal discovery:** The engineer identifies the issue during normal operations
All three paths lead to the same initial response workflow.
### Initial response workflow
**Step 1: Acknowledge the alert**
Acknowledge the PagerDuty alert within 5 minutes of trigger. Failure to acknowledge within 5 minutes escalates automatically to the secondary on-call engineer.
**Step 2: Assess and classify**
Open Datadog and assess the issue's scope. Determine:
- What is affected (payout processing, disbursement queue, reconciliation, dashboard, API)
- How many clients or transactions are impacted
- Whether the issue is ongoing or intermittent
Classify severity using the [Incident Severity Definitions](#incident-severity-definitions) table.
**Step 3: Declare the incident**
For SEV-1 and SEV-2, post in `#incidents` immediately using the following format:
```
INCIDENT DECLARED
Severity: [SEV-1 / SEV-2]
Summary: [One sentence description of impact]
IC: [Your name]
Bridge: [Zoom link if applicable]
Datadog: [Dashboard link]
Status: Investigating
```
For SEV-3, post in `#incidents` with a brief note. No bridge call required.
**Step 4: Open the incident channel**
For SEV-1 and SEV-2, create a dedicated Slack channel named `#inc-YYYYMMDD-[short-description]` (for example, `#inc-20260530-payout-queue-stall`). Move all incident coordination to that channel. Keep `#incidents` for declarations and status updates only.

## Incident Roles
### Incident Commander (IC)

The Incident Commander coordinates the response, assigns workstreams, and maintains the timeline. The first on-call engineer to acknowledge the alert is IC, unless another is designated immediately.
**IC responsibilities:**
- Declare the incident and assign severity
- Open the incident channel and bridge call
- Assign investigation and remediation owners
- Maintain communication cadence with stakeholders
- Post status updates at required intervals
- Decide when the incident is resolved
- Initiate the postmortem process
**IC is not responsible for:**
- Directly implementing fixes (unless no other engineer is available)
- Communicating with clients directly (CS Lead handles external communication)
- Approving infrastructure changes unilaterally for SEV-1 (requires second engineer sign-off)
### Investigation Lead
IC assigns the Investigation Lead, who diagnoses the root cause and directs the technical investigation.
**Responsibilities:**
- Identify root cause or contributing factors
- Propose and evaluate remediation options
- Implement fixes or coordinate implementation
- Report findings to IC at regular intervals
### Communications Lead
For SEV-1 and SEV-2, the CS Lead on-call acts as Communications Lead.
**Responsibilities:**
- Draft and post Statuspage updates
- Send client notification emails from templates
- Coordinate with the IC on timing and messaging
- Log all external communications in the incident channel
### Scribe
IC assigns a Scribe for SEV-1 to log the incident timeline in real time.
**Responsibilities:**
- Log all significant developments with timestamps
- Track action items and owners
- Compile the raw timeline for postmortem use

## Escalation Matrix

| Severity | First escalation                               | Second escalation                       | Executive notification                       |
| -------- | ---------------------------------------------- | --------------------------------------- | -------------------------------------------- |
| SEV-1    | On-call engineer + CS Lead (immediate)         | Engineering Manager (within 15 minutes) | VP Engineering + CEO (within 30 minutes)     |
| SEV-2    | On-call engineer + CS Lead (within 15 minutes) | Engineering Manager (within 1 hour)     | VP Engineering (if unresolved after 2 hours) |
| SEV-3    | On-call engineer                               | CS Lead (if client-impacting)           | Not required                                 |
| SEV-4    | On-call engineer                               | Not required                            | Not required                                 |
### PagerDuty escalation policy
PagerDuty escalates automatically if the alert is not acknowledged within 5 minutes. The escalation path is:
1. Primary on-call engineer (0 minutes)
2. Secondary on-call engineer (5 minutes)
3. Engineering Manager (10 minutes)
4. VP Engineering (15 minutes)

## Communication Protocols
### Internal communication
All active incident coordination happens in the dedicated incident channel. The `#incidents` channel is for declarations and status updates only. Do not troubleshoot in `#incidents`.
**Status update cadence:**

| Severity | Update frequency                                            |
| -------- | ----------------------------------------------------------- |
| SEV-1    | Every 15 minutes, or immediately on significant development |
| SEV-2    | Every 30 minutes, or immediately on significant development |
| SEV-3    | At IC discretion, minimum one update per hour               |
Status updates in the incident channel follow this format:
```
STATUS UPDATE [HH:MM UTC]
Status: [Investigating / Identified / Monitoring / Resolved]
Summary: [One sentence on current state]
Next update: [HH:MM UTC or "on next development"]
```
### External communication
External communication is managed exclusively by the Communications Lead via Statuspage and client email. Engineers do not communicate directly with clients during an active incident.
**Statuspage update timing:**

| Severity | Initial post                     | Update cadence   |
| -------- | -------------------------------- | ---------------- |
| SEV-1    | Within 10 minutes of declaration | Every 15 minutes |
| SEV-2    | Within 20 minutes of declaration | Every 30 minutes |
| SEV-3    | At IC discretion                 | At IC discretion |

## Resolution and Recovery
### Declaring resolution
The IC declares resolution when:
- The root cause has been identified
- The fix has been implemented and verified
- Affected metrics have returned to normal baselines in Datadog
- No further client impact is occurring
Do not declare resolution based only on a fix. Verify recovery in Datadog before closing the incident.
### Resolution post in incident channel
```
INCIDENT RESOLVED [HH:MM UTC]
Severity: [SEV-X]
Duration: [HH:MM]
Summary: [Two sentence description of what happened and how it was resolved]
Impact: [Estimated transactions affected, clients affected, financial impact if known]
Postmortem: [Required by DATE / Not required]
```
### Post-resolution monitoring
For SEV-1 and SEV-2, maintain active Datadog monitoring for at least 2 hours after the resolution declaration. Assign a monitoring owner in the incident channel.
If metrics degrade again within the monitoring window, re-open the incident immediately rather than treating it as a new event.
### Rollback procedures
If a code deployment is identified as the contributing cause, initiate rollback via the standard deployment pipeline. Do not attempt hotfixes on production without a second engineer review for SEV-1 incidents.

## Client Communication Templates
Use these templates without modification unless the IC and Communications Lead agree that a specific situation requires adjustment. Consistency in incident communication builds client trust.
### Template 1: Initial acknowledgment (SEV-1)
**Subject:** Northlane Service Disruption - Payout Processing [DATE]
```
We are currently investigating an issue affecting payout processing on the Northlane platform.

Our engineering team identified the issue at [TIME] UTC and is actively working to restore full service. We will provide an update within 15 minutes.

We apologize for the disruption and understand the impact this may have on your operations.

Northlane Engineering Operations
```

### Template 2: Degradation notice with update
**Subject:** Northlane Incident Update - [TIME] UTC
```
Update as of [TIME] UTC:

We have identified the root cause of the payout processing disruption as [brief non-technical description, e.g., "a queue synchronization issue following an upstream provider timeout"].

Our team is actively implementing a resolution. Affected payouts will be processed in order once service is fully restored. No transactions have been lost.

Next update: [TIME] UTC or sooner if the situation changes.

Northlane Engineering Operations
```
### Template 3: Recovery confirmation
**Subject:** Northlane Service Restored - [DATE]
```
Payout processing has been fully restored as of [TIME] UTC.

All payouts queued during the disruption are being processed and will be completed within [estimated time, e.g., "the next 2 hours"]. You do not need to resubmit any transactions.

We are conducting a full postmortem review and will share a summary of findings and preventive measures within [3 / 5] business days.

We apologize for the disruption and appreciate your patience.

Northlane Engineering Operations
```
### Template 4: Postmortem follow-up
**Subject:** Northlane Incident Review - [DATE] Payout Disruption
```
Following the payout processing disruption on [DATE], we have completed our postmortem review.

Summary of findings:
[2–3 sentence non-technical summary of root cause and contributing factors]

What we are doing:
[Bullet list of 2–4 concrete remediation actions with target dates]

We have implemented [immediate fix] and are scheduled to complete [longer-term fix] by [DATE].

If you have questions or have observed any ongoing issues, please contact your account manager or reach out to us at support@northlane.io.

Northlane Engineering Operations
```

## Postmortem Process
### Postmortem culture
Northlane postmortems are blameless. The goal is to understand what happened and improve systems and processes. Individual engineers are not blamed for incidents. Contributing factors are documented, not causes assigned to people.
A postmortem is an engineering artifact, not a performance review.
### Who owns the postmortem?
The Incident Commander owns the postmortem document. They are responsible for:
- Scheduling the postmortem meeting
- Drafting the initial document from the incident timeline
- Facilitating the postmortem meeting
- Ensuring action items are assigned and tracked in Linear
### Postmortem timeline

| Severity | Postmortem meeting                   | Document published              |
| -------- | ------------------------------------ | ------------------------------- |
| SEV-1    | Within 48 hours of resolution        | Within 5 business days          |
| SEV-2    | Within 5 business days of resolution | Within 7 business days          |
| SEV-3    | Optional, IC discretion              | If held, within 5 business days |
### Postmortem meeting format
- Duration: 60 minutes maximum
- Attendees: IC, Investigation Lead, Communications Lead, Engineering Manager, relevant engineers
- Format: Walk the timeline, identify contributing factors, and define action items
- No blame, no speculation beyond the evidence in the timeline

## Postmortem Template
```markdown
# Postmortem: [Brief incident title]

**Date:** [Incident date]  
**Severity:** [SEV-1 / SEV-2]  
**Duration:** [HH:MM]  
**IC:** [Name]  
**Document status:** [Draft / Final]

---

## Summary

[2–3 sentence plain-language summary of what happened, what was affected, and how it was resolved. Written for a non-technical audience.]

---

## Impact

| Metric | Value |
|--------|-------|
| Duration | [HH:MM] |
| Clients affected | [Number or "All"] |
| Transactions affected | [Estimated count] |
| Payouts delayed | [Estimated value or count] |
| Client-reported issues | [Yes / No, count] |

---

## Timeline

| Time (UTC) | Event |
|------------|-------|
| [HH:MM] | [Event description] |
| [HH:MM] | [Event description] |
| [HH:MM] | [Event description] |

---

## Root Cause

[Technical description of the root cause. Be specific. Avoid vague language like "system failure" or "unexpected behavior." Describe the exact mechanism that caused the incident.]

---

## Contributing Factors

[List of factors that contributed to the incident or made it harder to detect and resolve. These are not causes assigned to people. Examples: monitoring gap, missing alert threshold, documentation unclear, retry logic not configured.]

- [Factor 1]
- [Factor 2]
- [Factor 3]

---

## What Went Well

[List things that worked during the incident response. Detection speed, escalation clarity, communication quality, tooling reliability.]

- [Item 1]
- [Item 2]

---

## What Did Not Go Well

[List things that slowed response or made the incident worse. Missing runbooks, unclear escalation paths, alert fatigue, tooling gaps.]

- [Item 1]
- [Item 2]

---

## Action Items

| Action | Owner | Priority | Due date | Linear ticket |
|--------|-------|----------|----------|---------------|
| [Description] | [Name] | [P1 / P2 / P3] | [DATE] | [Link] |
| [Description] | [Name] | [P1 / P2 / P3] | [DATE] | [Link] |

---

## Lessons Learned

[1–2 paragraphs summarizing the key takeaways from this incident. What does this incident tell us about our systems, our processes, or our documentation? What would have changed the outcome if it had been in place before the incident?]

---

*Postmortem completed by: [IC name]*  
*Review status: [Pending review / Approved]*
```

## Lessons Learned Workflow
Postmortem action items are not the end of the process. Northlane maintains a Lessons Learned register in Notion to track patterns across incidents over time.
### After every SEV-1 and SEV-2 postmortem
1. IC adds a summary entry to the Lessons Learned register (Notion: Engineering Operations > Lessons Learned)
2. Entry includes: incident date, severity, root cause category, key action items, and link to full postmortem
3. The Engineering Manager reviews the register quarterly for recurring patterns
### Root cause categories
Tag every postmortem with one or more root cause categories to enable pattern analysis:
- `deployment` - Incident caused or worsened by a code or infrastructure deployment
- `provider` - Upstream payment provider or banking partner failure
- `configuration` - Misconfiguration of infrastructure, services, or monitoring
- `capacity` - Resource exhaustion or scaling failure
- `dependency` - Internal service dependency failure
- `human` - Operational error during routine procedure
- `unknown` - Root cause not fully determined
### Recurring pattern threshold
If the same root cause category appears in 3 or more incidents within a 90-day window, Engineering Manager initiates a dedicated reliability review. The output of that review is a structural remediation plan, not individual action items.

## Sample Incident Timeline
The following is an example of a real incident timeline format drawn from a historical SEV-2 event. Use this as a reference when building timelines during live incidents.

| Time (UTC) | Event                                                                             |
| ---------- | --------------------------------------------------------------------------------- |
| 14:02      | Datadog alert triggered: payout queue latency exceeding 2x baseline               |
| 14:04      | PagerDuty page sent to primary on-call engineer                                   |
| 14:05      | Alert acknowledged. Initial assessment begun                                      |
| 14:07      | INCIDENT DECLARED SEV-2. IC assigned. `#inc-20260530-payout-queue-latency` opened |
| 14:09      | CS Lead notified. Statuspage update drafted                                       |
| 14:11      | Upstream provider timeout correlation identified in Datadog logs                  |
| 14:14      | Statuspage: “Investigating payout processing delays” posted                       |
| 14:18      | Engineering Manager notified per SEV-2 escalation policy                          |
| 14:22      | Failover routing to secondary provider initiated                                  |
| 14:28      | Queue latency beginning to recover. Monitoring continues                          |
| 14:38      | Queue processing confirmed normal. Datadog metrics at baseline                    |
| 14:42      | INCIDENT RESOLVED. Duration: 40 minutes                                           |
| 14:45      | Statuspage: “Service restored” posted. Client notification email sent             |
| 14:47      | Postmortem scheduled for [DATE]. IC assigned                                      |

*Northlane Engineering Operations*  
*Production Incident Response Playbook*  
*Internal use only*