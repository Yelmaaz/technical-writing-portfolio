# A Technical Overview  
  
---  
  
## Executive Summary  
  
Open Banking regulation has fundamentally restructured how financial institutions expose and consume data. The PSD2 directive in Europe and the Open Banking Standard in the United Kingdom have transformed banking from a system of closed, proprietary data silos into a network of interconnected, interoperable APIs through which third-party providers access account information, initiate payments, and deliver financial services on behalf of customers.  
  
This transformation has created substantial value. It has also created a sophisticated and expanding attack surface where identity, data, and infrastructure risks intersect in ways that traditional perimeter-based security models were not designed to address.  
  
This paper argues that effective API security in banking cannot be achieved through compliance adherence alone. Regulatory frameworks including PSD2, GDPR, and the OWASP API Security Top 10 provide necessary but insufficient guidance. They establish minimum standards in an environment where attacker sophistication consistently outpaces enforcement cycles. Institutions that treat compliance as a security destination rather than a security floor will remain structurally exposed.  
  
The paper outlines a layered defense architecture organized around three operational imperatives: preventing attacks through enforcement of financial-grade standards, detecting attacks through behavioral and anomaly-based monitoring, and recovering from attacks through automated response and immutable audit infrastructure. This framework is designed to account for the legacy constraints and operational complexity that characterize real banking environments, not idealized ones.  
  
---  
  
## 1. Introduction  
  
### The Regulatory Catalyst  
  
Prior to Open Banking mandates, financial institutions maintained tight control over customer data through proprietary systems with limited external interfaces. Third-party access, where it existed at all, was managed through bilateral agreements, bespoke integrations, and heavily negotiated data sharing arrangements. The model was slow, expensive, and resistant to innovation, but it was also relatively contained from a security perspective. The attack surface was narrow and well-understood.  
  
PSD2, which came into force across the European Union in 2018, changed this structurally. By requiring banks to provide standardized API access to licensed Third-Party Providers (TPPs), the regulation created a new class of entities with legitimate, regulated access to sensitive account data and payment initiation capabilities. The Open Banking Implementation Entity (OBIE) in the United Kingdom operationalized comparable requirements through the Open Banking Standard. Similar frameworks have since emerged in Australia, Brazil, Canada, and across Southeast Asia.  
  
The result is a financial services ecosystem in which customer data and transaction capability flow through APIs at scale and velocity that would have been unrecognizable a decade ago. As of 2024, the Open Banking ecosystem in the UK alone processes over eleven million API calls per day. Each call represents a potential point of failure.  
  
### What Is at Stake  
  
The consequences of API failure in banking are qualitatively different from those in most other sectors. A successful API attack against a financial institution can result in direct financial exfiltration through unauthorized transaction initiation, mass extraction of personally identifiable information (PII) at a scale that triggers both GDPR enforcement and reputational damage, account takeover enabling downstream fraud that affects individual customers, and systemic disruption to payment infrastructure with cascading effects across dependent services.  
  
Beyond the immediate financial and operational impact, the regulatory exposure is substantial. GDPR penalties can reach four percent of global annual turnover. PSD2 liability frameworks impose specific obligations around Strong Customer Authentication (SCA) and incident reporting. The combination of financial loss, regulatory penalty, and reputational damage from a significant API breach can threaten institutional viability in ways that more localized security failures do not.  
  
---  
  
## 2. Threat Model  
  
Effective API security begins with an accurate model of who is attacking, from where, and toward what objective. Banking API threats do not originate from a single attacker class, and controls designed for one class are frequently ineffective against another.  
  
### External Adversaries  
  
The most visible threat comes from external actors: automated botnets conducting credential harvesting at scale, organized fraud syndicates targeting transactional logic, and opportunistic attackers exploiting publicly documented vulnerabilities in API implementations. External adversaries operate outside the trust boundary entirely and rely on either stolen credentials, authentication bypass techniques, or brute-force enumeration to gain initial access.  
  
External attacks tend to be high-volume and detectable through volumetric monitoring, but sophisticated external actors increasingly adopt low-and-slow techniques specifically designed to evade threshold-based detection. The distinction between high-burst and low-and-slow attack patterns has significant implications for detection architecture, which is addressed in Section 5.  
  
### Trusted-but-Compromised Actors  
  
A more structurally challenging threat category is the trusted-but-compromised actor: a Third-Party Provider or OAuth client that holds legitimate, regulated access to banking APIs but whose credentials or infrastructure have been hijacked by a malicious party.  
  
This threat is particularly significant in the Open Banking context because TPPs are explicitly granted access by regulatory mandate. A compromised TPP presents valid authentication credentials, operates within expected permission scopes, and generates API traffic that is, at the transaction level, indistinguishable from legitimate activity. Detection requires behavioral analysis rather than perimeter controls.  
  
### Insider Threats  
  
Administrative misuse and operational error represent a third threat class that is frequently underweighted in external-facing security assessments. Privileged users with access to API management infrastructure, configuration systems, or backend data stores can cause unauthorized data exposure through deliberate action or configuration drift. The consequences of insider-driven API misconfiguration, particularly the exposure of undocumented or legacy endpoints, can be as severe as those of external attack and significantly harder to detect.  
  
### Primary Attacker Objectives  
  
Across all threat classes, the primary objectives are consistent: Account Takeover (ATO) enabling fraud or lateral access, mass PII extraction for sale or downstream exploitation, and unauthorized transaction injection for direct financial gain. Understanding these objectives matters for prioritization: not all API vulnerabilities carry equal risk, and security investment should be weighted toward the controls that most directly disrupt high-consequence attack paths.  
  
---  
  
## 3. The API Failure Taxonomy  
  
Banking API vulnerabilities can be organized into three conceptually distinct categories based on what they compromise: identity and access, data and authorization, and runtime infrastructure. This taxonomy maps closely to the OWASP API Security Top 10 while reflecting the specific trust and compliance architecture of regulated financial services.  
  
### Category A: Identity and Access Failures  
  
Identity failures concern the question of who is permitted to access the API and how that permission is verified and maintained.  
  
**Authentication bypass** occurs when flaws in credential validation logic or multi-factor authentication flows allow an attacker to establish a session without presenting valid credentials. In banking APIs, where SCA is a regulatory requirement under PSD2, authentication bypass represents both a security failure and a compliance violation.  
  
**Token lifecycle abuse** exploits weaknesses in how access tokens are issued, scoped, and revoked. OAuth 2.0 tokens with excessive lifespans, insufficient binding to client identity, or inadequate revocation mechanisms can be replayed by an attacker who has obtained them through credential theft or interception. The asymmetry matters: a token issued for a one-hour session that is not properly revocable can remain exploitable indefinitely.  
  
**Trust boundary failures** occur when Third-Party Provider permissions are improperly scoped or when delegated access is implemented in ways that allow privilege escalation beyond the intended authorization level. Open Banking's consent model is designed to limit TPP access to specific accounts and actions explicitly authorized by the customer. Misconfigured implementations of this model can grant broader access than the consent framework permits.  
  
### Category B: Data and Authorization Failures  
  
Authorization failures concern what an authenticated identity is permitted to do or access.  
  
**Broken Object-Level Authorization (BOLA)** is consistently identified as the most prevalent API vulnerability class across sectors and is particularly consequential in banking. BOLA occurs when an API endpoint accepts a user-supplied resource identifier, such as an account number or transaction ID, without verifying that the requesting identity is authorized to access that specific resource. An attacker who has established a legitimate session can enumerate resource identifiers and access accounts or transactions belonging to other customers. The attack requires no privilege escalation: it exploits the gap between authentication (confirming identity) and authorization (confirming entitlement to a specific resource).  
  
**Excessive data exposure** occurs when API responses return more data than the requesting client requires or is authorized to receive. Financial APIs that return full account objects, including fields irrelevant to the specific request, expose data that serves no functional purpose but creates compliance and privacy risk. The principle of data minimization, a core GDPR requirement, has direct application to API response design.  
  
**Payload manipulation** encompasses injection techniques, including SQL injection, JSON injection, and XML injection, that exploit insufficient input validation to alter backend state or extract data through manipulated query parameters or request bodies. In banking APIs, successful payload manipulation can result in unauthorized transaction initiation or modification of account data.  
  
### Category C: Runtime and Infrastructure Failures  
  
Infrastructure failures concern the operational environment in which APIs run rather than their logical design.  
  
**Transport insecurity** results from weak or misconfigured TLS implementations that permit traffic interception. Banking APIs handling sensitive financial data require current TLS versions with strong cipher suites. Mutual TLS (mTLS), which requires the client to present a certificate in addition to the server, provides a stronger guarantee for B2B API traffic and is mandated under Financial-grade API (FAPI) profiles for high-risk operations.  
  
**Resource exhaustion** through rate limit bypass enables brute-force attacks on authentication endpoints, credential stuffing at scale, and denial of service through API flooding. Effective rate limiting must account for distributed attack patterns in which individual source IPs operate below threshold while the aggregate attack volume is significant.  
  
**Shadow APIs and undocumented endpoints** represent a discovery blind spot that is particularly prevalent in banking environments with extended legacy infrastructure. Endpoints that exist outside the current security perimeter, whether because they predate the API governance program, were introduced without proper change management, or were created for testing and never decommissioned, cannot be protected by controls they are not covered by. Shadow API discovery is both a security and a compliance challenge under audit frameworks that require complete inventory of data access points.  
  
---  
  
## 4. Observed Exploitation Patterns  
  
Understanding how these vulnerability classes are combined in practice is necessary for designing detection controls that match actual attacker behavior rather than theoretical risk models.  
  
### Pattern 1: Low-and-Slow Data Extraction  
  
The most sophisticated external attacks against banking APIs are designed specifically to evade volumetric detection. The pattern proceeds in stages: initial reconnaissance through human-in-the-loop endpoint enumeration, identification of BOLA-vulnerable resources through systematic identifier probing, and then slow, sustained data harvesting calibrated to remain below the velocity thresholds that trigger automated alerts.  
  
Low-and-slow attacks are particularly difficult to detect because individual API calls are legitimate. The signal is in the pattern across calls over time, which requires behavioral baseline analysis rather than per-request threshold monitoring. Detection depends on identifying anomalous access patterns, such as sequential resource enumeration or access to accounts with no prior interaction history, rather than anomalous volume.  
  
### Pattern 2: Automated Account Takeover  
  
High-burst credential stuffing attacks use credential lists obtained from prior data breaches to test authentication at scale. Where rate limiting is insufficient or IP-based blocking is bypassed through distributed bot infrastructure, these attacks can produce authentication successes at meaningful rates given the scale of available credential lists. Successful authentication is followed immediately by automated session takeover, with rapid fund movement or PII extraction before detection and response can occur.  
  
The automated ATO pattern illustrates why authentication success is an insufficient signal for legitimate access. Behavioral indicators immediately following authentication, including the speed of subsequent requests, the nature of actions performed, and deviation from established account usage patterns, are necessary complements to authentication-layer controls.  
  
### Pattern 3: Supply Chain Compromise  
  
The most structurally difficult attack pattern to defend against involves the compromise of a legitimate TPP whose OAuth credentials are then used to access banking APIs under cover of authorized access. The attacker presents valid tokens, operates within the TPP's authorized permission scope, and generates traffic that is, at the individual request level, indistinguishable from the TPP's normal operations.  
  
Detection requires cross-referencing API activity against established behavioral baselines for specific TPPs, identifying access to accounts or data types outside normal usage patterns, and monitoring for access volume anomalies relative to the TPP's historical activity. This level of behavioral monitoring is operationally demanding but necessary given the structural access that Open Banking regulation grants to the TPP ecosystem.  
  
---  
  
## 5. Control Architecture  
  
Effective API security in banking requires controls organized across three operational functions: prevention, detection, and correction. Controls in each category address different points in the attack lifecycle and are necessary complements to one another. Prevention alone is insufficient in an environment where some attacks will succeed; detection and correction determine the consequence of that success.  
  
### Preventative Controls  
  
**Financial-grade API (FAPI) profile enforcement** provides the most comprehensive preventative standard specifically designed for financial services API security. FAPI profiles, developed by the OpenID Foundation, extend OAuth 2.0 and OpenID Connect with requirements including Proof Key for Code Exchange (PKCE), sender-constrained access tokens, and requirements for signed and encrypted request objects. FAPI 2.0, the current profile, is designed to address the specific threat model of Open Banking environments including the TPP trust boundary challenges described in Section 2.  
  
**Mutual TLS enforcement** for B2B API traffic eliminates a significant class of token replay and interception attacks by binding access tokens to the client certificate presented during the TLS handshake. Tokens bound in this way cannot be used by an attacker who has obtained the token but does not possess the corresponding client certificate. Implementation in legacy banking environments requires careful management of certificate lifecycle and may require compensating controls for endpoints where mTLS cannot be enforced directly.  
  
**Strict schema validation at the API gateway** provides a preventative layer against payload injection by rejecting requests that do not conform to the defined schema before they reach backend systems. Schema validation should cover request structure, parameter types and ranges, and the presence of unexpected fields. Validation at the gateway layer is preferable to validation in backend application code because it intercepts malformed requests before they reach systems with access to sensitive data.  
  
### Detective Controls  
  
**Real-time anomaly detection** for transactional velocity and volume shifts provides the primary detection layer against both high-burst automated attacks and the slower extraction patterns that evade threshold-based controls. Effective anomaly detection requires established behavioral baselines for accounts, TPPs, and API endpoints, against which deviations can be measured in real time. Static thresholds are insufficient; machine learning-based behavioral models that adapt to legitimate usage pattern changes are the current standard of practice.  
  
**Behavioral bot detection on authentication endpoints** addresses the credential stuffing attack pattern specifically. Bot detection at the authentication layer uses signals including request timing, device fingerprinting, interaction patterns, and behavioral biometrics to distinguish automated authentication attempts from human-initiated ones. This layer is necessary because IP-based blocking alone is insufficient against distributed bot infrastructure that rotates source addresses to evade blocklists.  
  
**Automated shadow API discovery** addresses the infrastructure visibility gap created by undocumented and legacy endpoints. Continuous discovery tooling that monitors network traffic, API gateway logs, and infrastructure configuration for endpoints outside the known API inventory provides both a security control and an audit compliance capability. Discovery should be integrated with change management processes to ensure new endpoints are identified and assessed before they enter production.  
  
### Corrective and Governance Controls  
  
**Immutable audit trails** for all state-changing API requests provide the forensic foundation for post-incident investigation, regulatory reporting, and compliance demonstration. Audit records must capture sufficient detail to reconstruct the sequence of events in an incident, including request content, authentication context, and backend state changes. Immutability, through append-only logging infrastructure or cryptographic integrity verification, is a requirement for audit records that will be relied upon in regulatory or legal proceedings.  
  
**Automated token revocation and session termination** for compromised identities limits the window of exploitation following credential theft or session compromise. Revocation must be propagated immediately across all systems that honor the token, including downstream services and partner APIs. Delayed or incomplete revocation leaves exploitable sessions active after compromise has been identified.  
  
**API lifecycle governance** establishes the organizational processes through which APIs are introduced, modified, and decommissioned with security controls applied consistently throughout. Security gates at deployment, mandatory vulnerability assessment for new endpoints, and defined sunsetting processes for legacy versions address the organizational conditions that produce shadow APIs and accumulated technical debt in the security perimeter.  
  
---  
  
## 6. Regulatory Alignment  
  
The three primary regulatory frameworks applicable to banking API security address different aspects of the risk landscape and are complementary rather than overlapping.  
  
**PSD2** mandates Strong Customer Authentication for account access and payment initiation, requires incident reporting within prescribed timeframes, and establishes the liability framework for TPP-related failures. PSD2 compliance is a necessary condition for Open Banking participation but does not constitute a sufficient security program. The regulation establishes minimum authentication requirements; it does not specify the full range of controls necessary to address the threat model described in this paper.  
  
**GDPR** provides the legal framework for data minimization and privacy-by-design principles that have direct application to API response design. The excessive data exposure vulnerability class described in Section 3 is both a security risk and a GDPR compliance failure. The principle of data minimization requires that API responses return only the data necessary for the specific purpose of the request, which maps directly to the security practice of limiting response scope.  
  
**OWASP API Security Top 10** provides the engineering taxonomy against which technical risk mapping should be conducted. The vulnerability categories described in this paper, including BOLA, excessive data exposure, and authentication failures, correspond to entries in the OWASP framework. OWASP does not carry regulatory force but provides the most current and operationally grounded technical reference for API security practitioners.  
  
A critical limitation of all three frameworks is temporal: regulatory standards are defined through legislative and standards processes that move significantly more slowly than attacker capability development. Compliance with current frameworks provides assurance against known and documented attack patterns. It does not provide assurance against emerging techniques that have not yet been incorporated into regulatory guidance. Security architecture that assumes compliance equals security will consistently lag behind the threat environment.  
  
---  
  
## 7. Conclusion  
  
Banking API security is not a problem that can be solved and closed. It is a continuous operational challenge in which the attack surface evolves as regulation expands API access, as new TPPs enter the ecosystem, and as attacker techniques adapt to deployed controls.  
  
The institutions most exposed are those that have treated Open Banking compliance as a security program rather than a regulatory minimum. Compliance frameworks provide a necessary foundation. They do not address the full threat model, they lag behind emerging attack patterns, and they were not designed to substitute for defense-in-depth security architecture.  
  
Effective API security in banking requires three things simultaneously: preventative controls that enforce financial-grade standards and limit the attack surface at the authentication and authorization layers; detective controls that identify attack patterns through behavioral analysis rather than static thresholds; and corrective controls that limit the consequence of successful attacks through automated response and the forensic infrastructure necessary for recovery and accountability.  
  
The legacy constraints of real banking environments, accumulated technical debt, undocumented endpoints, and the operational complexity of large-scale infrastructure, make this architecture more difficult to implement than it is to describe. The appropriate response to that complexity is not to lower the security standard but to build governance processes that systematically reduce the gap between the target architecture and the operational reality. API security in banking is not a destination. It is a direction of travel, and the institutions that maintain that direction consistently will be better positioned than those that treat it as a project with an end date.  
  
---  
  
## References and Further Reading  
  
- OpenID Foundation. *Financial-grade API (FAPI) 2.0 Security Profile.* openid.net/specs/fapi  
- OWASP Foundation. *OWASP API Security Top 10.* owasp.org/API-Security  
- European Banking Authority. *Guidelines on ICT and Security Risk Management.* eba.europa.eu  
- Open Banking Implementation Entity. *Open Banking Standard.* openbanking.org.uk  
- European Commission. *Payment Services Directive 2 (PSD2).* Directive 2015/2366/EU  
- National Institute of Standards and Technology. *Digital Identity Guidelines.* NIST SP 800-63  
- Saltzer, J.H. & Schroeder, M.D. *The Protection of Information in Computer Systems.* Proceedings of the IEEE, 1975  
  
---  
  
*This paper is intended for a technically informed audience including security architects, API platform engineers, compliance officers, and senior technology leadership in financial services organizations. It does not constitute legal or regulatory advice.*