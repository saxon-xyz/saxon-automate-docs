---
layout: default
title: Imported Actions
---

# Imported Actions

YAML jobs cover the standard automation patterns — fire a choice when a deadline passes, when contracts match, when a contract exists. But some workloads need custom logic that doesn't fit a single choice exercise: a billing engine that scans the ledger, computes amounts, and submits a settlement; a treasury job that checks balances against thresholds across multiple parties; a multi-step orchestration that depends on its own off-chain state.

For those, Saxon Automate ships an **imported action** pattern. You register a function from any npm package in the catalog, and the daemon dispatches to it on the trigger you choose. The function receives the daemon's shared OAuth2 token provider, ledger client, and process-lifetime cache — same machinery the built-in jobs use, but with arbitrary code in between.

## When To Use It

Stick with YAML jobs when:

- One contract → one choice, no off-chain state
- Trigger fires, you exercise a choice on the matching contract, done

Reach for an imported action when:

- The choice arguments need to be computed from a scan of the ledger
- The job needs to consult an external API (price feed, scan, etc.) before submitting
- You need to persist progress between dispatches (so the next run doesn't re-do work)
- The same dispatch should result in multiple correlated submissions (e.g. settle several contracts atomically)

## Architecture

```
catalog.yaml
  apps:
    my-billing-app:
      jobs:
        bill-subscriptions:
          trigger: deadline
          watch:
            module: Acme.Billing.V1
            entity: Subscription
          when:
            field: nextDueDate
            condition: past
          action:
            type: imported
            package: '@acme/billing-runner'
            function: runBilling
            args:
              subscriptionId: "$contract.subscriptionId"
              contractId: "$contract.contractId"
```

When the trigger fires:

1. Daemon resolves the matching contract from its ContractStore.
2. Substitutes `$contract.*` placeholders into the `args` map.
3. Dynamically imports `@acme/billing-runner`, calls `runBilling(args, deps)`.
4. The imported function returns a `{ status, runId, ... }` record that's logged + metricked the same way built-in jobs are.

The package must be resolvable from the daemon's runtime — pin it as a `dependencies` entry in `apps/saxon-automate/package.json` (or its monorepo equivalent) before building the daemon image.

## What The Function Receives

```typescript
interface ImportedActionDeps {
  // OAuth2 token provider, shared across all dispatches in the daemon
  // lifetime. Reuses cached tokens; mints fresh ones on refresh-ahead-of-expiry
  // or 401 retry.
  tokenProvider?: TokenProvider;

  // Pre-built ledger client wired with the token provider above.
  ledger?: LedgerHttpClient;

  // Process-lifetime cache. Daemon hands the same Map to every dispatch.
  // Use this for heavyweight clients (registry, submitter) that should be
  // constructed once and reused — namespace your keys with your package
  // name to avoid collision with other imported actions.
  cache?: Map<string, unknown>;
}
```

The args map is whatever the catalog YAML resolved to — string-typed contract field values, plus any literals you specified. Your function is responsible for parsing/validating.

## Settlement Via The CIP-56 Token Standard

Imported actions that need to move CC on-chain can use Splice's CIP-56 transfer-factory pattern. The flow:

1. Build a `TransferFactory_Transfer` choice argument (sender, receivers, amounts, instrument).
2. POST it to the scan registry at `/registry/transfer-instruction/v1/transfer-factory` — scan returns `{ factoryId, transferKind, choiceContext: { choiceContextData, disclosedContracts } }`.
3. Construct your own Daml choice that takes a `TokenTransferContext` argument carrying that `choiceContextData` opaquely, plus the disclosed contracts on the submit request.
4. Submit via the JSON Ledger API's `/v2/commands/submit-and-wait`.

Saxon Automate's core ships a `RegistryClient` (`@saxon-xyz/saxon-automate-core` → `makeRegistryClient`) that wraps the registry call. Your imported action constructs the choice args; the rest is plumbing.

Notes from working examples:

- The choice context's `values: TextMap AnyValue` envelope must be unwrapped before passing through to Daml — splice's `ChoiceContext` Daml type is a record around the map; downstream code typically wants the bare TextMap.
- `TransferFactory_Transfer` asserts `length inputHoldingCids >= 1` even for self-transfers. Query the sender's active `Splice.Amulet:Amulet` contracts via `/v2/state/active-contracts` and pick enough to cover the amount.
- `nextDueDate` on any cadence-based contract should be advanced past `Date.now()` at submit time (loop `addFrequency` until in the future). Catches up correctly after daemon downtime.

## Operational Notes

- **Daemon owns the cache.** Tokens, registry clients, and any other heavyweight construction live for the daemon's lifetime. Per-dispatch construction means a fresh OAuth2 token every fire — measurable cost on a 10s polling cadence.
- **Failure modes.** An imported action that throws is logged as `failed_submission` (or `failed_validation` if it threw during schema check). The daemon does NOT retry automatically — the next deadline tick will dispatch again. Idempotency is your function's responsibility.
- **State persistence.** If you need per-agreement state between dispatches, write it to disk under a path you control (the daemon mounts `/data` by convention). Don't rely on the cache for durable state — it's process-lifetime only.
- **Observability.** Every dispatch generates one `runs_total{status}` metric increment and one `run_duration_seconds` histogram observation, tagged with the run status string your function returns. Use distinctive status strings (`skipped_x`, `not_due_y`) to make Grafana queries useful.

## Catalog Trigger Recap

Imported actions work with any of the catalog's trigger types:

| Trigger | When it fires | Typical imported-action use |
|---|---|---|
| `deadline` | A field crosses `now` (e.g. `nextDueDate < now`) | Recurring jobs: billing periods, traffic top-ups, scheduled rollovers |
| `interval` | Every N seconds, regardless of contract state | Verification / reconciliation that should run on its own clock |
| `exists` | At least one contract matches the filter | Lifecycle: process the queue, settle once, the trigger goes quiet |
| `match` | Two contract streams meet a join condition | Settlement pairs (proposal + allocation, mint + transfer) |

See [Configuration Reference](config) for the full trigger schema.
