---
title: Deploy a Follower Node
slug: /developers/node-operators/follower-node
---

# Deploy a Follower Node

A **follower** is an execution client and consensus client pair that tracks the canonical chain and
applies [preconfirmation fragments](../concepts/based-op-stack.md) as they arrive. It does not build
blocks, hold a sequencing key, produce proofs, or appear in the gateway registry.

**Why you would run one:** to serve RPC that reflects preconfirmed state — a transaction is visible,
with a receipt, before it is in a sealed block — without putting that traffic on a
sequencing-adjacent node. This is the shape a public RPC endpoint takes.

```
   UniFi main node                              Your follower VM
   ┌──────────────────────────┐                 ┌──────────────────────────┐
   │ op-geth   op-node        │ ── L2 p2p ───►  │ follower-op-node         │
   │ Portal    Registry       │ ── fragments ─► │ follower-op-geth         │
   │ Gateways ────────────────┼── frag gossip ► │   └─ serves your RPC     │
   └──────────────────────────┘                 └──────────────────────────┘
```

## What makes this the cheap option

- **No key of any kind.** No sequencing key, no attestation key. The only secret is a JWT generated
  locally and used solely between your own two containers.
- **No registration and no coordination.** Nothing has to be added on UniFi's side. Your `op-node`
  joins the fragment mesh **as a reader** by itself.
- **No inbound firewall rules required.** A follower dials out and holds its peers with all inbound
  traffic blocked. Open inbound p2p only if you want to accept peers.
- **No image of its own.** It runs the same two images every other role is built from.

## Before you start

Work through the [common prerequisites](./index.md#common-prerequisites) — hardware, Docker, and
especially NTP — and collect `PORTAL`, `TXPROXY` and your L1 endpoints from
[what UniFi gives you](./index.md#what-unifi-gives-you-when-you-are-onboarded).

:::danger PORTAL must be a literal IPv4 address
`make start-follower` parses the host out of `PORTAL` and hard-fails with
`PORTAL does not resolve to an IPv4 host` on anything that is not a dotted quad (or `localhost`). It
needs the literal address because it rewrites the portal-reported gossip multiaddr to that host. If
you were given a hostname, resolve it yourself and pass the address.
:::

## Deploy

```bash
git clone https://github.com/PufferFinance/unifi-op.git
cd unifi-op
git submodule update --init --recursive

cd devnet-deployment
cp config.mk.example config.mk    # required — see below

make start-follower \
    PORTAL=http://<main-node-ipv4>:<portal-port> \
    TXPROXY=http://<main-node-ipv4>:<txproxy-port> \
    FOLLOWER_IP=<this-vm's-own-ip> \
    L1_RPC_URL=<l1-rpc> \
    L1_BEACON_RPC_URL=<l1-beacon>
```

All five are required — the target refuses to run without them. The first build takes roughly 15–30
minutes; after that a redeploy is quick.

What it does: builds the two images, creates `.local_follower_node/` with a `compose.yml`, an `.env`,
a fresh `config/jwt`, and fetches `rollup.json`, `genesis.json` and the chain ID from the portal.
Then it starts both containers detached.

**The `cp` is not optional.** The Makefile hard-includes `config.mk`, and the file is not in the
repository, so without it every target stops with `No rule to make target 'config.mk'` before
building anything. A follower needs none of the key entries in the template; leave them as they are.

You can also set `PORTAL`, `TXPROXY`, `FOLLOWER_IP` and the L1 endpoints in that `config.mk` rather
than passing them each time — command-line values win. Keep `config.mk` out of any repository you
push; for a gateway it carries a private key.

### Choosing `FOLLOWER_IP`

It feeds both the execution client's advertised address and the consensus client's. Get it wrong and
the node still syncs, but its inbound reachability on the fragment mesh is degraded.

- **On a private network with the main node, use your internal address.** Advertising a public
  address you do not accept inbound on achieves nothing and pushes your own traffic out over the
  internet.
- **Never use `0.0.0.0`.** It is the template placeholder, not a valid advertised address.
- Use your public address only if you actually intend to accept inbound p2p and have opened the
  ports for it.

### Ports

The template's default ports are deliberately chosen not to collide with the other roles, so several
stacks can share one VM. On a dedicated follower host you will usually want the execution client's
HTTP and WS ports on something your load balancer already expects; override them in
`.local_follower_node/.env`, which is the intended place.

:::warning
Moving the RPC and WS ports is a breaking change for everything pointing at them — load-balancer
named ports, health checks, cloud firewall rules and external monitors. Decide once, before you put
traffic on it.
:::

The execution client exposes `eth`, `net`, `web3` and `txpool` over HTTP and WS. `debug` and `admin`
are deliberately not enabled.

## Verify, in this order

Do not stop at "the containers are up". Each of these catches a different failure, and the last one
is the only check that proves preconfirmations are working.

**1. It did not re-initialise or rewind.** This fails silently and costs a full resync:

```bash
docker logs follower-op-geth --since 5m 2>&1 | grep -c 'initializing geth datadir'   # 0 on a reused datadir
docker logs follower-op-geth --since 5m 2>&1 | grep -ci 'rewinding blockchain'       # 0
docker logs follower-op-geth --since 5m 2>&1 | grep 'Loaded most recent local block' # a sane height
docker logs follower-op-geth --since 5m 2>&1 | grep -ci 'permission denied'          # 0
```

**2. It joined the mesh as a reader.** Expect a line saying this node is not a registered gateway and
will mesh as a reader only. **That is success, not a warning** — it is what a follower is:

```bash
docker logs follower-op-node 2>&1 | grep -E 'reader only|added mesh peer'
```

Its *absence* is ambiguous, so do not read that as failure on its own. The line is written only after
the node actually attempts to announce itself and the registry replies that it is not a gateway. If
the address it would announce is unroutable — which is what `FOLLOWER_IP=0.0.0.0` gives you — nothing
is announced and you get `not publishing own p2p addr` instead. That one means `FOLLOWER_IP` is
wrong, not that meshing failed. Grep for both.

**3. Fragments are arriving.** Sample at least 60 seconds after start; the consensus client comes up
second:

```bash
docker logs --since 10m follower-op-node 2>&1 | grep -c 'Successfully sent frag to engine'   # > 0
docker logs --since 10m follower-op-node 2>&1 | grep -ci 'Method not found'                  # 0
```

`Method not found` means the execution client does not implement the fragment engine methods — i.e.
it is not a based execution client. Treat that second grep as a hint rather than a gate: the exact
wording depends on which component rejects the call, and an execution client's own phrasing for an
unimplemented method may not contain that string. The positive count above it is the check that
matters; a zero here is not evidence of health.

**4. Preconfirmed state is actually being served.** `unsafe` must run *ahead* of `safe`:

```bash
curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"optimism_syncStatus","params":[],"id":1}' \
  http://127.0.0.1:<OP_NODE_RPC_PORT> \
  | jq '{unsafe: .result.unsafe_l2.number, safe: .result.safe_l2.number}'
```

Note the `.result` prefix — without it every field reads `null`, which looks like a dead node. If the
two numbers are **equal**, you are tracking canonical blocks only and the preconfirmation path is not
working, *even though the block height looks perfectly healthy*. Height does not discriminate here.

Once it is serving, read
[Preconfirmation RPC Semantics](../reference/preconfirmation-rpc-semantics.md) — it is the contract
for what your endpoint now returns during the preconfirmation window.

## Reading the logs without false alarms

**These are not errors.** On a healthy follower they appear routinely, at a substantial fraction of
fragments applied:

- `frag is invalid … no unsealed block was opened`
- `Resetting unsealed block after two forkchoice updates`

They are normal when joining mid-block. **A monitoring rule that asserts zero will fire on a healthy
node.** There is no published baseline rate, so do not take a threshold from this page — establish
your own by watching the ratio to fragments applied over a stable window, and if you run two
followers, compare them against each other rather than against a number.

Two more things that look like incidents and are not:

- **Counts are elevated for roughly the first 10 minutes after any restart.** Catch-up inflates them
  several-fold and depresses the fragment rate. Wait 20 minutes before concluding anything from a
  rate.
- **`safe` can appear to move backwards after a restart.** The derivation pipeline resets to its last
  safe checkpoint and re-derives forward, so the gap balloons and then shrinks. Not data loss.

## Operating

```bash
make logs-follower
make stop-follower
```

- **Re-running `make start-follower` does not quietly re-do the setup — it refuses.** The target
  stops as soon as `.local_follower_node/.env` exists and tells you to bring the containers up
  directly instead. Only `make start-follower FORCE=1` proceeds, and it backs the `.env` up first
  before refreshing the main node's gossip and enode values and your `FOLLOWER_IP`. Neither path
  regenerates the JWT, and both refuse while the containers are up.
- **It will not re-copy `compose.yml` once `.env` exists.** After the first deploy, edit
  `.local_follower_node/compose.yml` directly and `docker compose up -d`. To force a full
  re-initialisation, delete `.local_follower_node/.env`.
- **Do not use `make start-follower` to upgrade.** It is a first-deploy target. Follow the upgrade
  procedure in the repository instead.
- **Shutdown takes time on purpose.** The archive execution client is given five minutes to flush
  state on stop. Killing it sooner rewinds the node to its last on-disk state on the next start.

## Troubleshooting

**A container exits at startup with a fork-activation or fork-ordering error.** This depends entirely
on the chain you are joining, so there is nothing to do here in the general case: the canonical
activation times come from the `genesis.json` the portal serves, and the templates deliberately ship
no `--override.<fork>` flag.

It only arises on a chain that activates a fork at a timestamp partway through its life rather than
from genesis, and where your node's view of that timestamp disagrees with the network's. UniFi Testnet
is such a chain — its Isthmus activation is a specific timestamp, and op-geth additionally requires
the Prague EIPs to be active no later than Isthmus, so a genesis whose `pragueTime` is later than its
`isthmusTime` makes the execution client refuse to start. A chain whose forks are all active from
genesis has no such window and needs no override.

If you do hit it, **ask UniFi for the canonical timestamp** and set that value in both the execution
and consensus client commands in your local `compose.yml`, then recreate. **Never pick a timestamp
yourself** — a value the network does not share forks you off the canonical chain, and you will never
apply a fragment.

**It syncs blocks but applies no fragments.** Check verification step 4 above rather than the block
height. The usual causes are a `PORTAL` the node cannot reach (fragment signatures are validated
against the registry's expected signer, so the portal is required, not optional) or an `op-node`
built without based support.

**It cannot find peers.** Check `FOLLOWER_IP` is a routable address for the network you share with
the main node, and that your egress permits the main node's p2p ports on both TCP and UDP.

## Full runbook

This page is the overview. The step-by-step runbook, including the failure modes above with their
exact log lines, is in the private repository:
[`.claude/skills/deploy-follower-node`](https://github.com/PufferFinance/unifi-op/blob/main/.claude/skills/deploy-follower-node/SKILL.md).
It is written to be executed by an AI coding agent as well as read — if you use Claude Code, point it
at that skill and it will drive the deployment.
