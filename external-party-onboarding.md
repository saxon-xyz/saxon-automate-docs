---
layout: default
title: External-Party Onboarding
---

# External-Party Onboarding

Onboard end users as **self-custody Canton parties** — at signup, at scale, with a simple API. The user's signing key stays in your own key custodian; it never touches the participant and never touches Saxon.

## Why it exists

Canton supports *external parties*: parties whose signing key is held outside the participant, so the end user (or your app's custodian) is the only one who can authorize that party's actions. That's the right model for consumer apps where users should self-custody. But standing up an external party means generating its topology, getting it signed by the key holder, and submitting it — per user, reliably, often thousands of times.

This service productizes that. Your signup backend calls a small API; Saxon handles the participant-side topology work. The result is a live, self-custody party your user can transact as.

## How it works

A **two-call handshake**, because the signing step sits in the middle and only your key custodian can perform it:

1. **Generate** — your backend sends the user's public key and a name hint; the service returns the party id and the topology to be signed.
2. **Sign** — your key custodian (an MPC/HSM such as DFNS, or your own signer) signs, using the user's key. The private key never leaves your custodian.
3. **Submit** — your backend returns the signatures; the service submits them and the party goes live.

The party id is deterministic and returned by the first call, so you can assign it to the user **immediately** at signup; the party becomes transactable a few seconds later once topology propagates. The flow is **idempotent and async-friendly** — run it in the background and retry freely; a transient hiccup never has to block or fail a signup.

## Custody

The defining property: **no one but your custodian can act as the party.** Saxon drives the participant-side topology but cannot sign for the user, and the user's private key is never sent to Saxon or stored in the participant. Self-custody is preserved end to end.

## Where it fits

This is the onboarding (write) counterpart to the rest of Saxon Automate: the [automation daemon](index) actuates contract lifecycle on your node, and this service stands up the parties that participate. It runs as a Saxon-operated, access-controlled service alongside your validator.

> **Integration:** endpoints, authentication, and the request/response reference are provided per deployment. Contact Saxon for the integration guide.
