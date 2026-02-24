# Heptaconn Protocol Specification (v0)

Status: Draft (v0)
Transport: TCP (newline-delimited JSON frames)
Encoding: UTF-8 JSON per line

---

## 1. Overview

Heptaconn is a framed connection protocol built around a seven-phase message loop. It provides:

- Explicit execution boundaries
- Deterministic upload sequencing
- Opaque payload transport
- Receipt-based operational metrics

Heptaconn does not inspect payload content. It enforces envelope and boundary rules only.
Heptaconn explicitly accounts for both served and non-served demand.

---

## 2. Transport Layer

- TCP is the transport.
- Each frame is one JSON object terminated by `\n`.
- No framing beyond newline separation.
- Connections may remain open for multiple frames.

Compression (e.g., gzip/zstd) applies to payload bytes only and must be declared in the manifest.

---

## 3. Frame Envelope

Each frame MUST contain:

```json
{
  "v": 0,
  "verb": "share",
  "job": "J_...",
  "seq": 1,
  "ttl_ms": 60000
}
```

### Required Fields

| Field   | Type   | Description |
|----------|--------|-------------|
| v        | int    | Protocol version (v0 only) |
| verb     | string | One of defined verbs |
| job      | string | Logical request identifier |
| seq      | uint64 | Monotonic per job |
| ttl_ms   | uint64 | Intended time budget |

Optional fields include `status`, `payload`, `payload_manifest`, `payload_chunk`, `receipts`, and error fields.

---

## 4. Verbs (v0)

Heptaconn defines exactly seven verbs:

| Verb       | Purpose |
|------------|----------|
| share      | Initiate / send |
| clarify    | Accept / request correction |
| explain    | Expand or provide detail |
| revisit    | Adjust prior state |
| retry      | Alternate attempt |
| interpret  | Semantic framing |
| confirm    | Final close of loop |

Only `share`, `clarify`, and `confirm` are required in v0 server implementations.

---

## 5. Status Values

Status is optional and applies primarily to `clarify` and `confirm`.

| Status | Meaning |
|--------|---------|
| ok     | Accepted |
| need   | More information required |
| yield  | Deferred |
| done   | Completed |
| abort  | Terminated |

---

## 6. Upload Protocol (v0)

### 6.1 Manifest Phase

Client sends a manifest frame (reference v1 implementation uses `clarify` as the verb) containing `payload_manifest`:

```json
{
  "payload_manifest": {
    "payload_id": "P_1",
    "content_type": "application/json",
    "content_encoding": "identity",
    "cipher": "none",
    "total_bytes": 0,
    "chunk_bytes": 8192,
    "chunks": 3,
    "sha256": "..."
  }
}
```

Server responds with exactly one admission decision:
- `clarify + ok` (phase = "pre_booked") → capacity reserved for this connection
- `clarify + need` → correction required

---

### 6.2 Chunk Phase

Client sends ordered chunk frames (reference v1 implementation uses `clarify` as the verb) containing `payload_chunk`.

Server enforces:

- Strict in-order delivery (`index` must match expected)
- SHA256 per chunk verification
- Optional full payload hash verification

Server replies only when necessary:
- `clarify + need` → index/hash correction required
- (no reply on successful in-order chunk; client continues streaming)

---

### 6.3 Completion Phase

When all chunks are accepted:
- Server sends `confirm + done`
This frame represents the terminal closing credit of the request attempt.

Exactly one terminal disposition MUST be emitted per request attempt. Servers MUST NOT emit multiple terminal frames for the same attempt.

---

## 7. Receipts

### 7.1 Core Transport Receipts (v1)

The following receipts are guaranteed by the v1 transport implementation:

- `conn.outcome`
- `conn.outcome_reason`
- `cdr.queue.ms`
- `cdr.backend.ms`
- `cdr.bytes_in`

These receipts are transport-level and business-facing. They MUST NOT depend on payload inspection.

### 7.2 Optional Backend Receipts (Non-Normative)

Backends MAY emit additional execution receipts, such as:

- `cdr.ingest.tok`
- `cdr.output.tok`
- `cdr.stop_reason`
- `cdr.run.ns_total`

These are optional extensions and are not required for served/not-served accounting.

Additional operational receipt keys (v0 extension):

- `conn.retry_hint_ms`
- `conn.queue_depth`

`conn.outcome` represents the terminal disposition of a request attempt.

Defined outcome codes (uint64):

| Code | Meaning |
|------|---------|
| 1 | served (confirm + done) |
| 2 | rejected (confirm + abort) |
| 3 | deferred (clarify + yield) |
| 4 | timeout |
| 5 | dropped |

`conn.outcome_reason` refines the cause. Recommended reason codes (uint64):

| Code | Reason |
|------|--------|
| 1 | busy |
| 2 | invalid_envelope |
| 3 | invalid_manifest |
| 4 | invalid_chunk |
| 5 | quota_exceeded |
| 6 | ticket_missing |
| 7 | ticket_expired |
| 8 | backend_error |
| 9 | timeout (backend or idle) |

When `status = yield`, servers SHOULD emit `conn.outcome = 3` and MAY include `conn.retry_hint_ms` and `conn.queue_depth`.



### 7.3 Admission Timing (v1)

cdr.queue.ms (uint64) measures admission latency.

Definition (v1): milliseconds from TCP Accept() to the first admission decision frame for the request attempt.

Admission decision is defined as:
- clarify + ok (phase = "pre_booked")
- clarify + yield
- confirm + abort (if rejected at admission time)

This value MUST reflect transport-side waiting only. It MUST NOT include backend execution time.

Backend execution latency is measured separately via `cdr.backend.ms`.
`cdr.backend.ms` measures time spent inside the backend submission boundary only and MUST NOT include admission latency.

Backend phase markers (e.g., `backend_error phase=...`) are diagnostic-only and are not part of the wire schema or receipt taxonomy.

Servers MUST attach cdr.queue.ms exactly once on the terminal disposition frame of the request attempt.

Receipts MUST be emitted exactly once for each terminal request disposition.

---

### 7.4 Operational Outcome Buckets (Derived Classification)

For operational dashboards and capacity analysis, request outcomes MAY be grouped into five higher-level buckets.

These buckets are derived from `conn.outcome` and `conn.outcome_reason`. They do not change wire semantics.

| Bucket  | Description |
|----------|-------------|
| Hold    | Deferred due to capacity or backpressure. |
| Meet    | Rejected because the request did not satisfy boundary or policy constraints. |
| Shift   | Continuation boundary changed; re-handshake or re-reservation required. |
| Spread  | Incomplete or abandoned execution (timeout, disconnect, partial upload). |
| Settle  | Terminal outcome recorded (served or cleanly rejected). |

Recommended mapping:

- Hold → `conn.outcome = 3` (deferred)
- Meet → `conn.outcome = 2` with validation/policy reason codes
- Shift → reason codes indicating slot/session/sequence reset
- Spread → `conn.outcome = 4` (timeout) or `5` (dropped)
- Settle → `conn.outcome = 1` (served) or `2` (clean abort with full accounting)

Operators SHOULD use these buckets to compute:

- Served rate (Settle where outcome = served)
- Not-served rate (Hold + Meet + Spread)
- Unsettled rate (Spread cases lacking clean confirmation)

This classification is advisory and intended for observability and governance layers.

These buckets are non-normative and are not transmitted on the wire. They are derived classifications for dashboards only.

---

## 8. Error Handling

Errors are returned using:

- `clarify + need` for recoverable issues
- `confirm + abort` for terminal failures

Error frames may include:

- `error_code`
- `error_message`
- `need_code`

---

## 9. Capacity & Admission Control

Heptaconn servers MAY implement capacity admission prior to compute execution.

A server MUST NOT begin expensive compute (e.g., inference) unless capacity has been explicitly granted.

Capacity grant occurs at the pre-booked admission decision.
Compute time (backend execution) is a separate phase and MUST NOT be conflated with admission latency.
Admission and execution timing are intentionally separated in v1 to allow millisecond-level fairness accounting.

Capacity decisions are expressed using existing verbs:

- `clarify + ok` → capacity granted
- `clarify + yield` → temporary backpressure
- `confirm + abort` → terminal rejection

Servers SHOULD:

- Enforce explicit budget ceilings (max tokens, max wall time, max bytes).
- Avoid silent queueing without acknowledgment.
- Prefer `yield` with retry hints over accepting work that cannot be scheduled.

Capacity enforcement is orthogonal to transport authentication and does not require external orchestration systems.

---

## 10. Non-Goals

- No content inspection
- No application-layer retries
- No HTTP semantics
- No schema enforcement of payload

Heptaconn defines execution boundaries and transport discipline only.

---

End of Specification (v1 transport / v0 protocol)
