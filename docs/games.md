# Game servers — Crafty Controller

`k8s/apps/games/crafty/` (namespace `games`, Flux `Kustomization crafty`).
Crafty is both the panel and the runtime: each server is a child process in the
pod. Servers are created in the panel and never appear in this repo.

| Item | Value |
| --- | --- |
| Panel | `https://mc.home.noe-t.dev` |
| Players | `10.40.1.5:25565-25579` TCP, `19132` UDP |
| Workload | StatefulSet `crafty`, 1 replica |
| PVCs | `config`, `servers`, `backups`, `logs` |
| Cert | `Certificate crafty-panel` -> secret `crafty-panel-tls` |
| LB IP | pinned to `10.40.1.5` via `lbipam.cilium.io/ips` |

## Creating a server

Panel -> Servers -> Create -> Minecraft Java -> core + version -> port from
`25565-25579` -> RAM -> accept the EULA -> start.

Initial admin credentials are in the container log:

```sh
kubectl -n games logs sts/crafty | grep -i -A3 -E 'account|password'
```

## Operating

- Reconcile: `flux reconcile kustomization crafty -n games`
- Port-forward the panel: `kubectl -n games port-forward svc/crafty 8443:8443`
- Image bumps restart every server (one pod, no rolling update); snapshot
  `config-crafty-0` first, its migrations are one-way.
- Crafty reads its certificate only at startup, so restart the pod after a
  renewal: `kubectl -n games rollout restart sts/crafty`
- More than 15 servers, or plugin web maps (8123/8100): add the port to
  `service.yaml` and push.
- Do not remove `games` from `k8s/apps/kustomization.yaml` while servers exist:
  that prunes the namespace and its PVC objects.
