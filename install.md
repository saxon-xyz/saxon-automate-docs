---
layout: default
title: Installation Guide
---

# Installation Guide

## Prerequisites

- A running Canton validator node with participant API on port 5001
- Auth0 or Keycloak credentials for the Ledger API
- Docker installed on your validator node

## Step 1: Get Your Credentials

You need these values from your validator setup:

| Value | Where to find it | Example |
|-------|-----------------|---------|
| Token URL | Auth0: domain + `/oauth/token`. Keycloak: realm + `/protocol/openid-connect/token` | `https://mynode.uk.auth0.com/oauth/token` or `https://keycloak.example.com/realms/canton/protocol/openid-connect/token` |
| Client ID | Your M2M application / service account settings | `aB1cD2eF3gH4iJ5kL6mN7oP8qR9sT0u` |
| Client Secret | Your M2M application / service account settings | `xY9wV8uT7sR6qP5oN4mL3kJ2iH1gF0e...` |
| Ledger API audience | Your participant's API identifier | `https://ledger-api.canton.mynode.example.com` |
| Party ID | Your validator party ID | `mynode-validator-1::1220abcd1234...` |
| Ledger API user | Usually `<client-id>@clients` | `aB1cD2eF3gH4iJ5kL6mN7oP8qR9sT0u@clients` |

## Step 2: Choose Your Jobs

**Option A: Auto-discovery (recommended, zero-config)**

Saxon Automate automatically scans your participant's installed DARs and generates jobs for any recognized apps. No config file needed. Just skip to Step 3.

**Option B: Manual config**

Download an example config for specific apps:

```bash
# For Canton Swap automation
curl -o saxon-automate.yaml {{ site.examples_url }}/canton-swap.yaml

# For DA Utility DAR automation
curl -o saxon-automate.yaml {{ site.examples_url }}/utility-dars.yaml
```

Or write your own — see the [Configuration Reference](config).

**Option C: Both**

Auto-discovery runs by default. Any manual jobs you add to `saxon-automate.yaml` take precedence over auto-discovered ones with the same name. To suppress a specific auto-discovered job:

```yaml
exclude:
  - settle-otc-trades
  - bill-base-fees
```

To disable auto-discovery entirely: `autoDiscover: false`

## Step 3: Run

```bash
docker run -d \
  --name saxon-automate \
  --restart unless-stopped \
  --network host \
  -v $(pwd)/saxon-automate.yaml:/app/saxon-automate.yaml \
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

## Step 4: Verify

```bash
docker logs saxon-automate
```

You should see Saxon Automate start, auto-discover apps, resolve packages, and begin streaming:

```
INFO  [automate] starting — host=localhost:5001 party=mynode-validator-1::1220abcd...
INFO  [automate] loaded 0 manual job(s)
INFO  [discover] scanning 87 package(s) against 4 catalog app(s)
INFO  [discover] found "utility-dars" (detected module: Utility.Settlement.App.V1.Model.Dvp)
INFO  [discover] found "bitsafe-cbtc" (detected module: Utility.Credential.App.V0.Model.Offer)
INFO  [discover] discovered 9 job(s) from 2 app(s)
INFO  [automate] auto-discovered 2 app(s): utility-dars, bitsafe-cbtc
INFO  [automate] total 9 job(s): settle-dvp, execute-accepted-mints, ...
INFO  [packages] scanning 87 package(s) for module "Utility.Settlement.App.V1.Model.Dvp"
INFO  [packages] found "Utility.Settlement.App.V1.Model.Dvp" in package 3f8a2b...
INFO  [automate] FeaturedAppRight discovered: 00a1b2c3d4e5f6...  (pkg=7804375f...)
INFO  [streamer] snapshotting active contracts...
INFO  [streamer] snapshot complete at offset 000000000000001a47, store size: 12
INFO  [streamer] streaming live from offset 000000000000001a47
```

If no FeaturedAppRight is registered for your validator, Saxon Automate still runs normally:

```
INFO  [automate] no FeaturedAppRight found — transactions will not earn rewards
```

To see per-poll detail, set `LOG_LEVEL=DEBUG`:

```bash
docker stop saxon-automate
# Re-run the docker run command from Step 3 with -e LOG_LEVEL=DEBUG
```

```
DEBUG [cancel-expired-proposals] tick: 3 contract(s)
DEBUG [streamer] transaction at offset 000000000000001a48, 2 event(s)
```

When a contract matches a trigger, Saxon Automate exercises the choice:

```
INFO  [cancel-expired-proposals] deadline passed for 007bc94ffd..., exercising Proposal_Cancel
INFO  [cancel-expired-proposals] Proposal_Cancel submitted for 007bc94ffd...
INFO  [store] archived Your.App.Module:Proposal 007bc94ffd...
```

## Environment Variables

| Variable | Required | Description | Example |
|----------|----------|-------------|---------|
| `AUTH0_TOKEN_URL` | Yes | OAuth2 token endpoint | `https://mynode.uk.auth0.com/oauth/token` |
| `AUTH0_CLIENT_ID` | Yes | Service account client ID | `aB1cD2eF3gH4iJ5kL6mN7oP8qR9sT0u` |
| `AUTH0_CLIENT_SECRET` | Yes | Service account client secret | `xY9wV8uT7sR6qP5oN4mL3kJ2iH1gF0e...` |
| `LEDGER_API_AUDIENCE` | Yes | Ledger API resource identifier | `https://ledger-api.canton.mynode.example.com` |
| `VENUE_PARTY` | Yes | Operator party ID (Canton format) | `mynode-validator-1::1220abcd1234...` |
| `LEDGER_API_USER` | Yes | Ledger API user (JWT subject) | `aB1cD2eF3gH4iJ5kL6mN7oP8qR9sT0u@clients` |
| `LEDGER_API_HOST` | No | Participant address (default: `participant:5001`) | `participant:5001` |
| `AUTOMATE_CONFIG` | No | Config file path (default: `saxon-automate.yaml`) | `/app/saxon-automate.yaml` |
| `AUTOMATE_DB_PATH` | No | SQLite state DB path (default: `./automate-state.db`) | `/data/automate-state.db` |
| `LOG_LEVEL` | No | `DEBUG`, `INFO`, `WARN`, `ERROR` (default: `INFO`) | `INFO` |

## Updating

```bash
docker pull ghcr.io/saxon-xyz/saxon-automate:latest
docker restart saxon-automate
```

## Updating Your Config

```bash
# Edit saxon-automate.yaml, then restart
docker restart saxon-automate
```

## Health Check

Saxon Automate exposes an HTTP health endpoint on port 8080:

```bash
curl http://localhost:8080/health
```

```json
{"status":"healthy","streamActive":true,"contractCount":12,"offset":"000000000000001a47"}
```

| Field | Meaning |
|-------|---------|
| `streamActive` | `true` when Saxon Automate is connected to the participant and receiving updates |
| `contractCount` | Number of contracts Saxon Automate is tracking across all subscribed templates |
| `offset` | Current ledger offset (increases with each transaction) |

If using Kubernetes, the example manifest includes liveness and readiness probes pointed at this endpoint.

## After a Network Upgrade

When the Canton Network upgrades (e.g. Splice 0.5.17 to 0.5.18), package IDs change because new DAR versions are deployed. Saxon Automate handles this automatically — it resolves all package IDs at startup by scanning the participant's package store.

**What to do: restart Saxon Automate.**

```bash
# Docker
docker restart saxon-automate

# Kubernetes
kubectl -n canton rollout restart deployment saxon-automate
```

After restart, check the logs for successful package resolution:

```
INFO  [packages] scanning 94 package(s) for module "Your.App.Module"
INFO  [packages] found "Your.App.Module" in package 9c4d7e...
INFO  [automate] resolved Your.App.Module → 9c4d7e...
INFO  [streamer] snapshot complete at offset 000000000000002b13, store size: 8
INFO  [streamer] streaming live from offset 000000000000002b13
```

The new package ID (`9c4d7e...` instead of the previous `3f8a2b...`) confirms Saxon Automate picked up the upgraded packages. No config changes required.

**If a module is not found** after an upgrade, Saxon Automate retries 3 times over 90 seconds (the DAR may still be deploying). If still not found, Saxon Automate skips that job and continues running the rest:

```
WARN  [packages] module "Your.App.Module" not found, retrying in 30s...
ERROR [automate] module "Your.App.Module" not found after retries — jobs using it will be skipped
ERROR [automate] skipping job "my-job" — missing module(s): Your.App.Module
```

Other jobs keep running normally. Once the DAR is deployed, restart Saxon Automate to pick it up.

## Kubernetes — Helm (recommended)

The Helm chart is published as an OCI artifact on GHCR, alongside the Docker image.

```bash
helm install saxon-automate oci://ghcr.io/saxon-xyz/charts/saxon-automate \
  --version 0.1.0 \
  -n canton \
  --set venueParty="mynode-validator-1::1220abcd..." \
  --set auth.tokenUrl="https://mynode.uk.auth0.com/oauth/token"

# Verify
kubectl -n canton logs -l app=saxon-automate -f
```

The chart assumes the standard Splice secret `splice-app-validator-ledger-api-auth` exists in the namespace. Override with `--set auth.existingSecret=my-secret` if your auth secret has a different name.

For custom values (disabling auto-discovery, manual jobs, custom storage class, etc.), use a values file:

```bash
helm install saxon-automate ./helm/saxon-automate -n canton -f values.yaml
```

See `helm/saxon-automate/values.yaml` for all available options.

## Kubernetes — Raw manifest

If you prefer not to use Helm, download the example manifest:

```bash
curl -o saxon-automate-k8s.yaml {{ site.examples_url }}/k8s-deployment.yaml
```

Then edit the placeholder values and apply:

```bash
kubectl -n canton create configmap saxon-automate-config \
  --from-file=saxon-automate.yaml=saxon-automate.yaml
kubectl -n canton apply -f saxon-automate-k8s.yaml
```

## Optional: Enable Traffic Auto-Top-Up

Saxon Automate can auto-purchase CIP-0104 synchronizer traffic when your validator's balance runs low — useful on hot validators where manual top-up is operationally painful. The feature is paused-by-default and requires explicit env opt-in to leave shadow mode.

See [Traffic Top-Up](traffic-topup) for the architecture, decision logic, env vars, and recommended workflow (shadow-mode first, then flip live).

## Troubleshooting

**No jobs fire:** Set `-e LOG_LEVEL=DEBUG` to see poll ticks and contract counts.

**Authentication errors:** Verify your credentials work:
```bash
# Auth0
curl -s -X POST https://mynode.uk.auth0.com/oauth/token \
  -H "content-type: application/json" \
  -d '{"client_id":"...","client_secret":"...","audience":"...","grant_type":"client_credentials"}'

# Keycloak
curl -s -X POST https://keycloak.example.com/realms/canton/protocol/openid-connect/token \
  -d "client_id=...&client_secret=...&grant_type=client_credentials"
```

**Module not found after upgrade:** Saxon Automate retries 3 times (30s apart), then skips that job. Other jobs keep running. Once the DAR is deployed, restart Saxon Automate.

**Stream disconnects:** Normal during participant restarts — Saxon Automate automatically reconnects within 5 seconds:
```
ERROR [streamer] cycle failed (code=14): 14 UNAVAILABLE: Connection dropped
WARN  [streamer] reconnecting in 5000ms
INFO  [streamer] snapshotting active contracts...
INFO  [streamer] streaming live from offset 000000000000002b15
```

## Support

Contact Saxon Nodes for support.
