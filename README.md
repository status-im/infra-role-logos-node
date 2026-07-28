# Description

This role provisions a [Logos Node](https://github.com/logos-co/logos-liblogos) running `logoscore` with configurable modules. It deploys a containerized Logos Node via Docker Compose with support for module loading, Consul service registration, and container monitoring.

Three modules are supported, each with its own config template, node info query and Consul service registration:

* `delivery_module` - wraps `libwaku` from [logos-delivery](https://github.com/logos-messaging/logos-delivery), provides Waku protocols (relay, store, filter, lightpush, mix) over TCP
* `storage_module` - content-addressed storage over TCP
* `blockchain_module` - cryptarchia consensus over QUIC

A node can run any combination of them. The `logos.dev` fleet runs one module per host for independent debugging, `logos.test` runs all three on combined nodes.

# Configuration

## Modules

Modules and their startup calls are configured via:
```yaml
logos_node_modules: ['delivery_module']
logos_node_module_calls:
  - 'delivery_module createNode @/conf/waku_config.json'
  - 'delivery_module start'
```

Base modules are always loaded before user modules:
```yaml
logos_node_base_modules: ['openmetrics']
```

Per-module tasks are enabled automatically based on `logos_node_modules`.

## Delivery Module

Node identity from Vault:
```yaml
logos_node_delivery_key: '{{ lookup("vault", "delivery/nodekeys", field=hostname) }}'
logos_node_delivery_mix_key: '{{ lookup("vault", "delivery/mixkeys", field=hostname) }}'
```

Protocols are defined as a single list, used both for templating the JSON config and for the Consul `protocols` metadata read by `wakucanary`:
```yaml
logos_node_delivery_protocols_enabled: ['relay', 'store', 'filter', 'lightpush', 'mix']
```

Discovery mechanisms are separate from protocols:
```yaml
logos_node_delivery_discv5: true
logos_node_delivery_kad_discovery: true
```

Store protocol requires a PostgreSQL connection:
```yaml
logos_node_delivery_store_db_url: 'postgres://user:pass@db-host:5432/dbname'
logos_node_delivery_store_retention_policy: 'size:1GB'
```

Kademlia bootstrap nodes are configured as a list of peer multiaddresses:
```yaml
logos_node_delivery_kad_bootstrap_nodes:
  - '/dns4/node1.example.com/tcp/30303/p2p/16Uiu2HAm...'
  - '/dns4/node2.example.com/tcp/30303/p2p/16Uiu2HAm...'
```

Websocket with TLS:
```yaml
logos_node_delivery_websocket_enabled: true
logos_node_delivery_websocket_secure_enabled: true
logos_node_delivery_websocket_domain: '{{ dns_entry }}'
```

## Storage Module

Nodes take one of two roles - `mp` (mix provider) or `rs` (regular storage):
```yaml
logos_node_storage_role: 'mp'
logos_node_storage_mix_pool_relays: []
logos_node_storage_bootstrap_nodes: []
```

Content can be preloaded on RS nodes via `logos_node_storage_preload_content`.

## Blockchain Module

Unlike the other two, the blockchain module uses a YAML config templated from Vault keys, and QUIC transport:
```yaml
logos_node_blockchain_deployment: 'devnet'
logos_node_blockchain_initial_peers: []
```

# Monitoring

## Consul

Each enabled module registers its own Consul services with health checks and metadata. Delivery nodes additionally expose `cluster-id` and `protocols` metadata, which the [Kuma canary](https://canary.infra.status.im) uses to build `wakucanary` checks.

## Container monitor

There's a [container monitor service](./MONITOR.md) that watches Docker events and updates Consul metadata on container restarts.
```yaml
logos_node_monitor_enabled: true
```

# Usage

You can re-create containers on the host using:
```
cd /docker/logos-node
docker-compose --compatibility up -d --force-recreate
```
Which will use the `docker-compose.yml` file in that directory.

Node info can be queried directly from a running module:
```
docker exec logos-node logoscore --config-dir /var/lib/logos/config call delivery_module getAvailableNodeInfoIDs
docker exec logos-node logoscore --config-dir /var/lib/logos/config call delivery_module getNodeInfo MyPeerId
```
# Requirements

Due to being part of Status infra this role assumes availability of certain things:

* Docker for running containers
* [Docker user namespace remapping](https://docs.docker.com/engine/security/userns-remap/) with `dockremap` user
* [Watchtower](https://github.com/containrrr/watchtower) for updating Docker images
* The [`iptables-persistent`](https://zertrin.org/projects/iptables-persistent/) module

# Known Limitations

* Modules expose no Prometheus scrape endpoint of their own - the `openmetrics` base module aggregates metrics from all loaded modules into a single endpoint.
* The blockchain module API only responds once initial block download completes, so its health check reports unhealthy while a node is still syncing.