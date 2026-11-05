# Home Assistant — Deltadore Tydom + Thread/Matter runbook

Everything HA-related lives in `k8s/apps/misc/home-assistant/` (namespace `iot`, applied
by the Flux `Kustomization home-assistant` in namespace `iot`) and is deployed by
commit + push to `main`. HA runs as the pajikos `home-assistant` Helm chart with
`hostNetwork`, pinned to node1 via the otbr/matter-server deployments.

## Architecture

```
10.40.0.101  Delta Dore Tydom gateway   -- local HTTPS/WS API -->  HA (helm, hostNetwork, node1)
10.40.0.102  SONOFF Dongle-M (Thread RCP, TCP :6638, baud 115200, no flow control)
                ^-- socat --v
            otbr (bnutzer/otbr-tcp, hostNetwork node1)            REST :8081 (svc otbr)
                |  Thread border router; wpan0 <-> enp1s0 (IPv6 RA/RIO, mDNS)
            matter-server (matter-js/matterjs-server, hostNetwork node1)  WS :5580 (svc matter-server)
                |
  HA Thread  integration -> OTBR URL   http://otbr:8081   (creates/owns ha-thread network)
  HA Matter  integration -> Server URL ws://matter-server:5580/ws   (Matter controller / fabric)
  HA Tydom   integration -> host 10.40.0.101 (Cloud mode: Delta Dore account)
```

Addressing is stateless: otbr and matter-server are exposed as headless Services in
namespace `iot`, so HA (same namespace, `ClusterFirstWithHostNet`) reaches them by
DNS name (`otbr` / `matter-server`, FQDN `…iot.svc.cluster.local`) — no node IP is
stored anywhere. The only bare IPs left are the physical Tydom gateway and the dongle.

Device commissioning for Matter-over-Thread (IKEA TRETAKT/VALLHORN/…) is done by a phone
(HA Companion app) over BLE — no Bluetooth hardware is used on the server.

## Fixed facts

| Item | Value |
| --- | --- |
| Tydom gateway IP / MAC | `10.40.0.101` / `001A250A0A63` (12 hex, no separators) |
| Dongle-M IP / MAC | `10.40.0.102` / `1c:c3:ab:12:69:cf` |
| Dongle mode | Thread RCP (EFR32MG24, firmware `SL-OPENTHREAD/2.4.5.0…`) |
| OTBR REST | `http://otbr:8081` (health: `GET /node/state`) |
| Matter Server WS | `ws://matter-server:5580/ws` (dashboard `http://matter-server:5580/`) |
| HA web | `https://home-assistant.home.noe-t.dev` |
| Deltadore component | `deltadore_tydom` **v0.31**, installed by the chart initScript into `/config/custom_components` |
| Thread/matter state dirs | PVC `otbr-thread-state` -> `/var/lib/thread`; PVC `matter-server-data` -> `/data` (both `piraeus-local`, node1) |

## HTTP / reverse proxy settings — UI-managed, not YAML

Since HA 2026.8 the HTTP server settings (`use_x_forwarded_for`, `trusted_proxies`,
port, SSL profile, …) live in `/config/.storage/http` and are managed from
**Settings > System > Network > HTTP server**. The old `http:` block in
`configuration.yaml` is **ignored** and stops working in HA 2027.2.0.

- The Helm `templateConfig` deliberately contains no `http:` block.
- On an existing install the chart's `forceInit` merge never deletes keys, so the
  init script removes a leftover `http:` block once `/config/.storage/http`
  exists. There is nothing to do in the UI: the migrated values
  (`use_x_forwarded_for: true`, trusted proxy `10.40.0.0/24`) are already stored.
- On a fresh install the init script seeds `/config/.storage/http` from
  `configuration.trusted_proxies` in `helm.yaml` (chart 0.3.75+), so the
  reverse proxy works before the first UI visit.
- To change the reverse proxy later, use the UI, not `configuration.yaml`.
  The settings survive pod restarts because `.storage/http` is on the PVC.

## Upgrading Home Assistant

The chart/image are pinned in `k8s/apps/misc/home-assistant/app/helm.yaml`.
Current target: chart `0.3.90`, HA `2026.11.0`.

- **HACS is now built in as the Marketplace (2026.11).** HA removes
  `/config/custom_components/hacs` and migrates its data on first start. The
  chart init script intentionally no longer reinstalls the old HACS; do not add
  it back. Automations that used the `hacs` integration must use `marketplace`
  (e.g. `integration_entities('marketplace')`).
- **2026.11 entity naming** changes some dashboard labels (the entity IDs are
  unchanged). Review dashboards after the upgrade.
- **2026.10 auth**: legacy usernames are normalized (lowercase, no surrounding
  spaces). The only user here is `noe`, so no action is needed.
- The Delta Dore Tydom custom component stays pinned at `v0.31` and only logs
  the `via_device` deprecation (removed in 2027.8); it does not block 2026.11.
  Newer releases (`v0.35`) exist if the integration misbehaves.

## Secrets (Bitwarden Secrets Manager -> ExternalSecret)

Create in the SM project referenced by the `bitwarden-secretsmanager` ClusterSecretStore:

- `deltadore-tydom-email` — Delta Dore account email (Cloud mode)
- `deltadore-tydom-password` — Delta Dore account password (Cloud mode)

The `ExternalSecret tydom-credentials` in namespace `iot` then syncs a secret with keys
`email` / `password`. It reports NotReady until the items exist.

## HA config entries — created by the ha-bootstrap Job (no UI, no tokens)

The `ha-bootstrap` Job (namespace `iot`) creates all three config entries via HA's
REST config-flow API. Authentication is passwordless: HA runs the built-in
`trusted_networks` auth provider trusting the pod network (`10.244.0.0/22`,
`allow_bypass_login`), and the Job runs as a normal pod. Node IPs are deliberately
NOT trusted — they fall inside `http.trusted_proxies` (10.40.0.0/24) and the
provider refuses proxy-range addresses.

Entries created (idempotent — existing ones are skipped):

1. **Thread** (integration entry; border router info comes from mDNS discovery of
   the otbr deployment — Settings > Thread shows `ha-thread-*`).
2. **Matter** — server URL `ws://matter-server:5580/ws`.
3. **Delta Dore Tydom** — host `10.40.0.101`, MAC `001A250A0A63` (uppercase: the
   Delta Dore cloud API matches the MAC case-sensitively), Cloud-mode email/password
   from the `tydom-credentials` ExternalSecret.

Re-run after a HA PVC wipe or to repair entries:

```sh
kubectl -n iot delete job ha-bootstrap
kubectl -n iot create job --from=job/ha-bootstrap ha-bootstrap-rerun
```

Then, to add IKEA Matter-over-Thread devices: in the HA Companion app, Settings >
Thread > "Send credentials to phone" (so the phone can hand the Thread network
credentials to the device over BLE), stay near the dongle, and use
Settings > Matter > Add device (scan the device QR / enter pairing code). The device
joins `ha-thread-*` and appears in HA.

## Operations & troubleshooting

- Reconcile/deploy: commit + push, then `flux reconcile kustomization home-assistant -n iot`.
- OTBR health: `curl http://otbr:8081/node/state` from a cluster pod
  (expect `"leader"` once HA formed the network; `"disabled"` before that).
  RCP link check: `kubectl -n iot logs deploy/otbr | grep -i "co-processor"`.
- Matter server health: `curl -I http://matter-server:5580/` (HTTP 200).
- The names resolve only inside the cluster (HA is `ClusterFirstWithHostNet`); for a
  quick check from this machine use `kubectl -n iot exec deploy/otbr -- curl -s http://otbr:8081/node/state`.
- Restarting `otbr` keeps the Thread network: the dataset persists on the
  `otbr-thread-state` PVC. Deleting the PVC = new network + re-pair every device.
- Matter devices join but become unreachable / drop subscriptions:
  - IPv6/multicast blocked: Thread requires IPv6 + multicast on the LAN; the node
    must have IPv6 enabled (`net.ipv6.conf.all.disable_ipv6 = 0`) and accept RAs.
    No internet IPv6 needed.
  - Stateful firewalls (host, router, OTBR host) expiring UDP conntrack entries
    between device report cycles — the known Matter/Thread failure mode.
  - 2.4 GHz interference: switch the Thread channel (Settings > Thread) to 25/26.
- Images are rolling tags (`bnutzer/otbr-tcp:latest`, `matterjs-server:stable`) —
  pinned builds via the repo's dagger setup are an optional hardening follow-up.
- The HA core container + Matter server standalone is officially "unsupported but
  documented" by HA; accepted tradeoff for a containerized HA.
