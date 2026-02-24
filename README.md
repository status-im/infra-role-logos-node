# Description

This role provisions a [Logos Node](https://github.com/logos-co/logos-liblogos) running `logoscore` with the [logos-waku-module](https://github.com/logos-co/logos-waku-module). It replaces [infra-role-nim-waku](https://github.com/status-im/infra-role-nim-waku) for fleets using the new modular Logos architecture.

The Logos Node loads `libwaku` (from [logos-delivery](https://github.com/logos-messaging/logos-delivery)) through a plugin wrapper, providing the same Waku protocols (relay, store, filter, lightpush, mix) via a modular runtime.

# Configuration

The crucial settings are:
```yaml
# Private key which affects peer ID
logos_node_key: '{{ lookup("vault", "delivery/nodekeys", field=hostname) }}'
logos_node_log_level: 'DEBUG'
```
Modules and their init commands are configured via:
```yaml
logos_node_modules: ['waku_module']
logos_node_module_commands:
  - 'waku_module.initWaku(@/conf/waku_config.json)'
  - 'waku_module.startWaku()'
```
Protocols are disabled by default and enabled per-fleet in `group_vars`:
```yaml
logos_node_store: true
logos_node_filter: true
logos_node_lightpush: true
logos_node_mix: true
logos_node_mix_key: '{{ lookup("vault", "delivery/mixkeys", field=hostname) }}'
logos_node_discv5: true
logos_node_kad_discovery: true
```
Store protocol requires a PostgreSQL connection:
```yaml
logos_node_store_db_url: 'postgres://delivery:PASSWORD@delivery-db-01.example.com:5432/delivery'
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

There's also a [container monitor service](./MONITOR.md).
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

* `libwaku` does not expose a REST API or metrics endpoint — these are `wakunode2` app features. Peer info (peer ID, ENR, multiaddr) is extracted from container logs instead.
