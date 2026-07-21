# zer0space

A nine-machine homelab built as a **fully stateless Docker Swarm** — public
documentation of how it works, why it is shaped this way, and what is still broken.

Seven nodes form a Swarm that owns no data. Two hosts outside the cluster hold
everything that must survive: one runs the databases, one holds the files. Any
Swarm node can be wiped and rebuilt without losing anything, and any service can be
scheduled anywhere.

> **This is a documentation repository.** It contains no deployable configuration
> and no secrets. Every address, domain and image path on these pages is a
> placeholder.

## Start here

| | |
|---|---|
| **[architecture.md](architecture.md)** | The nine machines, how the stateless split works, ingress and auth layers |
| **[journey.md](journey.md)** | How it got built — the Traefik detour, an overlay network that lied, making the Swarm forget, and a backup that isn't one yet |
| **[tech-stack.md](tech-stack.md)** | Every technology and why it was chosen — including what was evaluated and dropped |
| **[examples/](examples/)** | Two generic, copy-pasteable patterns extracted from this build |

## The short version

```
                     Cloudflare  (DNS · Tunnel · Access MFA)
                          │  outbound-only — no open inbound ports
                          ▼
     ┌────────────────────────────────────────────────┐
     │  DOCKER SWARM — 3 managers + 4 workers          │
     │  stateless: no local volumes, nothing pinned    │
     │  for storage reasons                            │
     └──────────┬──────────────────────┬───────────────┘
                │ NFS                  │ PostgreSQL
                ▼                      ▼
        ┌──────────────┐      ┌──────────────────┐
        │ zs-store-01  │      │ zs-state-01      │
        │ files        │      │ database · UI    │
        └──────────────┘      └──────────────────┘
```

**Ingress is a tunnel, not a reverse proxy.** `cloudflared` dials *out* to
Cloudflare and traffic comes back down that connection. No port forwarding, no
exposed origin IP, no TLS to manage. Traefik was tried first and removed — the
reasoning is [in the journey](journey.md#the-traefik-detour).

**Storage is what makes it stateless.** Swarm does not replicate volumes, so files
live on one NFS share mounted at the same path on every node, and structured data
lives in PostgreSQL off-cluster. Nothing is pinned to a node for storage reasons.

**Auth is two independent layers.** Cloudflare Access (e-mail OTP + MFA) at the
edge, and the application's own session auth inside. Neither trusts the other.

## What is actually running

Documentation that only shows the finished parts is not worth much, so:

| | |
|---|---|
| ✅ **Working** | 7-node Swarm, 3 managers · all persistent files on central NFS · application on PostgreSQL, genuinely stateless · GitOps deploys from git · nightly local backups with visible status · custom dashboard with per-user encrypted credential vault |
| 🔄 **Built, not switched on** | Cloudflare Tunnel and the Access policy — compose files and policy design exist, tunnel not yet deployed |
| ⚠️ **Known broken** | VXLAN overlay to the late-joining workers. Cluster and monitoring work around it; it is not fixed |
| ❌ **Missing** | Offsite backups · any database backup at all · a restore test |

That last row is the honest gap in this build. It is
[written up rather than smoothed over](journey.md#the-backup-that-isnt-one-yet).

## Four things worth stealing

1. **Ask what a component is actually for before adding it.** A reverse proxy is
   load-bearing when you terminate your own TLS and route your own DNS. Behind a
   tunnel that does both, it is mostly ceremony.
2. **"Ready" is a claim about the control plane, not the data plane.** Nodes can be
   healthy by every check the orchestrator makes and still fail to pass container
   traffic.
3. **Stateless is a constraint you accept early, not a property you add later.**
   Retrofitting it costs roughly the port you were avoiding.
4. **The dangerous backup failure is not the one that breaks.** It is the one that
   keeps reporting success after the data quietly moved somewhere else.

## May

The mascot watching over all this — seven interchangeable nodes, two anchors
that actually hold state, a tunnel that only calls out, an overlay that quietly
lied for a while. The full story is in
[`may (mascot)/story.md`](may%20(mascot)/story.md).

## Related repositories

The deployable configuration and application source live in separate, private
repositories. This one is documentation only.

## Note on placeholders

Addresses use the illustrative block `10.0.0.0/24`, domains are `example.com`, and
registry paths are generic. None of them correspond to the real deployment. Nothing
here is a credential, and nothing here is deployable as-is.
