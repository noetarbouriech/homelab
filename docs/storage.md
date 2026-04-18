# Storage

Storage is managed with Piraeus.

## Listing pools

```sh
kubectl -n piraeus-datastore exec deploy/linstor-controller -- linstor storage-pool list
```

## Listing volumes

```sh
kubectl -n piraeus-datastore exec deploy/linstor-controller -- linstor resource list-volumes
```
