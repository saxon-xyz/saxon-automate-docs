---
layout: default
title: Traffic Top-Up
---

# Traffic Top-Up (CIP-0104)

Post-CIP-0104, every transaction consumes synchronizer traffic. Validators that exhaust their traffic balance can't submit any commands until they buy more. Manual top-up is operationally painful on busy validators; the in-process loop shipped by Splice's validator-app has scale problems under load (concurrent state-row contention, log floods, can't react to load curves).

Saxon Automate ships an [imported action](imported-actions) that monitors the operator's traffic balance and auto-purchases more before it runs out — small frequent purchases tuned to your validator's burn curve.

## How It Works

The buy is split across two actors:

1. **Saxon Automate** exercises `Splice.Wallet.Install:WalletAppInstall_CreateBuyTrafficRequest` via the JSON Ledger API. The choice creates a `BuyTrafficRequest` contract — Saxon Automate is the **scheduler**, not the executor.
2. **The validator-app's internal wallet automation** picks up the `BuyTrafficRequest`, runs coin selection through its `TreasuryService`, exercises `BuyTrafficRequest_Complete` → `AmuletRules_BuyMemberTraffic`. The sequencer applies the new allowance.

This means the validator-app must be running for completion to happen. The wallet's existing automation handles coin selection, disclosed-contracts, and the round-rollover edge cases; Saxon Automate just decides when.

## Why JSON Ledger API, not the wallet HTTP endpoint

The validator-app exposes `POST /api/validator/v0/wallet/buy-traffic-requests` that does the same thing — internally just exercising `WalletAppInstall_CreateBuyTrafficRequest` after a participant-topology lookup. Going JSON Ledger API direct keeps Saxon Automate's submitter pattern unified: one client, one auth model, same path billing uses. The cost is replacing the endpoint's tracking-id dedup with catalog-level interval gating + a deterministic `commandId`, and configuring the receiving participant ID as static env (`MEMBER_ID`) — fine because it rarely changes.

## Decision Logic

Each tick fetches the operator's traffic-status from scan and computes:

```
remaining = total_limit - total_consumed
inFlight  = total_purchased - total_limit   (pending buy not yet applied)
```

- If `inFlight > maxInFlight` → skip (a buy is already in progress).
- If `remaining >= threshold` → skip (no action needed).
- Otherwise → fire (in live mode) or shadow-log (otherwise).

A per-`(validator, member, migration)` dedup window (`minIntervalMs`) prevents multiple ticks from firing the same buy in rapid succession.

## Safety: Two Opt-In Gates

The job is **paused by default in the built-in catalog** so first-deploy doesn't fire surprise purchases. Two explicit gates must both be flipped to actually start firing in production:

1. **Unpause the job** in `saxon-automate.yaml`:
   ```yaml
   jobs:
     topup-validator-traffic:
       paused: false
       trigger: interval
       watch:
         module: Splice.AmuletRules
         entity: AmuletRules
       every: 300000   # 5 min
       action:
         type: imported
         package: "@saxon-xyz/saxon-automate-daemon"
         function: runTrafficTopup
         args:
           threshold: 5000000      # fire when remaining < 5 MB
           topupAmount: 20000000   # buy 20 MB per fire
           maxInFlight: 1000000    # skip if pending buy > 1 MB
           minIntervalMs: 60000    # at least 1 min between fires
           expiresInSec: 600       # request auto-expires if uncompleted
   ```

2. **Set `TRAFFIC_TOPUP_ALLOW_LIVE=true`** in the daemon's env. Without it, the job ticks in **shadow mode** — logs the decision it would make but never submits. Useful for verifying threshold/topup tuning is sane before committing to real spends.

Recommended workflow: unpause first, watch the shadow-mode logs for a few hours/days against your validator's real burn curve, then flip live.

## Required Env

In addition to the standard Saxon Automate credentials:

| Variable | Description | Example |
|---|---|---|
| `LEDGER_API_URL` | JSON Ledger API base | `http://participant:7575` |
| `SCAN_URL` | Scan API base for traffic-status | `https://scan.sv-1.example.com` |
| `MEMBER_ID` | Receiving participant's member ID | `PAR::mynode-validator-1::1220abcd...` |
| `SYNCHRONIZER_ID` | Daml synchronizer ID | `global-domain::1220abcd...` |
| `MIGRATION_ID` | Domain migration ID (often `0` or `1`) | `1` |
| `TRAFFIC_TOPUP_ALLOW_LIVE` | `true` to leave shadow mode | `true` |

The action reuses the daemon's existing `AUTH0_*` / `LEDGER_API_AUDIENCE` credentials.

### Finding the cluster-specific values

- **`MEMBER_ID`** — the participant's member ID on the sequencer. Format: `PAR::<participant-name>::<namespace-fingerprint>`. Find via the participant admin API (`GetParticipantId`) or — quicker — by inspecting your own validator's traffic-status response from scan: any member-id that returns data is yours.
- **`SYNCHRONIZER_ID`** — from your validator-app's config or via the JSON Ledger API at `/v2/state/connected-synchronizers`.
- **`MIGRATION_ID`** — from your validator-app's helm/config (`canton.validator-apps.validator_backend.domain-migration-id` or `ADDITIONAL_CONFIG_MIGRATION_ID`). Often `0` on first install, `1`+ after a synchronizer migration.

## Tuning the Gating Params

The catalog defaults (5 MB threshold, 20 MB topup) are conservative — they suit a quiet validator that occasionally needs to keep its tank from running dry. Hot validators burning megabytes per minute should run with much larger values:

| Validator profile | `threshold` | `topupAmount` | `every` |
|---|---|---|---|
| Quiet (catalog default) | 5 MB | 20 MB | 5 min |
| Moderate volume | 20 MB | 100 MB | 2 min |
| Hot validator | 50 MB | 200 MB | 1 min |

The right pattern is small-but-frequent rather than rare-and-huge: the network's recommended topup config keeps purchase bounds tight to avoid wasting CC on unused allocations.

## Coexistence With the Built-In Topup Loop

Splice's validator-app has its own internal traffic-top-up loop driven by `ADDITIONAL_CONFIG_TOPUPS` (helm `topup.targetThroughput` + `minTopupInterval`). This in-process loop is the "official" mechanism shipped by Splice today, but has scale problems under load: bursty block topups cause concurrent Postgres write contention against the same state rows, generating serialization-retry log floods. The built-in's `target-throughput` config can't react to load curves either — it forces stalls during synchronization rounds.

Saxon Automate's traffic-topup is the **external-daemon pattern** that the in-process loop can't be: dynamic threshold, small frequent purchases sized to recent burn, sidestepping the contention. The two coexist fine while operators evaluate Saxon Automate's topup. Long-term, the built-in can be disabled (`ADDITIONAL_CONFIG_TOPUPS` unset) once Saxon Automate has run reliably for a while.

## Gotchas

These come up rarely but cost time when they do:

- **Daml `Int` must be JSON-string-encoded in choice args.** `migrationId` and `trafficAmount` sent as raw JSON numbers fail with `HTTP 500 "Expected ujson.Str (data: …)"`. Saxon Automate's submitter handles this internally; mentioned here for anyone reading the wire format or writing a similar integration.
- **Scan's `target.total_purchased` lags `actual.total_limit` by ~60-90s after a buy completes.** The `inFlight = purchased - limit` formula can be briefly *negative* right after a successful topup completes (sequencer applies the new allowance before scan's purchased counter refreshes). Saxon Automate's decision gate (`inFlight > maxInFlight`) handles this correctly; code that asserts `inFlight >= 0` will spuriously fire.

## Reward Implications

Every `WalletAppInstall_CreateBuyTrafficRequest` submission is a Daml transaction that earns Canton Coin rewards under the Featured App program — same as any other transaction Saxon Automate submits. Auto-top-up is therefore not "operational cost" but a positive-yield activity: you spend a little CC on traffic, earn rewards on the submit, and avoid the bigger cost of validator stalls when traffic runs dry.

See [Rewards](rewards) for the full accounting model.
