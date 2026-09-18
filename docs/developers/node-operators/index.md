---
title: Run a UniFi Node
slug: /developers/node-operators
---

# Run a UniFi Node

**Who this is for:** developers who want their own RPC endpoint on UniFi, and infrastructure
providers who want to run sequencing infrastructure for the network.

UniFi is an OP Stack rollup with a based sequencing layer
([based-op](../concepts/based-op-stack.md)). That means there is more than one kind of node, and the
one you want depends on whether you are *reading* the chain or *building* blocks for it.

## The node roles

| Role | What it does | Runs it |
|---|---|---|
| **Follower** | Tracks the canonical chain and applies preconfirmation fragments. Serves RPC that reflects preconfirmed state. Builds nothing, holds no key. | **Anyone.** No coordination with UniFi beyond connection details. |
| **Gateway** | Builds blocks. Orders transactions, emits fragments, signs and seals. Takes turns with the other registered gateways. | **Third parties, by arrangement.** UniFi has to add you to the registry. |
| **TEE prover** | Executes blocks inside a trusted execution environment and produces the attested proofs that back [instant withdrawals](../concepts/tee-multi-prover.md). | **By arrangement, and needs TDX or SEV-SNP hardware.** |
| **Main node** | The rollup's `op-node`/`op-batcher`/`op-proposer`, the portal, the gateway registry and the transaction proxy. | **UniFi.** One per network. |

### Which one do you want?

- **"I need an RPC endpoint I control"** — a [follower](./follower-node.md). This is the common case.
  It needs no key, no registration and no inbound firewall rules, and it serves preconfirmed state,
  so it behaves like the public endpoint rather than like a plain OP Stack node.
- **"I want to sequence for UniFi"** — a [gateway](./gateway-node.md). Read the trust model on that
  page before you start; running one means holding a signing key and sharing a JWT with UniFi.
- **"I want to run a prover"** — talk to the UniFi team first. Proving needs attested hardware and a
  registration on L1, and the deployment is not self-service.

:::info
A gateway host runs a follower underneath the builder, so a gateway deployment gives you both. You
do not deploy them separately on the same machine.
:::

## Getting the code

The node software and the deployment templates live in **`unifi-op`**, which is currently a private
repository:

**[github.com/PufferFinance/unifi-op](https://github.com/PufferFinance/unifi-op)**

Access is granted to developers and infrastructure providers who want to run a node — contact the
UniFi team to be added, and mention which role you intend to run. The deep links to runbooks on the
following pages are inside that repository and will 404 until you have access.

Everything you need is in the repo: `devnet-deployment/` holds the compose templates and a `make`
target per role. Images are built locally from source; there is no separate image registry to be
granted access to.

### Getting it onto the machine

Both `unifi-op` and its `op-geth` submodule are private, so **whatever credential you use has to
cover both** — the build pulls the submodule itself and builds the execution client image from it. A
credential scoped to `unifi-op` alone is not enough.

How the tree reaches the host is up to you, and the rest of the deployment is identical either way:

- **Clone on the host.** Configure a credential first — an SSH key (agent forwarding works and leaves
  nothing behind on the host), a deploy key, or a personal access token in a credential helper.
- **Build the tree elsewhere and copy it over.** `git bundle` plus `scp` is the practical route for a
  host with no GitHub access of its own.

Without a credential in place, the deployment stops on its very first command:

```
$ git clone https://github.com/PufferFinance/unifi-op.git
fatal: could not read Username for 'https://github.com'
```

## What UniFi gives you when you are onboarded

Every role needs the same small set of connection details for the network you are joining. You
cannot derive these, and guessing them will silently fork you off the chain:

| Value | What it is |
|---|---|
| `PORTAL` | The main node's portal endpoint. **Must be given to you as a literal `http://<IPv4>:<port>` URL** — the tooling rejects DNS names (see the traps on each page). Ask which address applies to *your* node: the portal port is not open to the whole internet, and on a shared private network the reachable address is the main node's **internal** one. Its public address can be filtered even for a node on the same network. |
| `TXPROXY` | Where your node forwards transactions it receives. |
| `MAIN_OP_GETH_ENODE` | The main node's execution-client enode, used as your bootnode. |
| The L2 p2p endpoints | The main node's `op-node` and `op-geth` gossip addresses and ports, for your egress rules. |
| The canonical fork-activation timestamps | e.g. Isthmus. Needed only if your node hits a fork-order error at startup. **Never pick one yourself.** |

You supply your own L1 execution and beacon RPC endpoints (`L1_RPC_URL`, `L1_BEACON_RPC_URL`), or
use the ones UniFi uses. They must be on the L1 the network settles to — see
[Network & RPC Endpoints](../reference/rpc-endpoints.md).

The chain's `genesis.json` and `rollup.json` are fetched from the portal automatically during setup.
You do not need to be sent them.

## Common prerequisites

These apply to every role:

- **Linux x86_64**, at least 8 vCPU and 16 GB RAM.
- **Disk sized for an archive execution client.** The templates default to `--gcmode=archive`, which
  grows with chain history. Size it from the current chain, not from genesis, and give yourself
  headroom — 200 GB SSD is a floor, not a target.
- **Docker Engine and the `docker compose` plugin**, plus `git`, `make`, `curl` and `jq`. Rust and Go
  are not needed on the host; everything compiles inside Docker.
- **NTP running** (`chrony` or equivalent). This is not boilerplate: clock drift breaks Engine API
  JWT authentication between your own containers and destabilises p2p.
- **Outbound** to the main node's portal and p2p ports, your L1 endpoints, and 443 for image builds.
- **A `config.mk` in `devnet-deployment/`.** The Makefile hard-includes it and the file is not in the
  repository, so on a fresh clone *every* target stops immediately with
  `No rule to make target 'config.mk'` before doing any work. `cp config.mk.example config.mk` from
  inside `devnet-deployment` is the first step of any deployment — it is a prerequisite, not a
  convenience.

Inbound requirements differ by role and are covered on each page — a follower needs none, a gateway
does.

## Next

- [Deploy a follower node](./follower-node.md) — an RPC node that serves preconfirmed state.
- [Deploy a gateway](./gateway-node.md) — sequencing infrastructure.
