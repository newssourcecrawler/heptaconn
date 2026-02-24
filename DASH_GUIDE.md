
# Dashboard Guide (Public v1)

This guide describes how to derive operational dashboards from Heptaconn frames and receipts.

It is transport-agnostic and UI-agnostic.

⸻

1. Inputs

You need newline-delimited JSON (NDJSON) frames emitted by a Heptaconn implementation.

Each frame may include:
	•	verb
	•	status
	•	job
	•	seq
	•	receipts[] (each receipt has kind and value_u64)

Dashboards are derived from receipts.

⸻

2. Core Receipts (v1)

Heptaconn transport guarantees the following business-facing receipts:
	•	conn.outcome
	•	conn.outcome_reason
	•	cdr.queue.ms
	•	cdr.backend.ms
	•	cdr.bytes_in

These receipts do not depend on payload inspection.

⸻

3. Outcome Accounting

conn.outcome represents the terminal disposition of a request attempt.

Outcome Codes

Code	Meaning
1	served
2	rejected
3	deferred
4	timeout
5	dropped

A request attempt is considered settled when a conn.outcome receipt is observed.

Each request attempt SHOULD emit exactly one terminal outcome.

⸻

4. Reason Codes

conn.outcome_reason refines the cause for non-served outcomes.

Recommended codes:

Code	Reason
1	busy
2	invalid_envelope
3	invalid_manifest
4	invalid_chunk
5	quota_exceeded
6	ticket_missing
7	ticket_expired
8	backend_error
9	backend_timeout

Dashboards SHOULD group non-served outcomes by reason to distinguish:
	•	capacity pressure
	•	validation failures
	•	backend instability

⸻

5. Latency Separation

Heptaconn separates admission latency from execution latency.

Admission Latency

cdr.queue.ms

Definition: milliseconds from TCP accept to the first admission decision.

Indicates transport-side capacity pressure.

⸻

Backend Latency

cdr.backend.ms

Definition: milliseconds spent inside backend execution.

Indicates model or compute performance.

⸻

Interpretation
	•	Rising cdr.queue.ms with stable cdr.backend.ms → admission contention.
	•	Rising cdr.backend.ms with stable cdr.queue.ms → backend slowdown.
	•	Both rising → overall saturation.

⸻

6. Core Derived Metrics

Over a time window:

Let:
	•	served = count(conn.outcome = 1)
	•	rejected = count(conn.outcome = 2)
	•	deferred = count(conn.outcome = 3)
	•	timeout = count(conn.outcome = 4)
	•	dropped = count(conn.outcome = 5)

Not Served

not_served = rejected + deferred + timeout + dropped

Served Rate

served_rate = served / (served + not_served)

Capacity Pressure

pressure = deferred / (served + rejected + deferred)


⸻

7. Five-Bucket View (Derived)

For higher-level dashboards, outcomes MAY be grouped into:

Bucket	Meaning
Hold	Deferred due to capacity
Meet	Rejected due to validation/policy
Spread	Timeout or dropped
Settle	Served or cleanly rejected
Shift	Boundary reset (if applicable)

These buckets are derived classifications. They are not transmitted on the wire.

⸻

8. Scope

This guide defines business-facing transport metrics only.

It does not require:
	•	token-level accounting
	•	GPU metrics
	•	model internals
	•	payload inspection

Future extensions MAY add optional receipts without changing served/not-served accounting.

⸻

End of Dashboard Guide (Public v1)


If you want, next we can tighten the README intro by 2 lines to match this tone exactly.
