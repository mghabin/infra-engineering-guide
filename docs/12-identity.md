# Chapter 12 — Human Identity & Workforce Access

This chapter is the missing half of the identity story. Chapter 06 §3
already owns *workload* identity — the rule that no `AKIA…`, no SP
client secret, and no service-account JSON key may sit on a workload.
This chapter owns *human* identity: employees, contractors, vendors,
and the privileged admin personas that the same humans wear when they
break-glass into production. Identity is named in chapter 01 as a
top-leverage decision; treating it only as a workload concern leaves
the larger blast-radius surface — joiners, movers, leavers, admins —
running on whatever the SaaS sign-up wizard defaulted to.

> Conventions, mirroring the sibling chapters:
> **must** = load-bearing, ≥2 independent sources required.
> **should** = strong default; deviate only with a written reason.
> **prefer** = taste call backed by evidence.
> **avoid** = explicitly rejected; replacement named.

> Scope boundary: workload identity (IRSA, GKE WI, AKS WI, SPIFFE,
> OIDC from CI to cloud) stays in ch06 §3 and is referenced, not
> re-decided, here. Audit log transport stays in ch06 §15. Zero-trust
> network access paths (IAP, BeyondCorp-style proxies, VPN
> replacement) stay in ch06 §13 — this chapter covers the *identity*
> side of the same control. mTLS is ch07.

---

## 1. WHO THIS CHAPTER IS FOR

- **[SCOPE] [must] Workforce identity is a single named owner with a single source of truth, distinct from workload identity.**
  Workforce identity covers every human principal that authenticates
  to a corporate system: employees, contractors, vendor support
  staff, and the same humans wearing admin hats. Workload identity
  (ch06 §3) covers non-human principals — pods, CI jobs, batch
  workers — that authenticate via short-lived federated tokens.
  Conflating them produces the two failure modes this chapter exists
  to prevent: humans authenticating with long-lived service-account
  secrets, and workloads carrying a human's password in a vault.
  - Sources: NIST SP 800-63-3, *Digital Identity Guidelines* (June 2017, includes errata through 2020), §4 — separate identity proofing and authenticator assurance for subscribers vs. machine-to-machine; NIST SP 800-207, *Zero Trust Architecture* (Aug 2020), §3.1 (subject = user *or* non-person entity, but as distinct subjects).
  - Concrete check: the IdP's user directory and the workload identity broker (cloud STS, SPIRE, etc.) are queryable separately. Any principal that authenticates via both a password and a service-account key is a finding — pick one, migrate the other.

---

## 2. IDP CHOICE

- **[IDP] [must] Exactly one corporate IdP is the source of truth for human authentication; every SaaS, cloud account, and internal service federates to it.**
  Two IdPs is two leaver processes, two MFA enrolment flows, two
  audit pipelines, and a guaranteed gap when an account is offboarded
  in one and not the other. The number of in-scope IdPs is `1`, plus
  a documented break-glass path (§8). NIST SP 800-63-3 §5 frames this
  as the Credential Service Provider role — there is one CSP per
  identity, not per application.
  - Sources: NIST SP 800-63-3, *Digital Identity Guidelines*, §5 (CSP role and binding); ISO/IEC 27001:2022, Annex A 5.16 *Identity management* and A 5.17 *Authentication information*; CISA, *Secure Cloud Business Applications (SCuBA) Hybrid Identity Solutions Guidance* (March 2023).
  - Concrete check: enumerate every production system's "sign in with…" button. Anything that still accepts a local username/password (other than the documented break-glass account) is a migration ticket, not an exception.

- **[IDP] [should] Pick the IdP your estate's centre of gravity already runs on; survey the alternatives before committing.**
  Identity is the stickiest piece of platform you will buy — migration
  is multi-quarter. Choose for fit, not feature parity:
  - **Microsoft Entra ID (formerly Azure AD)** — default when M365/Exchange/Intune are already deployed; deepest Conditional Access + PIM (§7); native federated credentials for workload identity (ch06 §3).
  - **Okta** — IdP-of-record when the estate is multi-cloud or SaaS-heavy and you need vendor-neutral SAML/OIDC plus mature SCIM connectors to long-tail SaaS.
  - **Google Workspace / Cloud Identity** — natural fit when GCP + Workspace are the productivity stack; Context-Aware Access integrates with BeyondCorp Enterprise for device-trust gating.
  - **JumpCloud** — small-to-mid estates that need IdP + MDM + RADIUS in one pane and cannot operate Entra/Okta at full surface area.
  - **Keycloak (self-hosted) / Red Hat build of Keycloak** — only when there is a hard requirement to *not* depend on a SaaS IdP (sovereign, air-gapped, regulated). Self-hosted IdP means *you* operate the tier-0 system whose outage takes down all sign-in; budget the SRE accordingly.
  - Sources: Microsoft Entra ID docs, *What is Microsoft Entra ID?* https://learn.microsoft.com/entra/fundamentals/whatis ; Okta docs, *Identity Engine overview* https://help.okta.com/oie/en-us/content/topics/identity-engine/oie-get-started.htm ; Google Cloud, *Cloud Identity overview* https://cloud.google.com/identity/docs/overview ; Keycloak docs, *Server administration guide* https://www.keycloak.org/docs/latest/server_admin/ ; NIST SP 800-63-3, §5 (CSP requirements apply identically regardless of vendor).
  - Concrete check: an ADR records the IdP decision, the alternatives surveyed, and the named owner team. "It came with the M365 tenant" is not an ADR.

---

## 3. AUTHENTICATION STRENGTH

- **[AUTH] [must] Every workforce identity authenticates with a phishing-resistant authenticator; password-only and SMS/voice OTP are unacceptable for any access to production, source code, cloud consoles, or the IdP itself.**
  Push-approval MFA (Duo-style numberless prompts) and TOTP raise the
  bar but were the proximate cause of the 2022 Uber, Cisco, and Okta
  support-portal compromises. NIST SP 800-63B §5.2.5 defines
  *verifier impersonation resistance* — a property satisfied by
  authenticators that cryptographically bind the assertion to the
  origin (FIDO2/WebAuthn, smart cards, PIV) and *not* by OTP/push.
  CISA's 2022 guidance is explicit that phishing-resistant MFA is
  the goal state.
  - Sources: NIST SP 800-63B, *Authentication and Lifecycle Management* (rev 3, June 2017 + errata), §5.2.5 (verifier impersonation resistance) and §4.3 (AAL3); W3C, *Web Authentication: An API for accessing Public Key Credentials, Level 2* (April 2021), https://www.w3.org/TR/webauthn-2/ ; CISA, *Implementing Phishing-Resistant MFA* (Oct 2022), https://www.cisa.gov/sites/default/files/publications/fact-sheet-implementing-phishing-resistant-mfa-508c.pdf .
  - Concrete check: the IdP's authentication-method policy disables SMS, voice, email-OTP, and security questions for production-touching roles. Push-with-number-matching is a transitional fallback, not a destination.

- **[AUTH] [should] Map authenticator policy to NIST AAL levels, and qualify the "AAL3 means hardware key" shorthand.**
  AAL1 is single-factor; AAL2 is multi-factor with replay resistance;
  AAL3 additionally requires that *both* factors be impersonation-
  resistant and one be hardware-based. NIST allows multiple
  combinations at AAL3 — a multi-factor cryptographic device (a
  resident-key FIDO2 authenticator with user verification, a PIV
  card with PIN), or a single-factor cryptographic device combined
  with a memorised secret. "AAL3 = YubiKey" is the most common
  implementation, not the definition. Production access, IdP admin,
  and PIM elevation should be AAL3; routine SaaS access can be AAL2.
  - Sources: NIST SP 800-63B, §4.2 (AAL2) and §4.3 (AAL3, especially Table 4-1 on permissible authenticator combinations); FIDO Alliance, *FIDO2: Web Authentication (WebAuthn)* https://fidoalliance.org/fido2/ ; NIST SP 800-157r1, *Guidelines for Derived PIV Credentials* (Aug 2023).
  - Concrete check: the Conditional Access / Okta sign-on policy explicitly references AAL2 vs AAL3 for distinct app groups. Every authenticator used to satisfy AAL3 appears on the IdP's allow-list (model + AAGUID), not just "any FIDO2".

- **[AUTH] [should] Mitigate push fatigue even on the way out.**
  While push-MFA is being retired, enable number-matching, geographic
  context, and additional-context-in-prompt; cap unanswered prompts;
  alert on rapid-fire denial patterns. These are stopgaps, not the
  destination — the destination is FIDO2 everywhere.
  - Sources: Microsoft Entra docs, *How number matching works in MFA push notifications* https://learn.microsoft.com/entra/identity/authentication/how-to-mfa-number-match ; Okta, *Verify with Okta Verify (push)* configuration guidance, https://help.okta.com/oie/en-us/content/topics/identity-engine/authenticators/configure-okta-verify.htm ; CISA, *Implementing Number Matching in MFA Applications* (Oct 2022).
  - Concrete check: push prompts without number-matching are blocked by policy, not by request. The IdP fires an alert (ch06 §15 pipeline) when a single principal denies ≥3 prompts in 5 minutes.

---

## 4. FEDERATION & SSO

Throughout this chapter, **tier-0 / production-critical** refers to any
system whose compromise or outage materially harms the business — the
IdP itself, cloud-account roots, source-code hosting, the CI/CD control
plane, the production deploy pipeline, customer-data stores, the
secrets manager, and any SaaS named in a regulator-facing risk
register. Treat the list as explicit (named in the IdP catalogue with a
"tier-0" tag), not implicit. Tier-1 is "hurts but recoverable in a
quarter"; tier-2 is everything else.

- **[FED] [must] Every SaaS in scope authenticates via SAML 2.0 or OIDC against the corporate IdP; local credentials are disabled where the vendor permits and isolated where they don't.**
  The point of an IdP is that disabling an account in one place
  disables it everywhere. A SaaS that keeps a parallel local
  password store defeats this — and the leaver process — silently.
  Where the SaaS only offers SAML/OIDC on a higher tier ("SSO tax"),
  the cost of the upgrade is part of the cost of the SaaS; budget it
  at procurement, not after the breach.
  - Sources: OASIS, *SAML V2.0 Core* (March 2005), https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf ; OpenID Foundation, *OpenID Connect Core 1.0* (Nov 2014, errata set 2), https://openid.net/specs/openid-connect-core-1_0.html ; NIST SP 800-63C, *Federation and Assertions* (rev 3), §5 (federation assurance levels).
  - Concrete check: SaaS inventory has a column "auth = IdP-federated | local-disabled | local-isolated | NONE". Any row with NONE is a procurement or migration ticket with a date.

- **[FED] [prefer] OIDC for new SaaS and internal apps; SAML where the vendor only ships SAML or where an existing federation is already operating.**
  Both protocols satisfy the federation requirement. OIDC is the
  better default for greenfield (JSON, mobile-friendly, simpler
  discovery, the same primitives that power workload identity in
  ch06 §3). SAML remains common in enterprise SaaS and is not
  deprecated — there is no urgency to migrate working SAML.
  - Sources: OpenID Connect Core 1.0; OASIS SAML 2.0 Core; NIST SP 800-63C §5.
  - Concrete check: new SaaS onboarding chooses OIDC if both are offered; SAML choice is recorded with reason.

- **[FED] [must] Provisioning is SCIM 2.0 from a single source of truth, not JIT-only.**
  Just-in-time (JIT) provisioning at first login is convenient but
  has no leaver path — the account exists in the SaaS forever
  unless something else removes it. SCIM 2.0 (RFC 7642 requirements,
  RFC 7643 schema, RFC 7644 protocol) is the standardised
  push-from-IdP that creates, updates, and *deactivates* accounts
  in the downstream system. Use JIT only as a fallback for SaaS
  that does not implement SCIM, paired with a scheduled
  reconciliation job.
  - Sources: IETF, *RFC 7642: SCIM Definitions, Overview, Concepts, and Requirements* (Sept 2015); *RFC 7643: SCIM Core Schema* (Sept 2015); *RFC 7644: SCIM Protocol* (Sept 2015); ISO/IEC 27001:2022 Annex A 5.16.
  - Concrete check: every SaaS in the inventory has either SCIM enabled or a documented reconciliation job; there is no SaaS where the only deprovisioning mechanism is a human ticket.

---

## 5. LIFECYCLE — JOINER / MOVER / LEAVER

- **[JML] [must] HR is the source of truth for employment status; identity changes are driven by HR events, not tickets.**
  When HR marks a worker terminated, the IdP must disable the
  account, revoke active sessions, and trigger SCIM deprovisioning
  to every downstream system within a bounded time. The same applies
  to *movers* — a transfer between teams must add the new
  group memberships and remove the old, not just add. Mover risk is
  the one most teams under-invest in; standing privilege accumulates
  across a career and is invisible without an attestation pass (§10).
  - Sources: ISO/IEC 27001:2022 Annex A 5.16 *Identity management*, A 5.18 *Access rights*, and A 6.5 *Responsibilities after termination or change of employment*; AICPA, *SOC 2 Trust Services Criteria* (2017, rev 2022), CC6.2 (registration and authorisation of new users) and CC6.3 (modification and removal); NIST SP 800-53 rev 5, AC-2 *Account Management*.
  - Concrete check: terminate a test account in the HRIS; measure wall-clock time until (a) IdP sign-in fails, (b) downstream SCIM deprovisioning completes, (c) active OAuth refresh tokens are revoked. The number is published; an SLO exists.

- **[JML] [should] The leaver SLO is hours, not days, for production-touching roles.**
  Insider-threat postmortems repeatedly find that the proximate
  failure was a leaver still holding access at the moment of the
  incident. SOC 2 CC6.2/CC6.3 require *timely* deprovisioning; the
  number is yours to set and defend.
  - Sources: SOC 2 Trust Services Criteria CC6.2/CC6.3; NIST SP 800-53 rev 5 AC-2(3) *Disable Accounts*.
  - Concrete check: the leaver SLO is in writing, the dashboard shows compliance over the last 90 days, and breaches generate incidents.

---

## 6. AUTHORIZATION

- **[AUTHZ] [must] Authorisation is group-based, not user-based; group membership is granted by request-and-approve, not by edit.**
  Per-user grants are unreviewable and survive movers indefinitely.
  Groups (or RBAC roles, or ABAC attribute sets) are the unit of
  review. The IdP, not the downstream app, is the source of group
  membership where federation supports it (SCIM groups, SAML group
  claims, OIDC `groups` scope).
  - Sources: NIST SP 800-162, *Guide to Attribute Based Access Control (ABAC)* (Jan 2014, updated Aug 2019); NIST SP 800-53 rev 5, AC-2/AC-3/AC-6; ISO/IEC 27001:2022 Annex A 5.18.
  - Concrete check: the cloud account / SaaS shows zero direct user-to-permission bindings outside the documented break-glass identity. Every grant traces to a group, every group has an owner.

- **[AUTHZ] [prefer] RBAC for stable job functions; ABAC for context-sensitive decisions; avoid mixing in the same decision point.**
  RBAC is cheap to reason about and audit; ABAC (NIST SP 800-162)
  expresses things RBAC cannot — "engineers in the on-call rotation
  for service X during their shift". Use ABAC where the policy
  genuinely needs runtime attributes, not as a default.
  - Sources: NIST SP 800-162; NIST SP 800-53 rev 5, AC-3(7) Role-Based Access Control and AC-3(13) Attribute-Based Access Control.
  - Concrete check: the access model is documented; the same resource is not protected by both an RBAC role and a contradicting ABAC rule.

- **[AUTHZ] [must] Just-in-time (JIT) elevation replaces standing privilege for sensitive roles; grants are time-bound and approved.**
  Standing admin is the largest single source of blast-radius growth.
  JIT elevation — request, approve, time-box, auto-revoke — is now
  table stakes across the major IdPs and clouds (§7). The default
  state of an engineer's identity at 3 a.m. is "no production write
  access"; they elevate when they need it and the grant expires.
  - Sources: Microsoft Entra docs, *What is Privileged Identity Management?* https://learn.microsoft.com/entra/id-governance/privileged-identity-management/pim-configure ; AWS, *IAM Identity Center temporary elevated access* https://docs.aws.amazon.com/singlesignon/latest/userguide/temporary-elevated-access.html ; Google Cloud, *Privileged Access Manager overview* https://cloud.google.com/iam/docs/pam-overview .
  - Concrete check: query the cloud IAM for human principals with standing write access to production. The number trends down quarter over quarter; each remaining grant has a written exception with an expiry date.

- **[AUTHZ] [avoid] Wildcard groups, wildcard role bindings, and "Domain Users"-style grants on production resources.**
  A group called `engineers-all` on a production resource is a JIT
  failure waiting to happen. Wildcard `*` IAM actions on human roles
  are the same anti-pattern at the policy layer.
  - Sources: AWS, *IAM best practices* https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html (least privilege); NIST SP 800-53 rev 5 AC-6 *Least Privilege*.
  - Concrete check: a static analysis pass over IAM/RBAC fails the build on `*` actions or `Everyone`/`AllUsers`/`Domain Users` on production resources outside named exceptions.

---

## 7. PRIVILEGED ACCESS (PAM / PIM)

- **[PAM] [must] Standard and admin identity are separate principals belonging to the same human; the admin principal does not read email, browse the web, or hold OAuth grants to SaaS.**
  Admin sessions originating from the same browser profile that
  opened a phishing email is the Equation that generates most
  enterprise breaches. The standard identity is for productivity
  software; the admin identity exists only to perform privileged
  operations and is enrolled with stronger authenticators (AAL3,
  §3) and stricter Conditional Access (§12).
  - Sources: NIST SP 800-53 rev 5 AC-6(2) *Non-Privileged Access for Nonsecurity Functions* and AC-6(5) *Privileged Accounts*; Microsoft, *Privileged access: strategy* https://learn.microsoft.com/security/privileged-access-workstations/privileged-access-strategy ; NCSC UK, *Secure system administration* https://www.ncsc.gov.uk/collection/secure-system-administration .
  - Concrete check: the IdP report "principals with role X" returns admin-suffixed accounts for role X and standard accounts for everyday roles, with no overlap.

- **[PAM] [should] Privileged operations originate from a hardware-isolated workstation (PAW) or an equivalent ephemeral, locked-down environment.**
  A PAW is a dedicated, locked-down, monitored device used only for
  admin work. Equivalent postures are acceptable: cloud-hosted
  bastion VDIs (AWS WorkSpaces, Azure Virtual Desktop, Chrome
  Enterprise) gated by device trust, ephemeral Just-Enough-Admin
  jumpboxes whose state is wiped per session.
  - Sources: Microsoft, *Privileged Access Workstations* https://learn.microsoft.com/security/privileged-access-workstations/privileged-access-devices ; NCSC UK, *Secure system administration*; NIST SP 800-53 rev 5 AC-6(7).
  - Concrete check: PIM elevation policy is conditional on device-trust signal (§12); a non-PAW device cannot complete the elevation flow.

- **[PAM] [must] Privileged elevation is just-in-time, time-bound, approved, and recorded with session detail proportionate to the blast radius.**
  Microsoft Entra PIM, AWS IAM Identity Center temporary elevated
  access, and Google Cloud Privileged Access Manager all implement
  the same primitive: request → approve → assume role for N hours →
  auto-revoke. For the highest-risk operations (DBA on prod
  database, root on customer-data store), session recording or
  command logging is the proportionate control.
  - Sources: Microsoft Entra docs, *Configure Privileged Identity Management* https://learn.microsoft.com/entra/id-governance/privileged-identity-management/pim-configure ; AWS, *Temporary elevated access in IAM Identity Center* https://docs.aws.amazon.com/singlesignon/latest/userguide/temporary-elevated-access.html ; Google Cloud, *Privileged Access Manager overview* https://cloud.google.com/iam/docs/pam-overview ; CyberArk / Teleport docs for session recording where used.
  - Concrete check: any standing admin role assignment with no expiry is a finding. Elevation events are queryable for the last 365 days and tie to a request ID.

---

## 8. BREAK-GLASS

- **[BG] [must] Every IdP, cloud tenant, and tier-0 system has a documented break-glass identity that bypasses federation, MFA-method-restriction, and Conditional Access — and only those constraints.**
  The IdP outage that takes down all sign-in is a known scenario, not
  a hypothetical (§13). Break-glass exists so the company is not
  locked out of itself when its own auth plane is the incident.
  Break-glass is *not* a backdoor; its credentials are sealed,
  use-monitored, and rotated.
  - Sources: Google SRE, *Building Secure and Reliable Systems* (O'Reilly 2020), Ch. 5 *Design for Least Privilege*, "Breakglass" pattern; Google SRE Workbook, Ch. 5 *Alerting on SLOs* (alerting hooks for break-glass use); Microsoft Entra docs, *Manage emergency-access accounts* https://learn.microsoft.com/entra/identity/role-based-access-control/security-emergency-access ; AWS, *Root user best practices* https://docs.aws.amazon.com/IAM/latest/UserGuide/root-user-best-practices.html .
  - Concrete check: each tier-0 system has a named break-glass identity; the credential is sealed in a tamper-evident envelope or split-knowledge vault; an alert fires within minutes on any successful sign-in.

- **[BG] [must] Break-glass use requires multi-party check-out, fires a high-priority alert, and triggers a post-use rotation.**
  Single-person access to break-glass collapses the control. Two-
  person check-out (split secret, dual approval, paired sealed
  envelopes) plus mandatory rotation post-use prevents the credential
  from being silently captured during legitimate use.
  - Sources: Google, *Building Secure and Reliable Systems*, Ch. 5; NIST SP 800-53 rev 5 AC-3(2) *Dual Authorization*; ISO/IEC 27001:2022 Annex A 8.2 *Privileged access rights*.
  - Concrete check: the break-glass runbook names two approvers; rotation is on the calendar; a quarterly drill exercises the path end-to-end without a real incident, and the drill itself is logged.

---

## 9. CONTRACTOR & VENDOR ACCESS

- **[EXT] [must] Contractor and vendor identities are first-class IdP principals with an enforced expiry date; no shared accounts, no exceptions.**
  A "vendor support" account shared across a vendor's staff has no
  attribution, no leaver process when one of their staff departs, and
  no MFA story. Provision a per-individual identity in your IdP, mark
  it as external, and set a contract-aligned expiry that requires
  re-approval to extend.
  - Sources: NIST SP 800-53 rev 5 AC-2(2) *Automated Temporary and Emergency Account Management*; ISO/IEC 27001:2022 Annex A 5.19 *Information security in supplier relationships* and A 5.16; SOC 2 CC6.1.
  - Concrete check: the IdP has a "external-identity" lifecycle with a default expiry ≤ contract term; expired identities are auto-disabled; zero shared accounts in audit.

- **[EXT] [should] Vendor access is scoped, time-bound, and routed through the same JIT/PIM machinery as employee admin access.**
  The vendor's blast radius is whatever standing access you give them
  between tickets. JIT applies identically.
  - Sources: NIST SP 800-53 rev 5 AC-6; ISO/IEC 27001:2022 Annex A 5.19.
  - Concrete check: vendor admin grants are visible in the same PIM dashboard as employee grants; the report distinguishes them.

---

## 10. IDENTITY GOVERNANCE & ADMINISTRATION (IGA)

- **[IGA] [must] Access reviews run on a defined cadence, with the resource owner — not the requester — as approver.**
  Mover-driven privilege accumulation is invisible without periodic
  attestation. ISO/IEC 27001:2022 Annex A 5.18 requires periodic
  review of access rights; SOC 2 CC6.2/CC6.3 require evidence that
  the review happened and that revocations followed. Quarterly is a
  reasonable default; sensitive systems (production cloud,
  source-code admin, finance, customer data) should be more frequent.
  - Sources: ISO/IEC 27001:2022 Annex A 5.18 *Access rights*; AICPA, *SOC 2 Trust Services Criteria* CC6.2 and CC6.3; NIST SP 800-53 rev 5 AC-2(j) *Account Management — periodic review*.
  - Concrete check: each in-scope system has a review owner, a cadence, and the last-review date is queryable. Reviews that revoke zero access in three consecutive cycles are themselves audited (rubber-stamping).

- **[IGA] [must] Segregation of duties (SoD) conflicts are encoded as policy, not memorised.**
  The classic pairs — developer + production deployer, requester +
  approver, payment initiator + payment approver — are SoD
  violations regardless of how trustworthy the individual is. Encode
  the conflicting role pairs in the IGA tool and block conflicting
  grants at request time.
  - Sources: ISO/IEC 27001:2022 Annex A 5.3 *Segregation of duties*; SOC 2 CC1.3, CC5.1; NIST SP 800-53 rev 5 AC-5 *Separation of Duties*.
  - Concrete check: the IGA system has an SoD ruleset; granting a conflicting role requires a documented compensating control.

---

## 11. AUDIT & TELEMETRY

- **[AUDIT] [must] Identity events feed the same audit pipeline owned by ch06 §15; identity-specific signals are derived in that pipeline, not in a parallel one.**
  Sign-in success/failure, MFA challenges, role-assumption, group
  membership change, PIM elevation, SCIM provisioning, and
  break-glass use are first-class events. Transport, retention, and
  query layer are ch06 §15's job; this chapter only specifies which
  signals must exist and what they must enable.
  - Sources: NIST SP 800-92, *Guide to Computer Security Log Management* (Sept 2006); ISO/IEC 27001:2022 Annex A 8.15 *Logging* and A 8.16 *Monitoring activities*; SOC 2 CC7.2 *Detection of security events*.
  - Concrete check: the ch06 §15 audit pipeline can answer: who signed in to system X in the last 24h, who currently holds elevated role Y, who used break-glass account Z and when.

- **[AUDIT] [should] Detections required at minimum: failed-MFA bursts, impossible travel, first-time-from-country, anomalous privilege use, elevation outside business hours without an incident, dormant-account sign-in.**
  These are the signals the major IdPs already compute (Entra ID
  Protection, Okta ThreatInsight, Google Workspace alert center).
  Subscribe to them and route to the SOC, do not rebuild them.
  - Sources: Microsoft, *What is Microsoft Entra ID Protection?* https://learn.microsoft.com/entra/id-protection/overview-identity-protection ; Okta, *ThreatInsight* https://help.okta.com/en-us/content/topics/security/threat-insight/ai-ti-overview.htm ; Google Workspace, *Alert center* https://support.google.com/a/answer/9105393 .
  - Concrete check: each detection is wired to a runbook; the SOC has acknowledged ownership.

---

## 12. WORKSTATION POSTURE AS IDENTITY DEPENDENCY

- **[POSTURE] [should] Sensitive sessions require a managed, healthy device; device trust is a Conditional Access input, not an afterthought.**
  Identity is necessary but not sufficient — a valid token on a
  compromised laptop is a compromised session. Device trust
  (Intune compliance, Jamf, Chrome Enterprise device trust, GCP
  Context-Aware Access) feeds the IdP's access policy and gates
  PIM elevation, production access, and admin SaaS.
  - Sources: Microsoft Entra docs, *Conditional Access: Require compliant device* https://learn.microsoft.com/entra/identity/conditional-access/concept-conditional-access-grant ; Google Cloud, *Context-Aware Access overview* https://cloud.google.com/beyondcorp-enterprise/docs/overview ; NIST SP 800-207 *Zero Trust Architecture*, §3.4.1 (device health as policy input).
  - Concrete check: Conditional Access policies for production-touching apps explicitly require device-compliance signal; non-managed device sign-in is denied or routed to a restricted session profile.

---

## 13. COMMON ANTI-PATTERNS

- **[avoid] Local-IDP fallback that survives migration.** Migrating to
  a cloud IdP without disabling local accounts in the migrated SaaS
  leaves a parallel auth plane with no leaver path. Fix: scheduled
  reconciliation that fails the build when a local account exists.
- **[avoid] Shared admin accounts.** "The DBA team uses
  `dba@company.com`." There is no attribution and no leaver story.
  Fix: per-individual admin identities under the PAM model (§7).
- **[avoid] MFA exemptions for "service accounts" that are really
  humans.** A "service account" that a human signs in to with a
  password and uses interactively is a human account with MFA
  disabled. Fix: either it is a workload identity (ch06 §3) or it is
  a human identity with full MFA — never a third category.
- **[avoid] Indefinite contractor access.** Six-month engagement,
  three-year-old account. Fix: enforced expiry with re-approval
  (§9).
- **[avoid] IdP outage that takes down the whole company.**
  Single-IdP is correct (§2); not having a tested break-glass path
  is the failure. Fix: §8.
- **[avoid] Group nesting deeper than two levels.** Effective
  membership becomes unreviewable. Fix: flatten on a schedule, with
  the IGA tool.
- **[avoid] OAuth grants from corporate IdP to arbitrary third-party
  apps without admin consent.** Token theft via consent phishing is
  the modern version of password phishing. Fix: admin-consent-only
  for risky scopes; quarterly OAuth grant review.

---

## 14. COMMON PATTERNS

- **[prefer] Break-glass envelope.** Two sealed envelopes in two
  fire safes; opening either fires the alert; opening one without
  the other is the incident; rotation drill quarterly.
- **[prefer] PIM with auto-expiry as the default elevation.** The
  default state of every engineer is "no production write access";
  elevation is one click, time-boxed, recorded.
- **[prefer] SCIM-driven provisioning from the HRIS.** HR event →
  IdP user lifecycle → SCIM push → downstream SaaS account state.
  No JIT-only.
- **[prefer] Vendor SAML enforcement at procurement.** SSO-tax tier
  is a line item, not an exception. The security team has veto on
  SaaS that does not federate.
- **[prefer] Conditional Access policy bundle.** A small, named
  set of policies — "require compliant device for prod", "require
  AAL3 for admin", "block legacy auth", "block sign-in from
  high-risk countries", "require re-auth for high-impact actions"
  — versioned and applied uniformly, not redrawn per app.

---

## 15. SOURCES

Primary sources only. Every normative claim above cites at least one
of these.

- NIST SP 800-63-3, *Digital Identity Guidelines* (June 2017, errata through 2020). https://pages.nist.gov/800-63-3/
- NIST SP 800-63B, *Digital Identity Guidelines: Authentication and Lifecycle Management* (rev 3). https://pages.nist.gov/800-63-3/sp800-63b.html
- NIST SP 800-63C, *Digital Identity Guidelines: Federation and Assertions* (rev 3). https://pages.nist.gov/800-63-3/sp800-63c.html
- NIST SP 800-53 rev 5, *Security and Privacy Controls for Information Systems and Organizations* (Sept 2020, updates through 2023). https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final
- NIST SP 800-162, *Guide to Attribute Based Access Control (ABAC)* (Jan 2014, updated Aug 2019). https://csrc.nist.gov/pubs/sp/800/162/upd1/final
- NIST SP 800-207, *Zero Trust Architecture* (Aug 2020). https://csrc.nist.gov/pubs/sp/800/207/final
- NIST SP 800-92, *Guide to Computer Security Log Management* (Sept 2006). https://csrc.nist.gov/pubs/sp/800/92/final
- NIST SP 800-157r1, *Guidelines for Derived PIV Credentials* (Aug 2023). https://csrc.nist.gov/pubs/sp/800/157/r1/final
- IETF RFC 7642, *SCIM Definitions, Overview, Concepts, and Requirements* (Sept 2015). https://datatracker.ietf.org/doc/html/rfc7642
- IETF RFC 7643, *SCIM Core Schema* (Sept 2015). https://datatracker.ietf.org/doc/html/rfc7643
- IETF RFC 7644, *SCIM Protocol* (Sept 2015). https://datatracker.ietf.org/doc/html/rfc7644
- OASIS, *SAML V2.0 Core* (March 2005). https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf
- OpenID Foundation, *OpenID Connect Core 1.0* (Nov 2014, errata set 2). https://openid.net/specs/openid-connect-core-1_0.html
- W3C, *Web Authentication: An API for accessing Public Key Credentials, Level 2* (April 2021). https://www.w3.org/TR/webauthn-2/
- FIDO Alliance, *FIDO2: Web Authentication (WebAuthn)*. https://fidoalliance.org/fido2/
- ISO/IEC 27001:2022, *Information security management systems — Requirements*, Annex A controls 5.3, 5.15–5.19, 6.5, 8.2, 8.15, 8.16.
- AICPA, *SOC 2 Trust Services Criteria* (2017, rev 2022), CC1.3, CC5.1, CC6.1–CC6.3, CC7.2.
- CISA, *Implementing Phishing-Resistant MFA* (Oct 2022). https://www.cisa.gov/sites/default/files/publications/fact-sheet-implementing-phishing-resistant-mfa-508c.pdf
- CISA, *Implementing Number Matching in MFA Applications* (Oct 2022). https://www.cisa.gov/MFA
- CISA, *SCuBA Hybrid Identity Solutions Guidance* (March 2023). https://www.cisa.gov/resources-tools/services/secure-cloud-business-applications-scuba-project
- Google, *Building Secure and Reliable Systems* (O'Reilly, 2020), Ch. 5 *Design for Least Privilege* (break-glass pattern). https://sre.google/books/building-secure-reliable-systems/
- Google SRE Workbook, Ch. 5 *Alerting on SLOs*. https://sre.google/workbook/alerting-on-slos/
- Microsoft Entra docs, *What is Microsoft Entra ID?* https://learn.microsoft.com/entra/fundamentals/whatis
- Microsoft Entra docs, *Privileged Identity Management — Configure*. https://learn.microsoft.com/entra/id-governance/privileged-identity-management/pim-configure
- Microsoft Entra docs, *Manage emergency-access accounts*. https://learn.microsoft.com/entra/identity/role-based-access-control/security-emergency-access
- Microsoft Entra docs, *How number matching works in MFA push notifications*. https://learn.microsoft.com/entra/identity/authentication/how-to-mfa-number-match
- Microsoft Entra docs, *Conditional Access — Grant controls*. https://learn.microsoft.com/entra/identity/conditional-access/concept-conditional-access-grant
- Microsoft Entra docs, *What is Microsoft Entra ID Protection?* https://learn.microsoft.com/entra/id-protection/overview-identity-protection
- Microsoft, *Privileged access strategy* and *Privileged Access Workstations*. https://learn.microsoft.com/security/privileged-access-workstations/privileged-access-strategy
- AWS, *IAM best practices*. https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html
- AWS, *IAM Identity Center — Temporary elevated access*. https://docs.aws.amazon.com/singlesignon/latest/userguide/temporary-elevated-access.html
- AWS, *Root user best practices*. https://docs.aws.amazon.com/IAM/latest/UserGuide/root-user-best-practices.html
- Google Cloud, *Cloud Identity overview*. https://cloud.google.com/identity/docs/overview
- Google Cloud, *Privileged Access Manager overview*. https://cloud.google.com/iam/docs/pam-overview
- Google Cloud, *BeyondCorp Enterprise / Context-Aware Access overview*. https://cloud.google.com/beyondcorp-enterprise/docs/overview
- Google Workspace, *Alert center*. https://support.google.com/a/answer/9105393
- Okta docs, *Identity Engine overview*. https://help.okta.com/oie/en-us/content/topics/identity-engine/oie-get-started.htm
- Okta docs, *Configure Okta Verify*. https://help.okta.com/oie/en-us/content/topics/identity-engine/authenticators/configure-okta-verify.htm
- Okta, *ThreatInsight*. https://help.okta.com/en-us/content/topics/security/threat-insight/ai-ti-overview.htm
- Keycloak docs, *Server administration guide*. https://www.keycloak.org/docs/latest/server_admin/
- NCSC UK, *Secure system administration*. https://www.ncsc.gov.uk/collection/secure-system-administration
