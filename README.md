# Homelab

## Update Talos config

```sh
talosctl apply-config --talosconfig=./clusterconfig/talosconfig --nodes=10.40.0.11 --file=./clusterconfig/homelab-cluster-node1.yaml;
talosctl apply-config --talosconfig=./clusterconfig/talosconfig --nodes=10.40.0.12 --file=./clusterconfig/homelab-cluster-node2.yaml;
talosctl apply-config --talosconfig=./clusterconfig/talosconfig --nodes=10.40.0.13 --file=./clusterconfig/homelab-cluster-node3.yaml;
``` 

## Shutting down the cluster

```sh
talosctl shutdown --talosconfig clusterconfig/talosconfig
```
