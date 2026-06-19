---
layout: default
title: Query API (PQS)
---

# Query API

A read/query layer over your validator's ledger state. Query active contracts, balances, and history in SQL or over HTTP — incrementally indexed and kept current — without hammering the Ledger API.

## Why it exists

The Canton Ledger API is built for transacting, not for analytics. Reading a large active-contract set, running reporting queries, or paginating history through it is slow and hits element-count caps. The Daml **Participant Query Store (PQS)** solves the indexing half: it projects your participant's contracts into a queryable store, decoded into typed columns, updated as the ledger advances.

Saxon runs PQS for you and puts a clean, authenticated **HTTP query API** in front of it — so your application or ops tooling can ask questions like "all active holdings for this party," "balance history," or "what changed since this offset" with simple calls, instead of maintaining your own indexer.

## What you get

- **SQL-queryable ledger state** — active contracts and history projected into typed, decoded views, indexed for the queries you actually run.
- **An HTTP API in front of it** — API-key-authenticated, with read endpoints for lookup, search, history, and offset-pinned snapshots (consistent reads at a point in time).
- **Incremental + current** — the store seeds from the active-contract set and then tails the ledger, so reads reflect live state.
- **Read-only and isolated** — runs beside your participant on a dedicated store; it never writes to the ledger.

## Where it fits

This is the **read/query** side of a validator's tooling, complementary to the [automation daemon](index)'s actuation side. The daemon acts on contracts; the Query API lets you and your apps see and report on them at scale.

> **Integration:** the API surface, authentication, and the view/endpoint reference are provided per deployment. Contact Saxon for access.
