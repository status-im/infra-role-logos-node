# Description

This role provisions a [Logos Node](https://github.com/logos-co/logos-liblogos) running `logoscore` with configurable modules. It deploys a containerized Logos Node via Docker Compose with support for module loading, Consul service registration, and container monitoring.

The first supported module is [logos-waku-module](https://github.com/logos-co/logos-waku-module), which wraps `libwaku` from [logos-delivery](https://github.com/logos-messaging/logos-delivery) to provide Waku protocols (relay, store, filter, lightpush, mix).

# Configuration

## Modules

Modules and their init commands are configured via:
```yaml
logos_node_modules: ['waku_module']
logos_node_module_commands:
  - 'waku_module.initWaku(@/conf/waku_config.json)'
  - 'waku_module.startWaku()'
```

Additional modules can be added to the list as they become available. Each module can have its own config file templated into the conf directory and its own init commands.

## Node Identity
```yaml
logos_node_key: '{{ lookup("vault", "delivery/nodekeys", field=hostname) }}'
logos_node_mix_key: '{{ lookup("vault", "delivery/mixkeys", field=hostname) }}'
```

## Waku Module

Protocols are disabled by default and enabled per-fleet in `group_vars`:
```yaml
logos_node_store: true
logos_node_filter: true
logos_node_lightpush: true
logos_node_mix: true
logos_node_discv5: true
logos_node_kad_discovery: true
```

Store protocol requires a PostgreSQL connection:
```yaml
logos_node_store_db_url: 'postgres://user:pass@db-host:5432/dbname'
logos_node_store_retention_policy: 'size:1GB'
```

Kademlia bootstrap nodes are configured as a list of peer multiaddresses:
```yaml
logos_node_kad_bootstrap_nodes:
  - '/dns4/node1.example.com/tcp/30303/p2p/16Uiu2HAm...'
  - '/dns4/node2.example.com/tcp/30303/p2p/16Uiu2HAm...'
```

Websocket with TLS:
```yaml
logos_node_websocket_enabled: true
logos_node_websocket_secure_enabled: true
logos_node_websocket_domain: '{{ dns_entry }}'
```

## Monitoring

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

# Requirements

Due to being part of Status infra this role assumes availability of certain things:

* Docker for running containers
* [Docker user namespace remapping](https://docs.docker.com/engine/security/userns-remap/) with `dockremap` user
* [Watchtower](https://github.com/containrrr/watchtower) for updating Docker images
* The [`iptables-persistent`](https://zertrin.org/projects/iptables-persistent/) module

# Known Limitations

* `libwaku` does not expose a REST API or metrics endpoint — peer info (peer ID, ENR, multiaddr) is extracted from container logs instead.