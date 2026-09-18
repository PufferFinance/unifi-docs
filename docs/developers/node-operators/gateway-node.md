---
title: Deploy a Gateway
slug: /developers/node-operators/gateway-node
---

# Deploy a Gateway

A **gateway** is the block builder. It simulates and orders transactions, emits
[fragments](../concepts/based-op-stack.md) as it builds so that users get preconfirmations, and seals
the block. UniFi's registry rotates sequencing duty between the registered gateways on a fixed
schedule, so once you are registered your gateway takes turns alongside the others.

This page is for operators running a gateway against a **main node operated by UniFi**.

```
   ┌──────────────────────────────────┐        ┌─────────────────────────┐
   │  UniFi main node                 │        │  Your gateway VM        │
   │                                  │        │                         │
   │  op-geth  op-node  op-batcher    │        │  based-gateway          │
   │  op-proposer                     │ ◄────► │  op-geth   op-node      │
   │  Portal   Registry   TxProxy     │ L2 p2p │                         │
   └──────────────────┬───────────────┘ + RPC  └─────────────────────────┘
                      │
                      ▼
                  user txs
```

Your gateway points at UniFi's portal, and its own `op-geth`/`op-node` sidecars peer with the main
node so you stay on the canonical chain. Those sidecars are a full [follower](./follower-node.md) —
a gateway host serves preconfirmed RPC as a side effect, and you should not deploy a separate
follower on the same machine.

## Trust model — read this before deploying

Running a gateway means accepting these properties. None of them is a bug; they are what the current
design is.

- **You hold a signing key.** Your gateway signs fragments under `GATEWAY_SEQUENCING_KEY`. The
  address derived from it is what every follower on the network uses to verify your fragments. Treat
  it as production-sensitive: anyone who obtains it can sign fragments in your name.
- **You share an Engine API JWT with UniFi.** It is generated on your VM but stored in the main
  node's registry so the portal can authenticate to you. UniFi can therefore read it. It only
  authenticates Engine API traffic — it cannot be used to move funds.
- **Registration is operator-mediated.** You cannot register yourself. You send UniFi your
  `[url, address, jwt]` row and they add it, which is also why onboarding is a conversation rather
  than a self-service form.
- **Removal is unilateral and fast.** UniFi can drop you from the registry and the transaction
  fan-out at any time, without notice. Those files hot-reload, so it takes effect within about 30
  seconds.
- **Preconfirmations are soft.** There is no collateral and no slashing behind them today. See
  [Gateway](../concepts/gateway.md) for where that is heading. A gateway that stalls or misbehaves
  costs preconfirmation liveness, not chain safety — the portal validates your block against a
  fallback client and produces its own if yours is late or invalid.

## Before you start

Work through the [common prerequisites](./index.md#common-prerequisites), and note two additional
requirements specific to a gateway:

- **A public, stable IP or DNS name.** Both UniFi's portal and the L2 p2p network reach your gateway
  from outside. A follower can hide; a gateway cannot.
- **Inbound firewall rules**, below.

Collect `PORTAL`, `TXPROXY`, `MAIN_OP_GETH_ENODE` and your L1 endpoints from
[what UniFi gives you](./index.md#what-unifi-gives-you-when-you-are-onboarded).

:::danger Two hard requirements that will stop you in the first minute
**`PORTAL` must be a literal IPv4 address, not a DNS name.** `make start-gateway` parses the host out
of it and aborts with `PORTAL does not resolve to an IPv4 host` on anything else, because it rewrites
the portal-reported gossip multiaddr to that host. Resolve a hostname yourself and pass the address.

**`MAIN_OP_GETH_ENODE` is mandatory** — the target refuses to run without it, and your value is used
*verbatim*, never rewritten. Make sure the address inside it is one your VM can actually reach.
:::

### Inbound rules on your gateway VM

Defaults from the template; substitute whatever you set in your own `.env`.

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| `GATEWAY_PORT` (9997) | TCP | UniFi's main node | Payload requests and transaction forwarding |
| `OP_NODE_GOSSIP_PORT` (9103) | TCP + UDP | Any | L2 consensus-client p2p |
| `OP_GETH_GOSSIP_PORT` (31303) | TCP + UDP | Any | L2 execution-client p2p |

Scope the gateway port to UniFi's source address if you have been given a stable one. The two gossip
ports must accept connections from any peer — the p2p mesh discovers peers dynamically, not only from
the main node.

Egress needs to reach UniFi's portal and its two p2p ports, your L1 execution and beacon endpoints,
and 443 for image builds. NTP outbound is required: drift causes Engine API authentication failures.

:::warning
If the VM is also behind a cloud security group, apply the same inbound rules there. A host firewall
runs *inside* the VM and cannot open a port the security group drops.
:::

## Deploy

### 1. Get the code

```bash
git clone https://github.com/PufferFinance/unifi-op.git
cd unifi-op
git submodule update --init --recursive
```

### 2. Generate a sequencing key

```bash
make -C devnet-deployment build-key_to_address
docker run --rm key_to_address
```

The first line of output is the private key and the second is the derived address. Save both — the
key goes in the next step, and UniFi needs the address to register you. **Do not reuse a key that is
registered against any other network.**

To derive the address from a key you already have: `docker run --rm key_to_address 0xYOUR_KEY`.

### 3. Start the gateway

```bash
cd devnet-deployment
cp config.mk.example config.mk    # required — see below

make start-gateway \
    PORTAL=http://<main-node-ipv4>:<portal-port> \
    TXPROXY=http://<main-node-ipv4>:<txproxy-port> \
    MAIN_OP_GETH_ENODE='enode://<pubkey>@<main-node-ip>:<port>' \
    GATEWAY_FOLLOWER_IP=<your-public-ip> \
    GATEWAY_SEQUENCING_KEY=0x<private-key-from-step-2> \
    L1_RPC_URL=<l1-rpc> \
    L1_BEACON_RPC_URL=<l1-beacon>
```

All of these are required; the target errors on any that is unset. Quote the enode — it contains `@`
and `:`.

**The `cp` is not optional.** The Makefile hard-includes `config.mk`, and the file is not in the
repository, so without it every target stops with `No rule to make target 'config.mk'` before
building anything. You can also set the values above in that file rather than passing them each time;
command-line values win. **Keep `config.mk` out of any repository you push — it holds your sequencing
key.**

The first build takes roughly 15–30 minutes. The target then creates
`.local_gateway_and_follower/` with a `compose.yml`, `.env`, a fresh per-VM JWT and the chain files
fetched from the portal, starts the three containers, and **prints the registration row and the exact
command for UniFi to run.** Send them that.

Re-running `make start-gateway` is safe: it refreshes chain metadata without regenerating your JWT or
your sequencing key.

### 4. Get registered — both halves

Registration is two separate things on UniFi's side, and they fail differently. Make sure both are
done:

| | If it is missing |
|---|---|
| **Registry row** — your `[url, address, jwt]` | You are not in the rotation at all and never build a block. |
| **Transaction fan-out** — your URL, paired with your signer address | You rotate into duty and build, but receive **no user transactions**. If the URL is added without the signer address, you receive transactions but your leg is advisory: another gateway's acceptance can mask your rejection, and a user gets a hash for a transaction you refused. |

Both hot-reload on UniFi's side, so neither needs a restart anywhere. Allow up to 30 seconds.

Send UniFi: your gateway URL (`http://<your-ip>:<GATEWAY_PORT>`) and your signer address from step 2.

## Verify

**Locally, first:**

```bash
docker ps --filter name=based-op    # based-op-gateway, based-op-node, based-op-geth all Up

make logs-gateway
# healthy: state=Sorting or state=WaitingForNewPayload
# expected at first: state=Syncing, until the sidecars catch up
```

The gateway leaves `Syncing` only once its sidecars have peered with the main node and pulled the
chain. If it stays there for more than a few minutes, it is almost always p2p reachability — see
troubleshooting.

**Then confirm registration**, from anywhere that can reach the portal:

```bash
curl -s -X POST <PORTAL> -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"registry_registeredGateways","params":[]}' | jq

curl -s -X POST <PORTAL> -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"registry_currentGateway","params":[]}' | jq
```

Your URL should appear in the list. Wait one rotation interval and re-run the second call — the
answer should change as rotation advances, and should eventually be you.

**Then watch yourself take a turn:** `make logs-gateway` and wait for the state to reach `Sorting`.
You are now building blocks. Finally, send a few test transactions across several rotation windows to
confirm they land regardless of which gateway is sequencing.

## Troubleshooting

**Stuck in `Syncing`.** Almost always p2p. Confirm your firewall permits inbound TCP *and* UDP on
both gossip ports — peering is bidirectional — and that the main node's gossip values in your `.env`
are non-empty. A blank enode is your own configuration; blank gossip values mean the portal fetch
failed, so re-run `make start-gateway`.

**A container exits at startup with a fork-activation or fork-ordering error.** Chain-specific, and
absent entirely on a chain whose forks are all active from genesis: activation times come from the
`genesis.json` the portal serves and the templates ship no `--override.<fork>` flag. It arises where a
chain activates a fork at a timestamp partway through its life — as UniFi Testnet does for Isthmus —
and your node's view of that timestamp disagrees with the network's. If you hit it, ask UniFi for the
canonical timestamp and set it in both the execution and consensus client commands in your local
`compose.yml`, then recreate. **Never choose a timestamp yourself** — a value the network does not
share forks you off the canonical chain.

**The portal logs authentication errors against your gateway.** The JWT in your registry row and the
one in your local `config/jwt` have diverged. This happens when the row was edited by hand. Have one
side replaced with the other's value and restart.

**You are registered but never asked for a payload.** Either rotation has not reached you yet, or
your round-trip time to the main node exceeds the portal's gateway timeout. Ask UniFi to check the
latter.

**You are registered but receive no transactions.** You are in the registry but not in the
transaction fan-out — the second half of step 4.

## Operating

- **Upgrading.** Pull the new revision and recreate the container. The other gateways keep serving
  during your restart, so users see no outage. Do not use `make start-gateway` to upgrade; it is a
  first-deploy target and refuses once `.env` exists.
- **Rotating your key or JWT.** Stop the gateway, have UniFi remove your row, regenerate, and deploy
  again. There is no in-place update. If the *signing key* changed, UniFi must also update your entry
  in the transaction fan-out — a stale signer address leaves your leg advisory.
- **Leaving.** Stop the containers and ask UniFi to remove your registry row and your fan-out
  entries. Both take effect within about 30 seconds.
- **Running several gateways.** Each needs its own VM, its own sequencing key and its own address,
  and each must be registered separately.

## Full runbook

This page is the overview. The complete guide — including the registry's upsert semantics, the
transaction fan-out details and every failure mode with its exact log lines — is in the private
repository:
[`doc/gateway/external-gateway-deployment.md`](https://github.com/PufferFinance/unifi-op/blob/main/doc/gateway/external-gateway-deployment.md).
There is also an agent-facing runbook at
[`.claude/skills/deploy-gateway-node`](https://github.com/PufferFinance/unifi-op/blob/main/.claude/skills/deploy-gateway-node/SKILL.md)
if you drive the deployment with Claude Code, and
[`doc/gateway/registry-operations.md`](https://github.com/PufferFinance/unifi-op/blob/main/doc/gateway/registry-operations.md)
for how the registry itself behaves.
