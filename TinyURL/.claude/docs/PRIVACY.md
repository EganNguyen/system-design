# Data Privacy & Compliance: TinyURL Service

> GDPR and CCPA mapping for the TinyURL Enterprise Link Management Infrastructure.
> Defines what personal data is collected, the legal basis for processing, data subject rights,
> the right-to-erasure flow, retention schedule, and third-party processor obligations.
> Required before any EU or California user traffic is processed.

---

## 1. Personal Data Inventory

### 1.1 What Counts as Personal Data

Under GDPR Article 4(1), personal data is any information that can identify a natural person
directly or indirectly. IP addresses are personal data. Pseudonymised data (hashed IPs) is
still personal data if re-identification is possible — however, one-way hashing with a
rotating daily salt makes re-identification computationally infeasible and reduces
regulatory risk significantly.

### 1.2 Data Map

| Data element | Stored in | Personal? | How collected | Lawful basis |
|---|---|---|---|---|
| Email address | `users.email` (PostgreSQL) | Yes — directly identifies | User registration | Contract (Art. 6(1)(b)) |
| Password hash | `users.password_hash` (PostgreSQL) | Derived | User registration | Contract (Art. 6(1)(b)) |
| Account plan tier | `users.plan` (PostgreSQL) | No | Set on registration / upgrade | Contract (Art. 6(1)(b)) |
| Submitted long URL | `urls.long_url` (PostgreSQL) | Potentially (if URL contains PII) | API write | Contract (Art. 6(1)(b)) |
| Short code | `urls.short_code` (PostgreSQL, Redis) | No | Generated | Contract |
| IP hash (click) | `click_events.ip_hash` (ClickHouse) | Pseudonymous | Analytics ingest | Legitimate interest (Art. 6(1)(f)) |
| Country code (click) | `click_events.country_code` (ClickHouse) | No — aggregated | MaxMind GeoIP lookup | Legitimate interest (Art. 6(1)(f)) |
| Referrer URL (click) | `click_events.referrer` (ClickHouse) | Potentially (if referrer URL contains PII) | HTTP Referer header | Legitimate interest (Art. 6(1)(f)) |
| User agent (click) | `click_events.user_agent` (ClickHouse) | Pseudonymous (device fingerprinting risk) | HTTP User-Agent header | Legitimate interest (Art. 6(1)(f)) |
| Audit log entry | `audit_log` (PostgreSQL) | Yes — contains `user_id` + `ip_hash` | System-generated | Legal obligation (Art. 6(1)(c)) |
| JWT / session token | Redis token store (ephemeral) | Yes — maps to `user_id` | Auth flow | Contract (Art. 6(1)(b)) |

### 1.3 Data We Explicitly Do Not Collect

- **Raw IP addresses** — hashed at the ingest boundary before any persistence (SEQUENCES.md §6).
- **City, region, or precise geolocation** — country-level only (ISO 3166-1 alpha-2).
- **Device advertising IDs** (IDFA, GAID).
- **Cross-site tracking cookies** or third-party tracking pixels.
- **Link destination page content** — we store the URL string, not page content.

---

## 2. Lawful Basis Summary (GDPR Article 6)

| Processing activity | Lawful basis | Notes |
|---|---|---|
| User account management | Contract (Art. 6(1)(b)) | Necessary to provide the service |
| URL shortening and redirect | Contract (Art. 6(1)(b)) | Core service delivery |
| Click analytics collection | Legitimate interest (Art. 6(1)(f)) | Balanced against data subject rights; LIA below |
| Audit logging | Legal obligation (Art. 6(1)(c)) | Fraud prevention, legal accountability |
| Safe Browsing URL check | Legitimate interest (Art. 6(1)(f)) | Abuse prevention; URL string only shared, no user PII |
| Transactional email | Contract (Art. 6(1)(b)) | Password reset, account notices |

### 2.1 Legitimate Interest Assessment (LIA) — Analytics

**Purpose:** Provide link owners with aggregate click metrics (total clicks, country breakdown,
referrer sources) to measure campaign performance.

**Necessity:** Analytics require recording that a click occurred. IP is needed only transiently
for deduplication and geo lookup — it is never stored.

**Balancing test:**
- Data is pseudonymised before storage (SHA-256 + rotating daily salt on IP).
- Country-level geo only — no city, no coordinates.
- Retention capped at 90 days.
- No cross-service tracking or profiling.
- Data subject can avoid collection by not clicking the link (no pre-existing relationship required).

**Conclusion:** Legitimate interest basis is defensible. A full DPIA is recommended if
processing exceeds 1M EU data subjects per month.

---

## 3. Data Subject Rights

### 3.1 GDPR Rights (Chapter III)

| Right | Article | Applicable | Response deadline |
|---|---|---|---|
| Right to be informed | Art. 13–14 | Yes — privacy notice on landing page | At point of collection |
| Right of access | Art. 15 | Yes | 30 days |
| Right to rectification | Art. 16 | Yes (email, account details) | 30 days |
| Right to erasure ("right to be forgotten") | Art. 17 | Yes — see §4 | 30 days |
| Right to restriction of processing | Art. 18 | Yes | 30 days |
| Right to data portability | Art. 20 | Yes (account data + owned URLs as JSON/CSV) | 30 days |
| Right to object | Art. 21 | Yes (for analytics processed under legitimate interest) | Immediate for opt-out |
| Rights re: automated decision-making | Art. 22 | No — no profiling that produces legal effects | N/A |

### 3.2 CCPA Rights (California Residents)

| Right | CCPA Section | How fulfilled |
|---|---|---|
| Right to know what is collected | § 1798.100 | Privacy notice + `/api/v1/users/me/data-export` endpoint |
| Right to know what is sold or shared | § 1798.115 | We do not sell personal data — disclosed in privacy notice |
| Right to delete | § 1798.105 | Same erasure flow as GDPR §4 below |
| Right to opt-out of sale | § 1798.120 | N/A — no data sale |
| Right to non-discrimination | § 1798.125 | No service degradation for exercising rights |
| Right to correct | § 1798.106 | Same as GDPR rectification |

---

## 4. Right-to-Erasure Flow (GDPR Art. 17 / CCPA § 1798.105)

A verified erasure request must complete within **30 days**. The flow is fully automated
via the account deletion endpoint and a cascading deletion job.

### 4.1 Trigger

```
DELETE /api/v1/users/me
Authorization: Bearer <access_token>
```

Or via a support ticket after email-based identity verification — same pipeline is triggered
once identity is confirmed.

### 4.2 Deletion Cascade (6 steps)

```
Step 1 — Revoke all sessions
  Redis: DEL refresh:{user_id}
  Redis: SET revoked:{jti} 1 EX <remaining_ttl>  for all active access tokens

Step 2 — Soft-delete all owned URLs
  UPDATE urls SET is_active = false WHERE user_id = $user_id
  Redis: DEL url:{short_code}  (pipelined, for each owned short_code)
  Effect: subsequent redirects return 410 Gone

Step 3 — Anonymise analytics records
  ClickHouse does not support row-level deletes natively — use mutations:
    Option A (preferred — if within retention window):
      ALTER TABLE click_events DELETE
      WHERE short_code IN (SELECT short_code FROM urls WHERE user_id = $user_id)
      → ClickHouse mutation (async, completes within minutes at typical scale)
    Option B (partition-level, faster if all data is in old partitions):
      DROP PARTITION for relevant monthly partitions where the user's data lives
      → Only viable if the entire partition belongs to this user or retention has passed

Step 4 — Delete user account record
  DELETE FROM users WHERE user_id = $user_id

Step 5 — Redact audit log (retain structure, remove identity)
  UPDATE audit_log
  SET user_id = 'REDACTED', ip_hash = 'REDACTED'
  WHERE user_id = $user_id
  Note: action, short_code, and timestamp are preserved for legal accountability.
        The user's identity is removed but the event record remains.

Step 6 — Confirm and log
  Send confirmation email to the address on file (last communication before deletion).
  INSERT INTO erasure_log (request_id, requested_at, completed_at, steps_completed)
  Do NOT store user_id in erasure_log — use a one-way hash of the email for
  deduplication only (prevents re-registration abuse check).
```

### 4.3 What Cannot Be Fully Deleted

| Data | Why retained | Legal basis |
|---|---|---|
| Redacted audit log entries | Fraud investigation, legal accountability | Legal obligation (Art. 6(1)(c)) |
| Billing / invoice records | Tax law (typically 7 years, jurisdiction-dependent) | Legal obligation |
| Anonymised aggregate analytics | No longer personal data once anonymised at aggregate level | N/A |

### 4.4 Deletion Verification Checklist

Within 30 days of the request, verify:
- [ ] `users` — no row exists for `user_id`
- [ ] `urls` — all rows for `user_id` have `is_active = false`
- [ ] `click_events` (ClickHouse) — no rows with `short_code` belonging to `user_id` remain
- [ ] Redis — no `refresh:{user_id}` key exists
- [ ] `audit_log` — `user_id` and `ip_hash` columns are `'REDACTED'` for all matching rows
- [ ] `erasure_log` — completion entry present with `completed_at` timestamp
- [ ] Confirmation email sent and delivery logged

---

## 5. Data Retention Schedule

| Data | Retention | Mechanism |
|---|---|---|
| User accounts | Until deletion request | Erasure cascade on `DELETE /api/v1/users/me` |
| Owned URLs | Until erasure request or 2 years post soft-delete | Soft-delete immediately; hard-delete via nightly cleanup |
| Click events (raw, ClickHouse) | **90 days** | `TTL clicked_at + INTERVAL 90 DAY DELETE` on table definition |
| Click daily aggregate MV | **1 year** | ClickHouse monthly partition drop |
| Audit log | 2 years (redacted after erasure) | PostgreSQL scheduled purge at 2-year mark |
| Refresh tokens | 30 days or until revoked | Redis key TTL |
| Access tokens | 15 minutes | Redis revocation list TTL |
| Erasure log | 3 years | Immutable append-only table; no delete |
| Kafka click events (in-flight) | 7 days (replay window) | Kafka topic `retention.ms` config |

---

## 6. Data Residency

### 6.1 EU Data (GDPR Chapter V)

Personal data of EU residents must not be transferred outside the EEA without a valid
legal mechanism. Required before processing any EU user traffic.

**Deployment requirements:**
- A dedicated **EU region** (e.g., `eu-west-1` or `eu-central-1`) hosts all EU user data.
- PostgreSQL, Redis, Kafka, and ClickHouse for EU traffic run in EU availability zones only.
- No EU personal data replicates to US or non-EEA regions without SCCs in place.

**Third-party processors handling EU data:**

| Processor | Data shared | Transfer mechanism |
|---|---|---|
| Google Safe Browsing | URL string only (no user PII) | Google Cloud DPA + SCCs |
| MaxMind GeoIP | Raw IP for lookup (not stored by MaxMind under their DPA) | MaxMind DPA |
| AWS | All data at rest and in transit | AWS DPA + SCCs |
| Monitoring vendor (Datadog/Grafana) | Metrics, logs (may contain IP hashes) | Vendor DPA + SCCs |

### 6.2 Data Residency Enforcement

- Each Kafka click event message is tagged with `region: EU` or `region: US` based on the
  origin API server's region label.
- Kafka topic routing enforces EU events to the EU ClickHouse cluster.
- Cross-region replication disabled for EU PostgreSQL and ClickHouse instances.
- Quarterly audit: synthetic EU user flow verified to produce no data in non-EU storage.

---

## 7. Third-Party DPA Checklist

All processors must have a signed DPA before EU traffic goes live.

| Processor | DPA signed | SCCs in place | Sub-processors reviewed |
|---|---|---|---|
| AWS | TBD | TBD | TBD |
| Google (Safe Browsing) | TBD | TBD | TBD |
| MaxMind (GeoIP) | TBD | TBD | TBD |
| Monitoring vendor | TBD | TBD | TBD |
| Alerting vendor (PagerDuty / OpsGenie) | TBD | TBD | TBD |

> All TBD items block EU traffic. Assign legal owner and target completion date.

Maintain a public sub-processor list at `/legal/sub-processors`. Provide 30 days notice
before adding any new sub-processor that processes EU personal data.

---

## 8. Privacy Notice Requirements

A privacy notice must be published and linked from the landing page before EU or
California traffic is accepted. Minimum required content (GDPR Art. 13):

- [ ] Identity and contact details of the data controller
- [ ] DPO contact details (required if large-scale systematic monitoring)
- [ ] Purposes and lawful basis for each processing activity (§2)
- [ ] Legitimate interests pursued (§2.1)
- [ ] Categories of personal data collected (§1.2)
- [ ] Recipients and categories of recipients
- [ ] Third-country transfers and safeguards (§6.1)
- [ ] Retention periods per data category (§5)
- [ ] How to exercise data subject rights (§3)
- [ ] Right to lodge a complaint with a supervisory authority
- [ ] Whether providing data is a contractual requirement and consequences of refusal

---

## 9. Data Breach Response (GDPR Art. 33–34)

### 9.1 72-Hour Timeline

| Hour | Action |
|---|---|
| 0 | Incident detected, contained. Incident Commander assigned. |
| 1 | Initial assessment: which data categories exposed, how many data subjects, which regions. |
| 4 | Decision: does breach meet GDPR notification threshold? (Likely to result in risk to rights and freedoms.) |
| 24 | Internal legal and DPO review of notification obligation. |
| **72** | **Hard deadline — notify supervisory authority** (e.g., ICO for UK, DPC for Ireland). |
| 72+ | If breach is "likely to result in high risk" to individuals: notify affected data subjects without undue delay. |

### 9.2 Supervisory Authority Notification Content (Art. 33(3))

- Nature of breach (categories and approximate number of data subjects and records)
- DPO / privacy contact details
- Likely consequences of the breach
- Measures taken or proposed to address and mitigate

### 9.3 Breach Severity Classification

| Severity | Example | Notify authority | Notify individuals |
|---|---|---|---|
| Low | Internal log briefly exposed, no external access confirmed | No | No |
| Medium | Hashed IP data exposed to wrong tenant via query bug | Yes (72h) | No (low individual risk) |
| High | User email + URL list exposed publicly | Yes (72h) | Yes |
| Critical | Password hashes or auth tokens exposed | Yes (72h) | Yes — immediately |

---

## 10. Open Compliance Actions

All items below block EU or California user traffic.

| Action | Blocking | Owner |
|---|---|---|
| Resolve retention conflict — update DATABASE_DESIGN.md §8 to 90 days | Yes | Engineering |
| Implement ClickHouse TTL (`INTERVAL 90 DAY DELETE`) on `click_events` | Yes | Engineering |
| Implement `DELETE /api/v1/users/me` erasure cascade (all 6 steps) | Yes | Engineering |
| End-to-end erasure flow test with deletion verification checklist | Yes | Engineering |
| Deploy dedicated EU region with data residency Kafka tagging | Yes | Infrastructure |
| Publish privacy notice on landing page | Yes | Legal / Product |
| Sign DPAs with all third-party processors | Yes | Legal |
| Appoint DPO or privacy contact | Yes (if > 1M EU subjects/month) | Legal |
| Publish sub-processor list at `/legal/sub-processors` | Yes | Legal |
| Conduct DPIA | Conditional (> 1M EU subjects/month) | Legal |

---

*Informed by: [DATABASE_DESIGN.md §8](DATABASE_DESIGN.md), [SECURITY.md §E1 §C3](SECURITY.md), [SEQUENCES.md §6](SEQUENCES.md), [LLD.md §5.2](LLD.md).*
