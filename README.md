# Heptaconn

Heptaconn is a framed TCP protocol for explicit execution boundaries and receipt-based accounting.

It separates:

- Admission latency (`cdr.queue.ms`)
- Backend execution latency (`cdr.backend.ms`)
- Terminal outcome accounting (`conn.outcome`)

Heptaconn enforces envelope discipline and transport-level fairness.
It does not inspect payload content.

## What this repository contains

- `SPEC.md` — Protocol specification (v0 protocol / v1 transport receipts)
- `DASH_GUIDE.md` — Operator-facing dashboard guidance

This repository defines the protocol and its operational model.
Reference implementations may vary.

---

Status: Draft (v0 protocol, v1 transport receipts)
License: MIT
