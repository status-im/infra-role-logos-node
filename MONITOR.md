# Container Monitor

Container [monitoring script](templates/monitor.sh.j2) is present as a service to update Consul metadata with current state of a service.

It listens to Docker events for given container and updates the JSON service definition in `/etc/consul`.

```
$ systemctl -o cat -n5 status monitor-logos-node
● monitor-logos-node.service - Container state monitor for logos-node
     Loaded: loaded (/etc/systemd/system/monitor-logos-node.service; enabled)
     Active: active (running)
```

# Known Issues

The method of modifying JSON files in `/etc/consul` using `sed` is crude but simple. It avoids the need to have to call [Consul Catalog API](https://developer.hashicorp.com/consul/api-docs/catalog#register-entity) with the whole set of service metadata only to update a single `ServiceMeta` key.

But there are a few problems with this approach:

* It only works for metadata fields that are already present in the JSON file.
* It causes Ansible to overwrite the fields, setting them to `unknown`.
* It can fail due to JSON file format changes.