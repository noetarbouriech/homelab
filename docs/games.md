# Game servers — Crafty Controller runbook

Everything game-related lives in `k8s/apps/games/crafty/` (namespace `games`,
applied by the Flux `Kustomization crafty` in namespace `games`) and is deployed
by commit + push to `main`. Crafty Controller is both the panel and the runtime:
each Minecraft server is a **child process inside the Crafty pod**, so nothing
here needs a Docker socket, and it works on Talos.

Server definitions are deliberately *not* in this repo. You create and manage
them in the panel; their state lives in Crafty's SQLite database and on the
`servers` PVC. Nothing about an individual server is ever committed or pushed.

## Architecture

```text
browser -> https://mc.home.noe-t.dev
           DNS: *.home.noe-t.dev -> 10.40.1.0 (wildcard, no per-name record)
           cilium-gateway (wildcard TLS cert)
             |
             +-- HTTPS (BackendTLSPolicy, System CAs) -> crafty Service :8443

players -> 10.40.1.5:<port> -> crafty Service (L2-announced LB IP)
             |
             v
        pod crafty-0 (namespace games)
          crafty :8443       panel (TLS)
          crafty :25565-25579 Minecraft servers (child JVMs)
          volumes: config / servers / backups / logs + panel cert
```

## Fixed facts

| Item | Value |
| --- | --- |
| Panel | `https://mc.home.noe-t.dev` (through the Gateway) |
| Panel, direct | `https://10.40.1.5:8443` (same cert) |
| Players | `10.40.1.5:25565-25579` TCP, `19132` UDP |
| Namespace | `games`, StatefulSet `crafty` (replicas 1, pod `crafty-0`) |
| Image | `registry.gitlab.com/crafty-controller/crafty-4:4.11.0` |
| LB IP | `10.40.1.5`, pinned via `lbipam.cilium.io/ips` |
| PVCs | `*-crafty-0`: config 2Gi, servers 40Gi, backups 30Gi, logs 2Gi |
| Panel cert | `Certificate crafty-panel` -> secret `crafty-panel-tls` |
| Gateway hop | `BackendTLSPolicy crafty`, hostname `mc.home.noe-t.dev` |
| Max servers | 15 (one per pre-declared port) |
| Pod memory | limit 12Gi; the sum of all servers' `-Xmx` must fit under it |

All four PVCs use `piraeus-replicated` (this cluster has no default
StorageClass, so the class is always explicit).

## Connecting

**Players** use `10.40.1.5:<port>`. Names under `home.noe-t.dev` cannot carry
Minecraft: the wildcard resolves them to the Gateway, which only speaks HTTP.
Real per-server hostnames would need external-dns records; IP:port is what we
chose instead.

**You** open `https://mc.home.noe-t.dev`. That name needs no DNS work — the
`*.home.noe-t.dev` wildcard already points at the Gateway, and the Gateway's
listener holds the wildcard certificate. The Gateway to Crafty hop is TLS,
which is why this repo runs Cilium >= 1.20.2 (see *Upgrades* below).

## First login

Crafty prints the initial admin (or anti-lockout recovery) credentials to the
container log:

```sh
kubectl -n games logs sts/crafty | grep -i -A3 -E 'account|password'
```

Change the password and enable TOTP immediately: the panel is reachable from
anything on the LAN.

## Creating a server

1. Panel -> **Servers** -> **Create New Server**.
2. Type `Minecraft Java`, pick the core (Paper/Vanilla/Forge/Fabric/...),
   version, and **a port from `25565-25579`** — Crafty writes it into
   `server.properties` as `server-port`. Use a port no other server holds.
3. Set min/max RAM (this becomes `-Xms`/`-Xmx`) and accept the EULA prompt.
4. Start it, then connect with a client to `10.40.1.5:<port>`.

Nothing above touches git, which is the point of the pre-declared port range.

## Backups

Crafty creates a default backup configuration for every server and also backs up
before updates. Backups are zip archives under `/crafty/backups`, i.e. on
`backups-crafty-0`, which sits on the **same Piraeus pool as the worlds**: that
protects you from a bad world or a wrong deletion, not from losing the cluster.
An offsite copy (restic/VolSync to Garage or S3) is an open follow-up.

Restore happens in the panel: *Backups* -> pick an archive -> restore (in place
or with a server-directory reset).

## Upgrades

### Crafty image

Change the tag in `statefulset.yaml`, commit, push. Because Crafty stops all
servers on SIGTERM and there is only one replica, **an image bump is a full
outage** — do it when nobody is playing, and keep
`terminationGracePeriodSeconds` (currently 180s) above your slowest server stop.

Before a version bump, snapshot the config volume: Crafty's DB migrations are
one-way and no downgrade procedure exists.

```sh
kubectl -n games get pvc config-crafty-0   # confirm the claim name
```

Then create a VolumeSnapshot of it (`piraeus-snapshots` is the default class).

### Cilium (why it is coupled to this app)

The Gateway to Crafty hop uses Gateway API `BackendTLSPolicy`, which Cilium only
implements from **1.20**. The upgrade in this repo followed Cilium's documented
order, and that order matters if it is ever redone:

1. Gateway API CRDs **v1.6.2** in
   `k8s/infra/networking/cilium/config/kustomization.yaml` — 1.20 requires
   >= v1.6.1, where `tlsroutes` moved to the v1 API.
2. Cilium **1.19.8** (latest patch of the then-current minor).
3. Cilium **1.20.2** plus `upgradeCompatibility: "1.19"` (the minor originally
   installed) so the datapath is not disrupted.

After any Cilium change, check that the pre-existing apps still answer
(`grafana.home.noe-t.dev`, `home-assistant.home.noe-t.dev`) and that the LB IP
is still announced (`status.loadBalancer.ingress[0].ip` of the gateway Service).

### Certificate renewal

cert-manager renews at roughly day 60 of a 90-day certificate. The mounted files
refresh on their own, but **Crafty reads its certificate only at startup**, so
restart the pod after a renewal:

```sh
kubectl -n games rollout restart sts/crafty
```

Miss it and nothing breaks immediately — the served certificate stays valid
until its expiry date; after that the Gateway to Crafty hop fails and the panel
returns 502 until the pod is restarted.

## Operations and troubleshooting

- Reconcile after a push: `flux reconcile kustomization crafty -n games`.
- Panel is a 502/503 from the Gateway: check the policy and the backend cert.
  `kubectl -n games get backendtlspolicy crafty -o yaml` (an `Accepted` or
  `ResolvedRefs` condition of `False` means Cilium rejected the trust anchor),
  then:

  ```sh
  openssl s_client -connect 10.40.1.5:8443 \
    -servername mc.home.noe-t.dev </dev/null | openssl x509 -noout -issuer -dates
  ```

- Pod stuck in `ContainerCreating`: it is waiting for the certificate secret.
  `kubectl -n games describe pod crafty-0` shows the mount error, and
  `kubectl -n games get certificate crafty-panel` shows why issuance failed.
- Crafty logs a permission error reading the certificate: raise `defaultMode`
  from `0440` to `0444` in `statefulset.yaml`.
- Service has no external IP: the address is taken or the label no longer
  matches. `kubectl -n games describe svc crafty` shows the LB-IPAM event; pick
  another free address in `10.40.1.0/24` (avoid the pool's first and last).
- Need ports beyond the 15, or a plugin web map (Dynmap 8123, BlueMap 8100):
  add the port to `service.yaml`, commit, push, then allocate it in the panel.
- Node maintenance: scale the StatefulSet to 0 and back. **Do not** remove
  `games` from `k8s/apps/kustomization.yaml` while servers exist — that prunes
  the namespace and with it the PVC objects (the DRBD volumes survive
  `Retain`, but re-binding them is manual work).
- Server feels laggy on world writes: `servers-crafty-0` is DRBD-replicated
  (2 copies). Switching that template to `piraeus-local` trades redundancy for
  write latency and needs the data copied to the new volume.

## Other games later

Crafty's non-Minecraft support is the *Experimental SteamCMD Alpha*, whose two
documentation pages are empty and whose catalog endpoint
(`jars.arcadiatech.org/steamcmd.json`) returned 404 when this was set up — do
not rely on it for Valheim/Rust/Terraria.

Instead add them as their own small workloads in the `games` namespace with a
mature upstream image, exposed either through a second Service labelled
`type: public-lb` (its own IP from the same pool) or on the same `10.40.1.5`
using `lbipam.cilium.io/sharing-key`. Keep those manifests out of this repo if
they contain anything server-specific.

## What is intentionally not in git

Server names, ports in use, worlds, RCON/OP passwords, player data and backups.
The only thing this repo pins is the *range* of available ports, which is why
adding a server is a panel action and not a commit.
