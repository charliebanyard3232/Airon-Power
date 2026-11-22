# Airon Power — Requirements (release one)

> **Fictional company for training purposes.** Airon Power, its customers, partners and people are invented. All data is simulated.

| | |
|---|---|
| Version | Draft v3, 22 Nov 2026 |
| Owner | Product owner |
| Scope | Release one (core customer value) |
| Contents | Decisions, assumptions, users, role–permission matrix, security principles, functional (FR), data (DR), non-functional (NFR) and security (SR) requirements, suggested Jira epics |
| Reviewed by | Enterprise architect (`docs/architecture/requirements-challenge-log.md`, 14 findings, all resolved in v2); AI Security Engineer (`docs/security/security-review.md`, merged in v3) |
| Status | **Not signed off.** No application code is to be built from this document yet. Rows marked Draft or Open need Charlie's decision. The full threat model follows the architect's data flow diagrams. |
| Next | Charlie answers Q1–Q4 → architect designs to the security principles → full threat model → backlog created in Jira from this document |

## How to read this

- **Priority** (MoSCoW for release one): Must, Should, Could.
- **Status**:
  - **Agreed**: follows a decision Charlie has made.
  - **Draft**: proposed; needs Charlie's OK.
  - **Open (AS-xx / Qx)**: depends on an open assumption or security decision.
- **Epic**: suggested Jira epic (section "Suggested Jira epics"). To be mapped onto the existing Airon HQ epics when the backlog is created.
- IDs are stable: new items are added at the end of each list, never renumbered.

## Change history

| Version | Change |
|---|---|
| v1 | First draft: 32 FR, 19 NFR, 8 assumptions. |
| v2 | Enterprise architect challenge log applied (RC-01 to RC-14): decisions D-07 to D-10; FR-33 to FR-38; DR-01 to DR-03; NFR-20 to NFR-22; role–permission matrix; data classification. |
| v3 | AI Security Engineer review merged: security principles SP-01 to SP-12; security requirements SR-01 to SR-28; security NFRs NFR-23 to NFR-28; security decisions Q1 to Q4 added as open; Jira epic column added. |

---

## Decisions agreed

| ID | Decision |
|---|---|
| D-01 | Release one = live supply, 48-hour maintenance warnings (in-app and email), outage tracking, admin portal. |
| D-02 | Not in release one: AI chatbot, partner API, SMS, alerts pushed into customer systems, premium-tier billing. |
| D-03 | Two customer roles: customer admin and customer viewer. |
| D-04 | AWS only, UK region (eu-west-2). |
| D-05 | Zero tolerance: one customer seeing, or being able to work out, another customer's data. |
| D-06 | Success metric: warned-ahead rate, the % of planned maintenance windows where every affected customer was **warned** (D-08) at least 48 hours before the start. Target 98%. |
| D-07 | Customers see plant **availability** only (generators running, % of capacity available), never plant output in MW or any figure from which another customer's load can be worked out. |
| D-08 | "Warned" means a warning was **delivered** (email accepted without bounce, or in-app message stored) to at least one alert recipient at the customer at least 48 hours before the window starts. Bounced or failed messages don't count. |
| D-09 | The Airon admin maintains the supply topology (which generators and feeds supply each customer). The system suggests affected customers for maintenance and outages; the operator confirms. |
| D-10 | Two environments: **dev** (£50 a month idle ceiling, no availability target) and **production** (availability and recovery targets apply). Separate AWS accounts. |

## Open assumptions and decisions

| ID | Question | Recommended answer | Status |
|---|---|---|---|
| AS-02 | Who publishes maintenance windows? | Engineers request; operators review and publish. | Open |
| AS-03 | Can a window be published less than 48 hours ahead? | Only as "urgent" with a reason; counts as a miss for the warned-ahead rate. | Open |
| AS-04 | Sites per customer? | One site per customer organisation in release one. | Open |
| AS-05 | Are 48-hour warnings standard or premium? | Standard (in-app and email); premium becomes SMS and push later. | Open |
| AS-06 | Are the NFR targets right? | As written in this document. | Open |
| AS-07 | Retention periods? | Raw readings 90 days; hourly aggregates, history and audit log 2 years; notification and application logs 1 year. | Open |
| AS-08 | 24-hour reminder before each window? | Yes, in addition to the 48-hour warning. | Open |
| AS-09 | Production cost ceiling? | To be set by Charlie. | Open |
| Q1 | Who needs multi-factor authentication? | Everyone. Staff use passkeys or security keys; customers may use an authenticator app or passkeys. | Open |
| Q2 | How is the production admin portal exposed? | Only from approved networks (allow-list or VPN) plus MFA. Dev may skip the network restriction. | Open |
| Q3 | Can one operator declare an outage alone? | Yes, with step-up MFA, confirmation of affected customers, an instant alert to a second staff member and a "declared in error" rescind. | Open |
| Q4 | Do maintenance engineers see customer feed readings? | No. Generator readings only. | Open |

## Users

| ID | User | Front end |
|---|---|---|
| P1 | Customer admin | Customer website |
| P2 | Customer viewer | Customer website |
| P3 | Control room operator (Airon) | Admin portal |
| P4 | Maintenance engineer (Airon) | Admin portal |
| P5 | Airon admin | Admin portal |
| S1 | Readings simulator (system) | None |

## Role–permission matrix

V = view, C = create, U = change, D = delete, cancel or deactivate. "Own" = the user's own customer organisation only. — = no access.

| Data or function | Customer admin | Customer viewer | Operator | Engineer | Airon admin |
|---|---|---|---|---|---|
| Customer organisation, site, contracted capacity, tier | V (own) | V (own) | V | V | V C U D |
| Customer users | V C U D (own) | — | — | — | V C U D |
| Alert recipients | V U (own) | — | V | — | V |
| Own feed supply readings | V (own) | V (own) | V | — (if Q4 agreed) | — |
| Plant availability | V | V | V | V | V |
| Generator readings | — | — | V | V | — |
| Supply topology | — | — | V | V | V C U D |
| Maintenance requests | — | — | V U (review) | V C U (own, before review) | V |
| Published maintenance windows | V (affecting own) | V (affecting own) | V C U D | V | V |
| Outages | V (affecting own) | V (affecting own) | V C U D | V | V |
| Notification log | — | — | V | — | V |
| History and CSV export | V (own) | V (own) | — | — | — |
| Staff users and roles | — | — | — | — | V C U D |
| Audit log | — | — | — | — | V |
| Warned-ahead report | — | — | V | — | V |

## Security principles (for the architect and builders)

| ID | Principle |
|---|---|
| SP-01 | Customer organisation and role come only from the verified token on the server, never from anything the client sends. Every read and write is checked at object level. Deny by default. |
| SP-02 | Customer separation is enforced twice: in the API **and** in the data layer, so one bug can't leak data. |
| SP-03 | Separation applies to every output: screens, API responses, live update channels, emails, in-app messages, exports, caches, logs, error messages, and figures from which another customer's data can be worked out. |
| SP-04 | Readings flow one way, from the plant (simulator now) into the platform. The platform can never send commands to plant systems. |
| SP-05 | Customer and staff identities are separate; the admin portal is a separate application on a separate hostname, not reachable through the customer website. |
| SP-06 | Actions that change what customers are told need strong authentication at the moment of action and are fully audited. |
| SP-07 | Notifications are authenticated: sent from Airon's domain with email authentication enforced; the in-app view is the authoritative record. |
| SP-08 | Dev and production are separate AWS accounts; no long-lived credentials; secrets in a managed secrets store; keys in a managed KMS; UK region only. |
| SP-09 | The audit log can't be changed or deleted by the application or its operators. |
| SP-10 | Every customer-owned record and file carries its organisation from creation, so later AI features can be filtered server-side. |
| SP-11 | Coding agents and pipelines have least privilege: no production credentials; all changes through reviewed pull requests. |
| SP-12 | Everything is built as infrastructure as code and scanned before it ships. |

---

## Functional requirements

| ID | Requirement | Users | Epic | Priority | Status |
|---|---|---|---|---|---|
| FR-01 | Customer users sign in to the customer website; Airon staff sign in to a separate admin portal. A customer account can't sign in to the admin portal. | All | EP-01 | Must | Agreed |
| FR-02 | Every user belongs to exactly one organisation: one customer organisation, or Airon. | All | EP-01 | Must | Agreed |
| FR-03 | Roles: customer admin, customer viewer, control room operator, maintenance engineer, Airon admin. Each role can do exactly what the role–permission matrix allows, and nothing else. | All | EP-01 | Must | Draft |
| FR-04 | A customer admin can invite users into their own organisation only, set them as admin or viewer, and remove them. The last customer admin can't be removed. | P1 | EP-01 | Must | Agreed |
| FR-05 | A removed or deactivated user loses access immediately, including any open sessions. | P1, P5 | EP-01 | Must | Draft |
| FR-06 | An Airon admin can create a customer organisation with its site and contracted capacity, and invite its first customer admin. | P5 | EP-07 | Must | Open (AS-04) |
| FR-07 | An Airon admin can create, change and deactivate staff users and their roles, including when a member of staff leaves. No user can change their own role. | P5 | EP-07 | Must | Draft |
| FR-08 | The customer website shows the organisation's own feed status (normal / reduced / off), delivered vs contracted power, and the time of the last reading. | P1, P2 | EP-02 | Must | Agreed |
| FR-09 | The customer website shows plant availability (generators running, % of capacity available), the same for every customer. It never shows plant output in MW or any figure from which another customer's load can be worked out. | P1, P2 | EP-02 | Should | Agreed |
| FR-10 | If readings are older than the NFR-02 limit, the screen shows "data delayed" instead of a status. | P1, P2, P3 | EP-02 | Must | Draft |
| FR-11 | The admin portal shows live readings for every generator and every customer feed, with stale readings flagged. | P3, P4 | EP-02 | Must | Draft |
| FR-12 | Supply screens update without a manual page refresh. | P1, P2, P3 | EP-02 | Must | Draft |
| FR-13 | A readings simulator produces realistic simulated generator output and per-customer feed readings, including faults and drops, at the volume in NFR-21. | S1 | EP-02 | Must | Agreed |
| FR-14 | A maintenance engineer can request a maintenance window: generators, start, end, reason. An engineer can't publish it. | P4 | EP-03 | Must | Open (AS-02) |
| FR-15 | A control room operator can review, change, publish or reject a requested window. Before publishing, the portal suggests the affected customers from the supply topology (FR-33) and the expected impact on each (reduced or no supply); the operator confirms or adjusts. | P3 | EP-03 | Must | Open (AS-02) |
| FR-16 | A window can't be published less than 48 hours before its start unless marked "urgent" with a reason. | P3 | EP-03 | Must | Open (AS-03) |
| FR-17 | Customers see upcoming windows affecting their own site: start, end (UK time), expected impact, and status (scheduled, changed, cancelled, in progress, complete). | P1, P2 | EP-03 | Must | Agreed |
| FR-18 | A changed or cancelled window stays visible to affected customers with what changed; it's never silently removed. | P1, P2 | EP-03 | Must | Draft |
| FR-19 | A control room operator can declare an outage, confirm the affected customers (suggested from the supply topology, FR-33), set status (declared, investigating, restoring, restored) and estimated restoration time, post updates and close it. | P3 | EP-04 | Must | Agreed |
| FR-20 | Customers see each active outage affecting their own site as a timeline of status changes and updates, with the current estimated restoration time. | P1, P2 | EP-04 | Must | Agreed |
| FR-21 | More than one outage or window can be active at once, and each customer sees only those affecting them. | P1, P2, P3 | EP-04 | Must | Draft |
| FR-22 | When a window is published, every alert recipient at each affected customer gets an email and an in-app message. | P1, P2 | EP-05 | Must | Open (AS-05) |
| FR-23 | When a window is changed or cancelled, the same recipients are told what changed. | P1, P2 | EP-05 | Must | Draft |
| FR-24 | A reminder is sent 24 hours before a window starts. | P1, P2 | EP-05 | Should | Open (AS-08) |
| FR-25 | When an outage is declared and when it's restored, every alert recipient at each affected customer gets an email and an in-app message. Interim updates are optional per user. | P1, P2 | EP-05 | Must | Draft |
| FR-26 | A customer admin chooses which of their organisation's users receive alerts. Recipients must belong to the same organisation. | P1 | EP-05 | Must | Agreed |
| FR-27 | Every notification is logged with recipient, channel, time sent and delivery status (delivered, bounced, failed). | P3, P5 | EP-05 | Must | Draft |
| FR-28 | Operators see, per window and per outage, who was notified and any failed deliveries. | P3 | EP-05 | Must | Draft |
| FR-29 | Customers see 12 months of their own outage and maintenance history, filterable by type and date, and can export it to CSV. | P1, P2 | EP-06 | Should | Draft |
| FR-30 | Every sign-in and every create, change and delete is recorded in an audit log: who, what, when, which record. | All | EP-08 | Must | Agreed |
| FR-31 | An Airon admin can view and search the audit log; nobody can edit or delete entries. | P5 | EP-08 | Must | Draft |
| FR-32 | The admin portal reports the warned-ahead rate per month using the D-08 definition; urgent windows count as misses. | P3, P5 | EP-06 | Should | Open (AS-03) |
| FR-33 | An Airon admin maintains the supply topology: generators, feeds, and which feeds and generators supply each customer site. Every change is audited. | P5 | EP-07 | Must | Agreed |
| FR-34 | Invitations expire after 7 days and can be re-sent. | P1, P5 | EP-01 | Must | Draft |
| FR-35 | Every user can reset their own password through a self-service flow. | All | EP-01 | Must | Draft |
| FR-36 | Maintenance and outage notifications are sent even if the customer website is unavailable; operators can still trigger them from the admin portal. | P3 | EP-05 | Must | Draft |
| FR-37 | Each customer organisation has a service tier (standard in release one; premium later) set by the Airon admin, so premium features can be switched on later without redesign. | P5 | EP-07 | Should | Draft |
| FR-38 | Customers never see which other customers are affected by a maintenance window or outage, or how many. | P1, P2 | EP-04 | Must | Agreed |

---

## Data requirements

| ID | Requirement | Epic | Priority | Status |
|---|---|---|---|---|
| DR-01 | Every customer-owned record and file carries its customer organisation from the moment it's created, ready for later features such as the AI assistant. | EP-09 | Must | Agreed |
| DR-02 | Every data entity has an owner and a classification as set out in the table below. | EP-09 | Must | Draft |
| DR-03 | The supply topology (FR-33) is the single source for which customers are affected by a generator or feed. | EP-07 | Must | Agreed |

**Data ownership and classification (DR-02)**

Classifications: **Customer confidential** (only that customer and authorised Airon staff), **Airon internal** (Airon staff only), **Shared with customers** (all signed-in customers).

| Entity | Owner | Classification |
|---|---|---|
| Customer organisation, site, contracted capacity | Customer | Customer confidential |
| Service tier | Airon | Airon internal |
| Customer users and alert recipients (simulated personal data) | Customer | Customer confidential |
| Customer feed readings | Customer | Customer confidential |
| Generator readings | Airon | Airon internal |
| Plant availability | Airon | Shared with customers |
| Supply topology | Airon | Airon internal |
| Maintenance requests | Airon | Airon internal |
| Published maintenance windows and outages | Airon | Each affected customer sees only its own view (FR-38) |
| Notification log | Airon (about customer recipients) | Customer confidential |
| Audit log | Airon | Airon internal |
| Staff users | Airon | Airon internal |

---

## Non-functional requirements

| ID | Area | Requirement and target | How it's tested | Epic | Priority | Status |
|---|---|---|---|---|---|---|
| NFR-01 | Availability | **Production** customer website: unplanned downtime no more than 4 hours a month (about 99.5%). No availability target for dev. | Uptime monitoring, monthly report | EP-09 | Must | Open (AS-06) |
| NFR-02 | Data freshness | Readings on screen no more than 60 seconds old; "data delayed" shown after 5 minutes. | Simulator test with a delayed feed | EP-02 | Must | Open (AS-06) |
| NFR-03 | Warning lead time | 100% of non-urgent windows warned (D-08) at least 48 hours before start; warned-ahead rate at least 98%. | Warned-ahead report | EP-05 | Must | Agreed |
| NFR-04 | Notification speed | Outage declared or restored → email and in-app message sent within 2 minutes (95th percentile). | Load test on the notification path | EP-05 | Must | Open (AS-06) |
| NFR-05 | Performance | Pages load within 2 seconds (95th percentile) at release-one load. | Performance test | EP-10 | Should | Open (AS-06) |
| NFR-06 | Scale | Release one: 2 customer organisations, about 50 customer users, 20 staff. Must reach 200 organisations and 5,000 users without redesign. | Architecture review, load test | EP-09 | Must | Open (AS-06) |
| NFR-07 | Customer separation | Zero cross-customer exposure through any screen, API response, export, notification, log, error message, or figure from which another customer's data can be worked out. | Separation and inference tests on every release | EP-10 | Must | Agreed |
| NFR-08 | Audit | 100% of sign-ins and writes logged; the log is read-only and tamper-evident. | Audit tests | EP-08 | Must | Draft |
| NFR-09 | Data residency | All data stored and processed in the UK AWS region. | Configuration review | EP-09 | Must | Agreed |
| NFR-10 | Retention and deletion | Retention as AS-07; data deleted automatically at the end of its period; a customer organisation's data can be fully deleted on request. | Configuration review, deletion test | EP-09 | Should | Open (AS-07) |
| NFR-11 | Recovery | **Production**: lose at most 1 hour of data (recovery point); service back within 4 hours (recovery time). | Restore test | EP-09 | Should | Open (AS-06) |
| NFR-12 | Accessibility | WCAG 2.2 AA on both front ends. | Accessibility audit | EP-10 | Must | Draft |
| NFR-13 | Mobile | Customer website fully usable on a phone over mobile data (shift supervisors on the floor, sites without power). | Device test | EP-10 | Must | Draft |
| NFR-14 | Time handling | All times shown in UK time and correct across clock changes; 48-hour checks calculated in UTC. | Tests across a clock change | EP-03 | Must | Draft |
| NFR-15 | Browsers | Current Chrome, Edge and Safari, desktop and mobile. | Test matrix | EP-10 | Should | Draft |
| NFR-16 | Observability | Staff can see service health, error rate and notification delivery rate. | Dashboard review | EP-08 | Must | Draft |
| NFR-17 | Cost (dev) | **Dev** environment costs no more than £50 a month when idle. | AWS cost report | EP-09 | Should | Agreed |
| NFR-18 | Simulated data only | No real personal data, real plant data or real company names anywhere in the system. | Data review before each release | EP-09 | Must | Agreed |
| NFR-19 | Rebuild | Any environment can be rebuilt from the repo within 1 working day. | Rebuild a clean environment | EP-09 | Should | Draft |
| NFR-20 | Cost (production) | Production costs no more than a monthly ceiling set by Charlie. | AWS cost report | EP-09 | Should | Open (AS-09) |
| NFR-21 | Reading volume | One reading per generator and per customer feed every 10 seconds. Release one: 4 generators, 2 customer feeds. Design for 20 generators and 200 feeds without redesign. | Simulator load test | EP-02 | Must | Draft |
| NFR-22 | Operations | Platform support in UK business hours (Mon–Fri 08:00–18:00); incidents acknowledged within 1 hour in support hours; planned platform maintenance announced to customers at least 5 working days ahead. | Runbook review, incident drill | EP-08 | Should | Draft |
| NFR-23 | Vulnerability management | Critical vulnerabilities fixed within 7 days, high within 30 days, in production. | Monthly vulnerability report | EP-09 | Must | Draft |
| NFR-24 | Security monitoring | Alerts staff act on for: repeated refused cross-organisation access, unusual export volume, mass notifications, staff sign-ins from new locations. | Detection test per use case | EP-08 | Must | Draft |
| NFR-25 | Incident response | Documented incident plan, including a customer-data exposure playbook and customer communication; tested once per release. | Tabletop exercise | EP-08 | Must | Draft |
| NFR-26 | Log retention | Notification log and application logs kept 1 year; audit log 2 years. Logs contain no secrets or passwords. | Configuration review; log scan | EP-08 | Should | Open (AS-07) |
| NFR-27 | Deletion completeness | Deleting a customer organisation removes its data from stores, caches and search indexes; backups age out within 35 days. | Deletion test | EP-09 | Should | Draft |
| NFR-28 | Security testing | Separation, authorisation-matrix and inference tests run on every release; an independent pen test before production go-live. | Test reports | EP-10 | Must | Draft |

---

## Security requirements

All security requirements carry the Jira label `security-req`. Standards are named by section, not version-specific clause numbers.

| ID | Requirement | Links | Acceptance test | Epic | Priority | Status | Standard |
|---|---|---|---|---|---|---|---|
| SR-01 | The API takes organisation and role only from the verified token; any organisation or role value sent by the client is ignored. | FR-02, SP-01 | Given a Northvale token, when the request names Harrowgate's ID, then the response contains only Northvale data or is refused. | EP-01 | Must | Draft | OWASP API Top 10 API1; ASVS Authorization |
| SR-02 | Every read and write checks the caller's organisation and role against the role–permission matrix at object level; customer data is also filtered in the data layer. | FR-03, NFR-07, SP-02 | Automated matrix test: every role × every endpoint × own and other organisation; all cross-organisation calls refused. A data-layer query without organisation context returns nothing. | EP-01 | Must | Draft | ASVS Authorization; OWASP API Top 10 API1, API3, API5 |
| SR-03 | No screen, API, export or notification exposes plant output, other customers' names or counts, or any figure from which another customer's load can be derived. | D-07, FR-09, FR-38 | Inference test: no customer-visible figure changes when another customer's simulated load changes. | EP-10 | Must | Draft | LINDDUN (linking); ASVS Data Protection |
| SR-04 | All users sign in with MFA. | FR-01 | Sign-in without a second factor is refused for every role. | EP-01 | Must | Open (Q1) | ASVS Authentication; NCSC |
| SR-05 | Staff use phishing-resistant MFA (passkeys or security keys); the production admin portal is reachable only from approved networks. | FR-01, SP-05 | Admin portal unreachable from an unapproved address; staff sign-in with SMS or TOTP refused. | EP-01 | Must | Open (Q1, Q2) | ASVS Authentication; NIST SP 800-207 |
| SR-06 | Sessions expire after inactivity (staff 15 minutes, customers 30 minutes) and are revoked at once when a user is removed, deactivated or changes password. | FR-05 | A removed user's existing session is refused on the next request; idle sessions expire on time. | EP-01 | Must | Draft | ASVS Session Management |
| SR-07 | Password reset never bypasses MFA, gives the same response whether or not an account exists, uses single-use links valid for 30 minutes, and is rate-limited. | FR-35 | Known and unknown emails get identical responses; a used link is refused; reset alone doesn't grant access without MFA. | EP-01 | Must | Draft | ASVS Authentication |
| SR-08 | Invitations are single-use, expire after 7 days, are bound to the invited email, and the organisation is set by the server. | FR-34 | Reused or expired invite refused; invite accepted with a different email refused. | EP-01 | Must | Draft | ASVS Authentication |
| SR-09 | Adding or restoring a customer admin by Airon requires verification with the customer's contract contact on record, recorded in the audit log. | FR-06 | The admin portal won't complete the action without a recorded verification reference. | EP-07 | Must | Draft | NCSC; process control |
| SR-10 | Live update channels are authorised server-side on subscribe and on every message; channel names are never chosen by the client. | FR-12 | A Northvale client subscribing to any other channel is refused; fan-out test confirms no cross-organisation delivery. | EP-02 | Must | Draft | ASVS Authorization; OWASP API Top 10 API1 |
| SR-11 | Publishing, changing or cancelling a window, and declaring, updating or restoring an outage, require step-up authentication, show the affected customer list for confirmation, and alert a second staff member at once. A "declared in error" rescind notifies affected customers. | FR-15, FR-19 | Action refused without fresh MFA; second staff member alerted within 1 minute; rescind sends a correction to each recipient. | EP-04 | Must | Open (Q3) | Separation of privilege |
| SR-12 | All user- and staff-entered text is encoded on output; a strict content security policy is set on both front ends. | FR-15, FR-19, FR-20 | Script payloads in every free-text field render as text; security headers present. | EP-10 | Must | Draft | ASVS Encoding and Sanitization; Web Frontend Security |
| SR-13 | Customer-facing free text is checked for other customers' names before publishing; a match blocks publishing until the text is changed. | FR-19, FR-38 | An update containing another customer's name is blocked. | EP-04 | Must | Draft | LINDDUN (disclosure) |
| SR-14 | CSV exports are scoped server-side to the user's organisation and escape cells starting with formula characters. | FR-29 | Export never contains another organisation's rows; formula payloads export as text. | EP-06 | Must | Draft | ASVS Encoding and Sanitization |
| SR-15 | Each notification is sent per recipient and contains only that customer's view; no shared recipient lists. | FR-22–FR-25 | A notification to two affected customers produces separate messages with no reference to the other. | EP-05 | Must | Draft | ASVS Data Protection |
| SR-16 | Email is sent from Airon's domain with SPF, DKIM and DMARC at reject; emails link only to Airon's known domain and never ask for credentials. | FR-22–FR-25 | DNS checks pass; templates contain no credential prompts or third-party links. | EP-05 | Must | Draft | NCSC email security |
| SR-17 | The audit log also records exports, staff views of customer-confidential data, topology changes and every refused access attempt. | FR-30, FR-33 | Each listed event produces an audit entry with user, organisation, record and time. | EP-08 | Must | Draft | ASVS Security Logging |
| SR-18 | The audit log is write-once: the application and its operators can append but can't change or delete entries; integrity can be verified. | NFR-08 | Attempts to modify or delete an entry with application and staff credentials fail; an integrity check detects tampering. | EP-08 | Must | Draft | ASVS Security Logging |
| SR-19 | Rate limits on sign-in, password reset, invitations, exports and notification sending, per user, per organisation and per source address. | FR-29, FR-34, FR-35 | Limits enforced in a load test; legitimate traffic unaffected. | EP-09 | Must | Draft | OWASP API Top 10 API4 |
| SR-20 | Readings are accepted only from an authenticated, authorised source; the ingestion path is never publicly writable. | FR-13, SP-04 | Unauthenticated or unknown-source readings are rejected and logged. | EP-02 | Must | Draft | IEC 62443 (zones and conduits); ASVS API |
| SR-21 | Every customer has at least one supply link; topology changes are audited and shown to operators; a customer with no supply link is flagged in the admin portal. | FR-33 | Removing a customer's last feed is blocked or flagged; the change appears in the audit log. | EP-07 | Must | Draft | Integrity control |
| SR-22 | Error messages shown to users contain no stack traces, internal identifiers or other organisations' data. | NFR-07 | Forced errors return generic messages with no internal detail. | EP-09 | Must | Draft | ASVS Security Logging and Error Handling |
| SR-23 | Separate AWS accounts for dev and production; least-privilege roles per service; no long-lived access keys; secrets in a managed store; keys in a managed KMS; storage public access blocked. | D-10, SP-08 | Configuration scan against the CIS AWS Foundations Benchmark shows no high findings; no IAM user access keys exist. | EP-09 | Must | Draft | CIS AWS Foundations; AWS Well-Architected Security |
| SR-24 | Main branch protected; changes only through reviewed pull requests; coding agents have no production credentials. | SP-11 | Direct push to main refused; pipeline credentials can't reach production outside the release job. | EP-09 | Must | Draft | NIST SSDF |
| SR-25 | The pipeline blocks merges on: secrets found, high or critical dependency vulnerabilities, high SAST findings, failed infrastructure-as-code checks, failed separation tests. | SP-12 | A planted test secret and a known vulnerable dependency each block the merge. | EP-09 | Must | Draft | NIST SSDF; OWASP SAMM |
| SR-26 | Dependencies pinned; an SBOM produced per build; build artefacts signed. | SP-11 | SBOM present for each release; unsigned artefact rejected at deploy. | EP-09 | Should | Draft | SLSA |
| SR-27 | Infrastructure built only from code in the repo; manual changes in production are detected. | NFR-19, SP-12 | Drift detection reports any console change. | EP-09 | Should | Draft | NIST SSDF |
| SR-28 | Encryption in transit (TLS 1.2 or higher) everywhere, and at rest for every data store and backup. | NFR-09 | Configuration scan shows no unencrypted store or plain-text endpoint. | EP-09 | Must | Draft | ASVS Secure Communication; Data Protection |

---

## Suggested Jira epics

To be mapped onto the existing Airon HQ epics (keep, change, add or retire) when the backlog is created.

| Epic | Name | Covers |
|---|---|---|
| EP-01 | Identity and access | Sign-in, roles, customer user management, invitations, password reset, MFA, sessions |
| EP-02 | Live supply and readings | Customer supply view, plant availability, admin readings, live updates, simulator, ingestion |
| EP-03 | Planned maintenance | Requests, review and publishing, customer maintenance view, changes and cancellations, time handling |
| EP-04 | Outages | Declaring, updating and restoring outages, customer outage view, free-text checks, step-up actions |
| EP-05 | Notifications | Email and in-app warnings, reminders, delivery log, email authentication, warned-ahead tracking |
| EP-06 | Customer history and reporting | History, CSV export, warned-ahead report |
| EP-07 | Airon administration and topology | Customer onboarding, staff users, supply topology, service tier |
| EP-08 | Audit, logging and monitoring | Audit log, observability, security monitoring, incident response, operations |
| EP-09 | Platform, infrastructure and delivery | AWS accounts, data model, residency, retention, recovery, cost, pipeline, supply chain, encryption |
| EP-10 | Quality, separation and security testing | Separation and inference tests, output encoding, accessibility, mobile, browsers, performance, pen test |

**How these become tickets (later):** each functional requirement becomes a story under its epic, carrying its linked NFR and SR acceptance tests. Platform and testing requirements with no story become tasks. Every item keeps its requirement ID in the summary so it traces both ways. Security items carry the label `security-req`.

## Summary

| Type | Count | Agreed | Draft | Open |
|---|---|---|---|---|
| Functional (FR) | 38 | 13 | 18 | 7 |
| Data (DR) | 3 | 2 | 1 | 0 |
| Non-functional (NFR) | 28 | 5 | 14 | 9 |
| Security (SR) | 28 | 0 | 25 | 3 |
| **Total** | **97** | **20** | **58** | **19** |

## Out of scope for release one

AI chatbot and chatbot-raised maintenance tickets; partner API; SMS; alerts pushed into customer systems; premium-tier billing; more than one site per customer organisation.
