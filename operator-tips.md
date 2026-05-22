---
layout: default
title: Operator Tips
---

# Operator Tips

Things about Canton + Splice that the docs don't tell you but the daemon's behaviour relies on. Useful background for operators triaging unexpected log output.

## Template IDs come in two forms

The participant accepts **either** form on submission:

- `#<package-name>:<Module>:<Entity>` — the shorthand. Uses the package's logical name from `daml.yaml`.
- `<hex-package-id>:<Module>:<Entity>` — the explicit form. Uses the full hex package ID.

But the participant **emits the hex form** in ACS responses (`/v2/state/active-contracts`) and update streams (`/v2/updates/flats`), even when the request used the `#`-form. Exact-match equality of templateIds in response handlers silently drops every event. The daemon's classifiers all suffix-match on `:<Module>:<Entity>` instead. Custom imported actions that walk ACS responses should do the same.

## `templateFilter` is best-effort, not authoritative

The `/v2/state/active-contracts` request shape includes `filter.filtersByParty.<party>.cumulative[].templateFilter.templateId`. On some Canton versions this filter doesn't actually filter — the response returns all templates the party is involved with, mixed together. Saxon devnet (Canton 3.5+) regularly returns 100+ contracts of mixed templates for a single-template request.

Always post-filter client-side by `event.templateId.endsWith(:<Module>:<Entity>)`. Performance-wise, the wire format doesn't shrink with the filter on these versions, but the daemon's correctness can't depend on the server having dropped non-matching contracts.

## `ArchivedEvent` carries `witnessParties`, not `signatories` / `observers`

The JSON Ledger API's archived event variant has a shorter shape than the create event — it doesn't replay the signatory and observer lists from the prior create. The party-identity field present is `witnessParties` (the parties that observed this archive).

For "did the party own the archived contract?" filters (the typical self-spend filter in billing engines or settlement logic), check `witnessParties.includes(party)`. Reading `signatories` or `observers` on archive events returns undefined.

## `nextDueDate` advances past `now()` automatically

Cadence-based daml templates (typical billing, subscription, periodic-settlement contracts) usually assert `nextDueDate > currentTime` on each settle. When the daemon's been offline (or any cadence interval is missed), the next dispatch's `nextDueDate = window.to + cadence` could still be in the past, breaking the assertion.

The fix: at submit time, loop `nextDueDate = addFrequency(nextDueDate, cadence)` until `Date.parse(nextDueDate) > Date.now()`. Keeps the cadence schedule aligned (same period boundaries on the original wall clock) while skipping all missed periods in one settle.

## OAuth2 audience must match exactly

Mainnet validators commonly run separate Auth0 tenants from their devnet counterparts. The audience claim on the JWT must match what the participant's auth config expects — `https://ledger-api.canton.<host>` for devnet, `https://ledger-api.main.canton.<host>` (note the `.main.` infix) for mainnet on the deployments we've seen.

Symptom: every Ledger API call returns 401, even with apparently-valid credentials. Decode the JWT's `aud` claim and compare against `kubectl -n canton get secret splice-app-validator-ledger-api-auth -o jsonpath='{.data.audience}' | base64 -d`. They must match exactly.

## DAR upload ≠ DAR vetted

Uploading a DAR to a participant via the Admin API only makes the package bytes available; it doesn't authorize the package for use in transactions on the synchronizer. Mainnet uses the Splice SV super-validator vote to vet packages — until the vote passes, contracts using templates from the uploaded DAR fail with `MissingVettedPackages` (or equivalent).

On devnet, `synchronize_vetting=true` on the upload request usually succeeds immediately because the local SV vetoes are auto-approved. On mainnet, expect a wait window between upload and the first usable submission.

## `KNOWN_PACKAGE_VERSION` blocks vetting, not upload

The error code suggests "you can't upload this package twice", but the actual semantics are subtler: two packages with the same `(name, version)` and different package IDs CAN coexist in the participant's package store. The constraint is per-synchronizer-per-vetting-topology — only one variant of `(name, version)` can be vetted at a time.

If you need to ship a fixed-but-bytewise-identical package without bumping the version (e.g. you rebuilt with a newer SDK that produced different bytes), the cleanest path is to bump the package name, not the version. The old name+version stays vetted but inert; the new name gets its own vetting.

## Validator-admin credentials = highest-privilege

The k8s secret `splice-app-validator-ledger-api-auth` (in the `canton` namespace) holds the validator-app's M2M credentials. The token minted from these has `participant_admin: true` and `can_act_as` for the validator party plus all hosted wallet users.

Treat these as the highest-privilege credential in the deployment. They're appropriate for the daemon itself (one trusted process, narrow scope). They're not appropriate to share with client applications that should only act-as their own party.

## Scan API access from outside the cluster

The global scan API (`scan.sv-1.dev.global.canton.network.sync.global` for devnet, mainnet has a similar pattern) is reachable from inside any canton-namespace pod, but blocked by an AWS ELB IP allowlist from outside. For ad-hoc queries:

```bash
kubectl -n canton exec <any-pod> -- \
  curl -s -X POST -H 'content-type: application/json' \
  https://scan.sv-1.dev.global.canton.network.sync.global/api/scan/v0/<endpoint> \
  -d '{}'
```

This is fine for ad-hoc; for daemon use the in-cluster pod-network path is automatic.
