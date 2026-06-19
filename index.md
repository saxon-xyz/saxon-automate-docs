---
layout: default
title: Saxon Automate
---

# Saxon Automate

Zero-config automation daemon for Canton Network validators. Scans your participant's installed DARs, auto-discovers what to automate, and exercises choices based on triggers — deadlines, settlement matching, contract existence, and periodic intervals. No YAML required to get started.

## Why You Need It

Canton's privacy model means no third party can observe your contracts or act on your behalf. Unlike public blockchains where services like Chainlink Keepers can automate transactions for anyone, Canton requires automation to run on your own node with your own credentials. Every validator that wants automated settlement, expiry management, or recurring billing needs its own automation daemon.

Saxon Automate solves this: one daemon that auto-discovers installed apps and handles any Daml contract lifecycle automation. Zero config to get started, YAML overrides when you need control. No custom code, no Daml expertise required.

Saxon Automate follows the same automation patterns used internally by [Splice](https://github.com/hyperledger-labs/splice) — polling jitter to prevent thundering herd across validators, silent retries for transient failures, and graceful reconnection on stream interruptions.

**Financial benefits:**
- **Earn Canton Coin rewards** — for a registered Featured App, the transactions Saxon Automate submits earn Canton Coin rewards from the network reward pool. Saxon Automate keeps that rewarded volume flowing automatically.
- **Maximize transaction volume** — automated choices fire immediately when conditions are met, generating more rewarded transactions than manual operation
- **Reduce operational cost** — no manual monitoring or intervention needed for routine contract lifecycle operations
- **Featured App program** — the reward pool is front-loaded toward active applications in the network's early years. Featured App status isn't automatic, though: it requires a `FeaturedAppRight` and, under CIP-0116, locking a Canton Coin stake (App-provider tier) to activate and maintain reward eligibility. Saxon helps you set this up.

## What It Does

Saxon Automate runs alongside a Canton participant node and automates contract lifecycle operations that would otherwise require manual intervention or custom code. Saxon Automate auto-discovers installed apps from a built-in catalog of known Daml templates. Operators can override or extend with YAML config, or plug in custom logic via [imported actions](imported-actions).

**Example automations:**
- Cancel expired trade proposals when their deadline passes
- Settle multi-leg trades when all allocations are present
- Execute accepted mints and transfers automatically
- Process recurring subscription payments
- Bill fees on commercial agreements at regular intervals
- Clean up audit records (settled DVPs, failed transfers)
- Auto-purchase synchronizer traffic when the balance runs low — see [Traffic Top-Up](traffic-topup) for the CIP-0104 auto-top-up flow
- Submit on-chain payments via the [Splice CIP-56 transfer-factory pattern](imported-actions#settlement-via-the-cip-56-token-standard) — custom Daml settlement choices that exercise the registry-mediated transfer flow

For a registered Featured App, the transactions Saxon Automate submits earn Canton Coin rewards under the Featured App program.

## Beyond the daemon

Saxon Automate is part of a small family of Saxon-operated validator services. Alongside the automation daemon described here:

- **[External-Party Onboarding](external-party-onboarding)** — stand up self-custody (external) parties for your end users at signup, via a simple API. The user's key stays in your own custody.
- **[Query API (PQS)](query-api)** — query your validator's ledger state (active contracts, balances, history) in SQL or over HTTP, incrementally indexed.

## Example Output

```
INFO  [automate] starting — host=participant:5001 party=mynode-validator-1::1220abcd...
INFO  [automate] loaded 0 manual job(s)
INFO  [discover] scanning 87 package(s) against 4 catalog app(s)
INFO  [discover] found "utility-dars" (detected module: Utility.Settlement.App.V1.Model.Dvp)
INFO  [discover] discovered 6 job(s) from 1 app(s)
INFO  [automate] auto-discovered 1 app(s): utility-dars
INFO  [automate] total 6 job(s): settle-dvp, execute-accepted-mints, ...
INFO  [streamer] snapshotting active contracts...
INFO  [streamer] streaming live from offset 000000000000001a47
...
INFO  [settle-dvp] deadline passed for 00234042dbb64c32..., exercising Dvp_Settle
INFO  [settle-dvp] Dvp_Settle submitted for 00234042dbb64c32...
```

## Quick Start

Just provide your credentials. Saxon Automate auto-discovers everything else:

```bash
docker run -d \
  --name saxon-automate \
  --restart unless-stopped \
  --network host \
  -v saxon-automate-data:/data \
  -e AUTH0_TOKEN_URL="https://mynode.uk.auth0.com/oauth/token" \
  -e AUTH0_CLIENT_ID="your-client-id" \
  -e AUTH0_CLIENT_SECRET="your-client-secret" \
  -e LEDGER_API_AUDIENCE="https://ledger-api.canton.mynode.example.com" \
  -e VENUE_PARTY="mynode-validator-1::1220abcd..." \
  -e LEDGER_API_USER="your-client-id@clients" \
  -e LEDGER_API_HOST="localhost:5001" \
  ghcr.io/saxon-xyz/saxon-automate:latest
```

No config file needed. Saxon Automate scans your participant for installed apps and generates jobs automatically. Add a `saxon-automate.yaml` mount only if you need to override or extend.

See the full [Installation Guide](install) for details.

## Pages

- [Installation Guide](install) — Step-by-step setup for Docker and Kubernetes
- [Configuration Reference](config) — Trigger types, field paths, and argument expressions
- [Imported Actions](imported-actions) — Plug in custom JS/TS functions for workloads that don't fit a single choice exercise (multi-step orchestration, ledger-derived choice args, CIP-56 settlement)
- [External-Party Onboarding](external-party-onboarding) — Onboard self-custody (external) end-user parties at signup via a simple API
- [Query API (PQS)](query-api) — Query your validator's ledger state in SQL/HTTP, incrementally indexed
- [Traffic Top-Up](traffic-topup) — Auto-purchase CIP-0104 synchronizer traffic when the operator's balance runs low
- [Example Configs](examples) — Ready-made configs for DA Utility DARs, Cantara, BitSafe CBTC, and traffic top-up
- [Canton Coin Rewards](rewards) — How Saxon Automate earns rewards under the traffic-based CIP-0104 model
- [Operator Tips](operator-tips) — Canton/Splice platform quirks worth knowing
- [Roadmap](roadmap) — Shipped, active, planned

## Support

Contact Saxon Nodes for support.
