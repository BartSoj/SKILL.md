# STRIDE Checklist — Per-Element × Per-Category Enumeration Prompts

This reference file is read by the `/security` skill during Phase 3 (STRIDE-driven threat enumeration). It supplies the prompt questions the agent asks for every (DFD element × STRIDE category) cell. The goal is systematic coverage — every cell is either a `THREAT-NN` entry in SECURITY § 5 or an explicit dismissal with a one-line justification.

## How to use this checklist

For each element in § 4 Data Flow Diagram, walk through every STRIDE category below. For each prompt question, answer one of:

- **Threat exists** — write a `THREAT-NN` entry following the template in SECURITY § 5, with the prompt question as seed for the Description field. Extend with the concrete attack scenario.
- **Dismissed** — write `{Category} dismissed — {one-line justification tied to this element's properties}`. Valid dismissals: the element has no caller identity to impersonate (for Spoofing); the element is stateless and emits no authoritative logs (for Repudiation); the element handles only public data (for Information Disclosure). Invalid dismissals: "unlikely", "seems fine", or silence.

Never leave a cell unaddressed. Silent cells are exactly the attack surface STRIDE exists to surface.

---

## DFD element types

STRIDE analysis distinguishes four element types. Each has different threat applicability:

| Element type | What it is | Examples |
|--------------|-----------|----------|
| **Process** | A component that executes code | `APIGateway`, `AuthService`, `OrderProcessor`, `WebhookDispatcher` |
| **Data flow** | An edge between two elements carrying data | `Client → APIGateway`, `OrderProcessor → EventBus`, `WebApp ↔ CDN` |
| **Data store** | Persistent storage | PostgreSQL table `users`, S3 bucket `uploads`, Redis cache, Kafka topic |
| **External entity** | An actor or system outside the trust zone | End user, third-party webhook source, OAuth provider, CDN origin |

Not every STRIDE category applies to every element type. The applicability matrix below is a starting point; the prompts that follow are ordered by element type × category.

| | Spoofing | Tampering | Repudiation | Info Disclosure | DoS | Elevation |
|---|---|---|---|---|---|---|
| Process | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Data flow | — | ✓ | — | ✓ | ✓ | — |
| Data store | — | ✓ | ✓ | ✓ | ✓ | — |
| External entity | ✓ | — | ✓ | — | — | — |

A `—` entry is a strong default dismissal — state the default dismissal explicitly (e.g., "Spoofing dismissed — data flows have no identity of their own; spoofing of endpoints is covered under the endpoint process.") rather than omitting the cell.

---

## Processes (components that execute code)

### Spoofing of a process

Prompt questions:

1. Can an attacker impersonate the caller of this process? Under what authentication does the process accept requests? Can the authentication be forged, replayed, or downgraded?
2. Can an attacker impersonate this process to its downstream callers? Does this process authenticate itself to downstream services (mTLS, signed requests, service-account tokens)?
3. Does this process accept anonymous requests? If yes, which endpoints? Is anonymity a genuine requirement or a historical accident?
4. Can credentials presented to this process be stolen at rest or in transit (phishing, man-in-the-middle, log leakage)? What replay window exists?
5. Are there any trust-on-first-use paths (pairing, device registration, invite acceptance)? What prevents an attacker from racing to claim?

### Tampering with a process

Prompt questions:

1. Can an attacker modify input in transit to cause the process to take an unintended action? Is there integrity protection (TLS, signed request envelopes)?
2. Can an attacker tamper with configuration read by this process (env vars from a compromised config source, poisoned remote config, cached bad values)?
3. Can an attacker tamper with code or binaries loaded by this process (supply-chain attack on dependencies, code injection via plugin/extension mechanism, dynamic loading of untrusted modules)?
4. Can an attacker tamper with in-memory state via shared memory, race conditions, or speculative execution?
5. For processes that transform data before forwarding, can the transformation be bypassed (content-type confusion, alternate parser path, pre-validation bypass)?

### Repudiation by a process

Prompt questions:

1. Does this process take actions whose authoritative record matters (money movement, data deletion, permission grants)? Does it emit an audit log for every such action?
2. Are audit logs tamper-evident (append-only, signed, off-host)? Can the process itself delete or modify its own audit history?
3. Does the audit log carry enough context to attribute an action to a specific authenticated principal (user ID, session ID, source IP, timestamp)?
4. For long-running workflows or sagas, are compensating actions and rollbacks equally logged? Can a user claim "I never did that" plausibly?
5. Is log retention long enough to cover the claim window relevant to the business (regulatory, contractual, statute of limitations)?

### Information disclosure by a process

Prompt questions:

1. What data does this process read, process, or emit? Which of it is confidential per § 1 Assets?
2. Can an attacker trick this process into returning data it should not (IDOR, broken access control, verbose error messages, debug endpoints in production)?
3. Are there side channels (timing attacks on comparisons, response-size differences on auth failures, cache residue, error-message fingerprinting)?
4. Does this process log sensitive data by accident (forbidden fields per QUALITY § 1, stack traces with PII, query strings with tokens)?
5. Can an attacker enumerate resources by probing responses (user-exists vs user-not-found distinguishable; rate-limit vs block-list distinguishable)?

### Denial of service against a process

Prompt questions:

1. What resource exhaustion paths exist (CPU via expensive algorithms, memory via unbounded allocation, disk via log flooding, connections via slow-loris)?
2. Are there amplification paths (small request → expensive response, single call → many downstream calls, unbounded recursion)?
3. Does the process handle input with bounded complexity (max body size, max nesting depth, max array length, max string length)?
4. Can a single tenant or caller monopolise resources (no per-tenant quotas, no fair-share scheduling, shared thread pool)?
5. What happens when a downstream dependency is slow or unavailable (retry storm, thread-pool starvation, cascading timeouts)?

### Elevation of privilege within a process

Prompt questions:

1. Can an attacker with low privileges execute actions reserved for higher privileges (missing authorization check, wrong check, check-then-use TOCTOU)?
2. Can an attacker inject input that the process executes as code (command injection, SSTI, expression-language injection, deserialization of untrusted data)?
3. Can an attacker cross a tenant boundary (tenant-ID not enforced in query, shared caches, shared resources)?
4. Can an attacker escape a sandbox or jail (container escape, path traversal, SSRF to internal services)?
5. For admin or service-account paths, what prevents a compromised low-privilege account from escalating (weak role hierarchy, missing re-authentication for privileged operations, cached tokens that outlive role changes)?

---

## Data flows (edges between elements)

### Tampering with a data flow

Prompt questions:

1. Is the flow encrypted and integrity-protected in transit (TLS 1.2+, mTLS, signed payloads)?
2. Can an attacker with a network position on this flow modify the data? Are there intermediaries (load balancers, proxies, service meshes) that could be compromised?
3. For flows crossing untrusted networks (public internet, VPN transit, third-party provider backbones), what prevents tampering beyond TLS (e.g., end-to-end signed envelopes)?
4. Are compressed or encoded payloads vulnerable to tampering that decompresses into malicious content (zip bombs, compression side-channels)?

### Information disclosure on a data flow

Prompt questions:

1. What data classification travels on this flow (per § 3 Trust Boundaries)? Does the encryption match the classification?
2. Can an attacker passively observe the flow (shared-segment networks, compromised router, SSL-stripping proxy, WiFi sniffing)?
3. Are there metadata leaks beyond the payload (SNI revealing hostname, packet-size revealing content, timing revealing decisions)?
4. Does the flow carry data beyond the receiver's need-to-know (over-fetching from APIs, verbose responses to minimal requests)?

### Denial of service against a data flow

Prompt questions:

1. Can an attacker saturate the flow's bandwidth (flood, amplification attack, slow-drain)?
2. Are there per-connection or per-source limits on throughput?
3. For bidirectional flows, can backpressure propagate correctly, or does one side's slowness starve the other?

---

## Data stores (persistent storage)

### Tampering with a data store

Prompt questions:

1. Who has write access to this store (users, services, operators, backup processes)? Is access enforced at the store level (row-level security, IAM policies) or only at the application layer?
2. Can an attacker bypass the application layer and write directly to the store (SQL injection, compromised service account, stolen DB credentials, leaked connection string)?
3. Are mutations logged tamper-evidently? Can the attacker cover their tracks by also modifying the audit log?
4. For caches, can an attacker poison the cache with values that later serve other users (cache key collision, user-ID missing from cache key)?
5. For event logs and append-only stores, can an attacker truncate, rewrite history, or replay events?

### Repudiation via a data store

Prompt questions:

1. Does the store retain enough history to answer "who did this, when, and from where?" Are changes versioned or overwritten?
2. Can an administrator with legitimate write access modify historical records without trace? Is there a separate audit trail off-host?
3. For soft-delete patterns, does the tombstone carry attribution and timestamp?

### Information disclosure from a data store

Prompt questions:

1. Is the data encrypted at rest (transparent DB encryption, filesystem encryption, application-level field encryption for high-sensitivity columns)?
2. Can backup copies leak data (unencrypted snapshots, stale backups in untrusted storage, third-party backup providers)?
3. Can query-side attacks reveal data (SQL injection reading tables beyond intent, blind extraction via error-based or timing-based queries, LIKE queries enabling enumeration)?
4. Can adjacent tenants read each other's data via shared-pool patterns (RLS policy bug, missing tenant_id filter, cache-key collision)?
5. Are there operational side channels (DB logs carrying query text with PII, slow-query logs, telemetry exports)?

### Denial of service against a data store

Prompt questions:

1. Can a single caller exhaust connections, IOPS, or storage (unbounded queries, unbounded uploads, unbounded log writes)?
2. Can an attacker cause lock contention that starves other callers (long transactions, SELECT FOR UPDATE on hot rows, deadlock loops)?
3. For stores with quota semantics, can an attacker fill another tenant's quota (cross-tenant write leakage, missing quota enforcement)?
4. What happens if the store is unavailable? Does the application fail safely, fail open (serving stale or default data), or fail catastrophically?

---

## External entities (actors or systems outside the trust zone)

### Spoofing of an external entity

Prompt questions:

1. How does the system establish the external entity's identity (user credentials, third-party OAuth token, mTLS certificate, IP allowlist, HMAC-signed webhook)?
2. Can the identity be forged at the credential layer (password guessing, credential stuffing, token theft, certificate issuance bypass)?
3. For webhook or server-to-server integrations, is the source cryptographically authenticated, or is it only source-IP checked?
4. For OAuth delegations, is the token audience checked? Is the token scope honoured? Can a token for another client be replayed?
5. Can an attacker register as a legitimate-looking but adversarial external entity (sign-up abuse, domain look-alikes, sub-account abuse under a legitimate parent)?

### Repudiation by an external entity

Prompt questions:

1. Does the system log enough about incoming external requests to attribute them to the entity (source IP, user agent, authenticated identity, request IDs)?
2. For transactions with non-repudiation requirements (payments, contracts, deletions), is there a cryptographic non-repudiation signal (signed request, TLS client certificate, MFA challenge)?
3. Can the external entity plausibly claim "my account was compromised; I never sent that request"? What evidence does the system retain to counter that claim (device fingerprinting, behavioural signals, multi-factor at the time of the action)?
