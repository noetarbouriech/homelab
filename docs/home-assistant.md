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
            otbr (bnutzer/otbr-tcp, hostNetwork node1)            REST :8081 (10.40.0.11:8081)
                |  Thread border router; wpan0 <-> enp1s0 (IPv6 RA/RIO, mDNS)
            matter-server (matter-js/matterjs-server, hostNetwork node1)  WS :5580 (ws://10.40.0.11:5580/ws)
                |
  HA Thread  integration -> OTBR URL   http://10.40.0.11:8081   (creates/owns ha-thread network)
  HA Matter  integration -> Server URL ws://10.40.0.11:5580/ws   (Matter controller / fabric)
  HA Tydom   integration -> host 10.40.0.101 (Cloud mode: Delta Dore account)
```

Device commissioning for Matter-over-Thread (IKEA TRETAKT/VALLHORN/…) is done by a phone
(HA Companion app) over BLE — no Bluetooth hardware is used on the server.

## Fixed facts

| Item | Value |
| --- | --- |
| Tydom gateway IP / MAC | `10.40.0.101` / `001a250a0a63` (12 hex, no separators) |
| Dongle-M IP / MAC | `10.40.0.102` / `1c:c3:ab:12:69:cf` |
| Dongle mode | Thread RCP (EFR32MG24, firmware `SL-OPENTHREAD/2.4.5.0…`) |
| OTBR REST | `http://10.40.0.11:8081` (health: `GET /node/state`) |
| Matter Server WS | `ws://10.40.0.11:5580/ws` (dashboard `http://10.40.0.11:5580/`) |
| HA web | `https://home-assistant.home.noe-t.dev` |
| Deltadore component | `deltadore_tydom` **v0.31**, installed by the chart initScript into `/config/custom_components` |
| Thread/matter state dirs | PVC `otbr-thread-state` -> `/var/lib/thread`; PVC `matter-server-data` -> `/data` (both `piraeus-local`, node1) |

## Secrets (Bitwarden Secrets Manager -> ExternalSecret)

Create in the SM project referenced by the `bitwarden-secretsmanager` ClusterSecretStore:

- `deltadore-tydom-email` — Delta Dore account email (Cloud mode)
- `deltadore-tydom-password` — Delta Dore account password (Cloud mode)

The `ExternalSecret tydom-credentials` in namespace `iot` then syncs a secret with keys
`email` / `password`. It reports NotReady until the items exist.

## First-time HA config entries (one-time, done in the HA UI)

1. **Thread** — Settings > Devices & services > Add > OpenThread Border Router / Thread:
   URL `http://10.40.0.11:8081`. HA forms a `ha-thread-*` network (check
   Settings > Thread; border router state should leave "disabled" and become leader).
2. **Matter** — Add integration > Matter; connection method: *already running in a
   custom container*, then URL `ws://10.40.0.11:5580/ws`.
3. **Delta Dore Tydom** — Add integration > Delta Dore Tydom: host `10.40.0.101`,
   MAC `001a250a0a63`, Delta Dore email/password (Cloud mode) from the Bitwarden
   entries above; default refresh interval is fine.

Then, to add IKEA Matter-over-Thread devices: in the HA Companion app, Settings >
Thread > "Send credentials to phone" (so the phone can hand the Thread network
credentials to the device over BLE), stay near the dongle, and use
Settings > Matter > Add device (scan the device QR / enter pairing code). The device
joins `ha-thread-*` and appears in HA.

## Operations & troubleshooting

- Reconcile/deploy: commit + push, then `flux reconcile kustomization home-assistant -n iot`.
- OTBR health from the HA pod: `curl http://10.40.0.11:8081/node/state`
  (expect `"leader"`/`"router"` once HA formed the network; `"disabled"` before that).
  RCP link check: `kubectl -n iot logs deploy/otbr | grep -i "co-processor"`.
- Matter server health: `curl -I http://10.40.0.11:5580/` (HTTP 200).
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
