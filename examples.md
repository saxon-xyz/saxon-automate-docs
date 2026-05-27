---
layout: default
title: Example Configs
---

# Example Configs

Download a config to get started, or combine jobs from multiple examples.

## Canton Swap

Automates OTC trade settlement and expired proposal cleanup for the Canton Swap application.

```yaml
defaults:
  pollIntervalMs: 10000
  deduplicationSeconds: 300

jobs:
  cancel-expired-proposals:
    trigger: deadline
    watch:
      module: Obsidian.CantonSwap.V1
      entity: TradeProposal
    when:
      field: validUntil
      condition: past
    exercise:
      choice: TradeProposal_Cancel
      args:
        cancelor: "$party"

  settle-otc-trades:
    trigger: match
    watch:
      module: Obsidian.CantonSwap
      entity: OTCTrade
    match:
      module: Splice.AmuletAllocation
      entity: AmuletAllocation
      ref: allocation.settlement.settlementRef.cid??
      leg: allocation.transferLegId
      legs: transferLegs
    deadline: settleBefore
    exercise:
      choice: OTCTrade_Settle
      args:
        allocationsWithContext:
          $eachLeg:
            _1: "$match.contractId"
            _2:
              context:
                values: {}
              meta:
                values: {}
```

## Digital Asset Utility DARs

Automates operations for Digital Asset's reference utility packages: DVP settlement, registry mint/transfer execution, collateral transfers, commercial fee billing, and audit record cleanup.

```yaml
defaults:
  pollIntervalMs: 10000
  deduplicationSeconds: 300

jobs:
  settle-dvp:
    trigger: match
    watch:
      module: Utility.Settlement.App.V1.Model.Dvp
      entity: Dvp
    match:
      module: Splice.AmuletAllocation
      entity: AmuletAllocation
      ref: allocation.settlement.settlementRef.cid??
      leg: allocation.transferLegId
      legs: transferLegs
    deadline: settleBefore
    exercise:
      choice: Dvp_Settle
      args:
        allocationsWithContext:
          $eachLeg:
            _1: "$match.contractId"
            _2:
              context:
                values: {}
              meta:
                values: {}

  execute-accepted-mints:
    trigger: exists
    watch:
      module: Utility.Registry.V0.Holding.Mint
      entity: AcceptedMint
    exercise:
      choice: AcceptedMint_Execute
      args: {}

  execute-accepted-transfers:
    trigger: exists
    watch:
      module: Utility.Registry.V0.Holding.Transfer
      entity: AcceptedTransfer
    exercise:
      choice: AcceptedTransfer_Execute
      args: {}

  execute-collateral-transfers:
    trigger: exists
    watch:
      module: Utility.Collateral.App.Model.Collateral
      entity: InstructedCollateral
    exercise:
      choice: InstructedCollateral_ExecuteTransfer
      args: {}

  bill-base-fees:
    trigger: interval
    watch:
      module: Utility.Commercials.V0.Model.CommercialAgreement
      entity: CommercialAgreement
    every: 86400000
    exercise:
      choice: CommercialAgreement_BillBaseFee
      args:
        currentTime: "$now"

  cleanup-settled-dvps:
    trigger: exists
    watch:
      module: Utility.Settlement.App.V1.Model.Dvp
      entity: SettledDvp
    exercise:
      choice: SettledDvp_Delete
      args: {}
```

## Cantara Subscriptions

Automates subscription lifecycle for the Cantara billing platform. Based on user feedback from the Cantara Slack community:
- Subscriptions with payments older than 30 days have expired locks needing manual unlock
- Cancellations older than 30 days get stuck because locked amulets expired

Saxon Automate automates all of these operations.

```yaml
lookups:
  - module: Splice.Round
    entity: OpenMiningRound
  - module: Splice.Amulet
    entity: FeaturedAppRight

defaults:
  pollIntervalMs: 10000
  deduplicationSeconds: 300

jobs:
  # Auto-accept user service requests
  accept-user-service-requests:
    trigger: exists
    watch:
      module: Cantara
      entity: UserServiceRequest
    exercise:
      choice: UserServiceRequest_Accept
      args: {}

  # Process recurring payments on time (prevents 30-day lock expiry)
  bill-subscriptions:
    trigger: deadline
    watch:
      module: Cantara
      entity: Subscription
    when:
      field: nextPaymentTime
      condition: past
    exercise:
      choice: Subscription_MakePayment
      args:
        issuerTransferContext:
          openMiningRound: "$lookup.Splice.Round:OpenMiningRound"
          featuredAppRight: "$lookup.Splice.Amulet:FeaturedAppRight"
        operatorTransferContext:
          openMiningRound: "$lookup.Splice.Round:OpenMiningRound"
          featuredAppRight: "$lookup.Splice.Amulet:FeaturedAppRight"
        issuerRewardShare: null
        subscriberRewardShare: null
        operatorRewardShare: null

  # Unlock expired locked amulets automatically
  # (no more manual clicking on Reporting > Expired Locks page)
  unlock-expired-locks:
    trigger: exists
    watch:
      module: Splice.Amulet
      entity: LockedAmulet
    exercise:
      choice: LockedAmulet_OwnerExpireLock
      args: {}
```

## BitSafe CBTC

Automates onboarding and credential lifecycle for [BitSafe](https://docs.bitsafe.finance/)'s wrapped Bitcoin token (CBTC) on Canton Network. Based on the BitSafe onboarding checklist:
- Phase 3: Auto-accept credential user service requests
- Phase 4: Auto-accept credential offers from BitSafe
- Execute credential transfers without manual intervention

```yaml
defaults:
  pollIntervalMs: 10000
  deduplicationSeconds: 300

jobs:
  # Auto-accept credential offers from BitSafe (Phase 4)
  accept-credential-offers:
    trigger: exists
    watch:
      module: Utility.Credential.App.V0.Model.Offer
      entity: CredentialOffer
    exercise:
      choice: CredentialOffer_AcceptFree
      args: {}

  # Auto-accept credential user service requests (Phase 3)
  accept-credential-service:
    trigger: exists
    watch:
      module: Utility.Credential.App.V0.Service.User
      entity: UserServiceRequest
    exercise:
      choice: UserServiceRequest_Accept
      args: {}

  # Execute accepted credential transfers
  execute-credential-transfers:
    trigger: exists
    watch:
      module: Utility.Registry.V0.Holding.Transfer
      entity: AcceptedTransfer
    exercise:
      choice: AcceptedTransfer_Execute
      args: {}
```

## Traffic Top-Up (CIP-0104)

Auto-purchases synchronizer traffic when the operator's balance falls below a threshold. The catalog ships this app paused-by-default — this config unpauses it and tunes for a hot validator. Live submits also require `TRAFFIC_TOPUP_ALLOW_LIVE=true` in the daemon's env; without it, the action runs in **shadow mode** (logs decisions only). See [Traffic Top-Up](traffic-topup) for the full architecture + tuning guide.

```yaml
jobs:
  topup-validator-traffic:
    trigger: interval
    paused: false
    watch:
      module: Splice.AmuletRules
      entity: AmuletRules
    every: 60000   # poll every 1 min — tighter than the catalog default for hot validators
    action:
      type: imported
      package: "@saxon-xyz/saxon-automate-daemon"
      function: runTrafficTopup
      args:
        threshold: 50000000      # fire when remaining < 50 MB
        topupAmount: 200000000   # buy 200 MB per fire
        maxInFlight: 10000000    # skip if a pending buy of > 10 MB is already in flight
        minIntervalMs: 60000     # at least 1 min between fires per (validator, member, migration)
        expiresInSec: 600        # BuyTrafficRequest auto-expires if uncompleted in 10 min
```

## Custom Imported Action (CIP-56 settlement)

For automations that need to compute settlement amounts from ledger state or external pricing before submitting, see [Imported Actions](imported-actions). Catalog stanza shape:

```yaml
defaults:
  pollIntervalMs: 10000
  deduplicationSeconds: 300

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
        # Forward the live contract id so the daemon doesn't submit
        # against an archived predecessor after the first settle.
        contractId: "$contract.contractId"
```

The runner package builds the choice args, posts to the scan registry at `/registry/transfer-instruction/v1/transfer-factory` to get the disclosed contracts + opaque choice context, then submits a Daml choice that exercises the transfer factory via `TokenTransferContext`. The daemon supplies the shared OAuth2 token provider and ledger client — see [Imported Actions](imported-actions) for the `deps` interface.
