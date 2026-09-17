# Game servers — Crafty Controller

`k8s/apps/games/` (namespace `games`): `crafty` is the panel and the runtime
(each server is a child process in the pod), `mc-router` routes Minecraft
connections to the right server by hostname. Servers are created in the panel
and never appear in this repo.

| Item | Value |
| --- | --- |
| Panel | `https://mc.home.noe-t.dev` through the Gateway |
| Players | `<server>.home.noe-t.dev` (port 25565) |
| Workload | StatefulSet `crafty` (1 replica), Deployment `mc-router` |
| PVCs | `config`, `servers`, `backups`, `logs` |

Every name resolves through the existing `*.home.noe-t.dev` wildcard, so no DNS
record is ever added for a server.

## Creating a server

1. Panel -> Servers -> Create -> Minecraft Java -> core + version -> any free
   port (25565, 25566, ...) -> RAM -> accept the EULA -> start.
2. Register its hostname once, from your machine:

```sh
kubectl -n games port-forward svc/mc-router-api 8080:8080 &
curl -X POST localhost:8080/routes -H 'Content-Type: application/json' -d '{
  "serverAddress": "survival.home.noe-t.dev",
  "backend": "crafty-headless.games.svc.cluster.local:25565"
}'
```

3. Players connect to `survival.home.noe-t.dev`. mc-router refuses connections
   that do not name a registered hostname, which also keeps port scanners out.

Initial admin credentials are in the container log:

```sh
kubectl -n games logs sts/crafty | grep -i -A3 -E 'account|password'
```

## Operating

- Reconcile: `flux reconcile kustomization crafty -n games` (and `mc-router`)
- Port-forward the panel: `kubectl -n games port-forward svc/crafty 8443:8443`
- Route list: `curl -s localhost:8080/routes` (same port-forward),
  remove one with `curl -X DELETE localhost:8080/routes/<hostname>`
- Image bumps restart every server (one pod, no rolling update); snapshot
  `config-crafty-0` first, its migrations are one-way.
- Panel TLS: the Gateway terminates with the wildcard certificate and does not
  verify Crafty's self-signed certificate, so a browser warning cannot appear.
- Do not remove `games` from `k8s/apps/kustomization.yaml` while servers exist:
  that prunes the namespace and its PVC objects.
