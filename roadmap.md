---
layout: default
title: Roadmap
---

# Roadmap

What's shipped, what's actively being built, what's planned.

## Shipped

- **Auto-discovery of installed apps.** Scan participant DARs, match against built-in catalog entries, generate jobs automatically. Zero YAML required for supported apps.
- **YAML jobs with four trigger types:** `deadline`, `interval`, `exists`, `match`. See [Configuration Reference](config).
- **Imported actions:** custom JavaScript/TypeScript functions registered in the catalog, dispatched on triggers with the daemon's shared OAuth2 token provider + ledger client + process-lifetime cache. See [Imported Actions](imported-actions).
- **On-chain settlement via Splice CIP-56 transfer factory.** `RegistryClient` in core wraps the scan registry call; imported actions can submit `SettleX` choices that exercise the CIP-56 transfer flow without daml needing to know the registry shape.
- **OAuth2 with refresh-ahead-of-expiry.** Token provider mints once, refreshes proactively before expiry, deduplicates concurrent callers, invalidates and retries on 401. 24h JWTs are minted once per daemon lifetime.
- **Canton 3.5+ JSON Ledger API support.** `filtersByParty.<party>.cumulative[].templateFilter` request shape; suffix-match on hex-vs-`#`-prefixed templateId in responses; activeAtOffset anchored to ledger-end for consistent ACS snapshots.
- **Adaptive backoff on participant element-count caps.** Catches HTTP 413 `JSON_API_MAXIMUM_LIST_ELEMENTS_NUMBER_REACHED` mid-scan, halves the window from the same cursor, retries; sticks at the post-shrink size for the rest of the scan so dense ledger regions don't repeatedly re-hit the cap.
- **Traffic auto-top-up (CIP-0104).** New imported action `runTrafficTopup` monitors the operator's traffic balance via scan and submits `WalletAppInstall_CreateBuyTrafficRequest` when the balance falls below a configured threshold. The validator-app's internal wallet automation completes the buy on-ledger. Paused-by-default in the catalog; `TRAFFIC_TOPUP_ALLOW_LIVE=true` in env to leave shadow mode. See [Traffic Top-Up](traffic-topup).

## Active

- **Per-validator deployment template** (Helm values templating). The daemon already supports running one instance per tenant via env config; the Helm chart is being parameterized so spinning up a new instance is a values-file diff.

## Planned

- **Prometheus alerting rule pack.** The daemon already exposes Prometheus metrics (`billing_runs_total`, `outstanding_amount_cc`, `record_skip_total`); the alerting rules to consume them are next.

## Open questions

These shape later work:

- **CIP-0104 rewards semantics.** The traffic-based rewards model is live, but several pricing and accounting details are still under discussion at the network level. Future Saxon Automate features may evolve as those settle.
- **DAR vetting workflow on mainnet.** Mainnet DAR uploads require the Splice SV super-validator vote to vet a new package — `MissingVettedPackages` errors until the vote passes. Faster than waiting, but no automation hook yet.

## Versioning

The daemon's published image is tagged with both `latest` and the commit SHA. There's no semantic-version release cadence yet — every push to main publishes. Pin to a SHA tag in production if you want change control.
