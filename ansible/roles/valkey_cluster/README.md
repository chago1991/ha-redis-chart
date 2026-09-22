# valkey_cluster role

This role converts the main runtime ideas from the `redis-ha` chart into an on-prem Ansible deployment for Valkey, Sentinel, and optional HAProxy.

## What it does

- installs Valkey on cluster nodes
- configures one bootstrap primary and the remaining nodes as replicas
- configures Sentinel on each Valkey node for automatic failover
- optionally installs HAProxy either on the same hosts or on dedicated proxy hosts
- keeps the HAProxy master and read-only routing behavior aligned with the chart

## Inventory layouts

### 1. Colocated HAProxy

Apply the role to the same hosts that run Valkey and enable `valkey_haproxy_enabled`.

```yaml
all:
  children:
    valkey:
      hosts:
        valkey-1:
        valkey-2:
        valkey-3:
```

```yaml
- hosts: valkey
  become: true
  roles:
    - role: valkey_cluster
      vars:
        valkey_haproxy_enabled: true
```

### 2. Dedicated HAProxy hosts

Run the role once for the Valkey nodes and again for the HAProxy nodes.

```yaml
all:
  children:
    valkey:
      hosts:
        valkey-1:
        valkey-2:
        valkey-3:
    haproxy:
      hosts:
        proxy-1:
        proxy-2:
```

```yaml
- hosts: valkey
  become: true
  roles:
    - role: valkey_cluster

- hosts: haproxy
  become: true
  roles:
    - role: valkey_cluster
      vars:
        valkey_server_enabled: false
        valkey_sentinel_enabled: false
        valkey_haproxy_enabled: true
        valkey_haproxy_backend_nodes: "{{ groups['valkey'] }}"
```

## Important variables

| Variable | Purpose | Default |
| --- | --- | --- |
| `valkey_cluster_nodes` | Inventory hosts that run Valkey and Sentinel | `groups['valkey']` |
| `valkey_primary_host` | Bootstrap primary used for the initial replica configuration | first cluster host |
| `valkey_requirepass` | Password applied to Valkey and replica auth | `""` |
| `valkey_auth_mode` | Supported authentication mode for this role | `none` or `shared_password` |
| `valkey_sentinel_password` | Reserved for future peer-auth support; keep empty with the current role | `""` |
| `valkey_sentinel_binary` | Sentinel executable when it differs from the server binary | `valkey_server_binary` |
| `valkey_sentinel_overwrite_config` | Re-template `sentinel.conf` even after Sentinel has persisted runtime state | `false` |
| `valkey_haproxy_enabled` | Installs and configures HAProxy on the current host | `false` |
| `valkey_haproxy_allow_insecure_auth_transport` | Explicitly allow HAProxy backend checks to use password auth without TLS | `false` |
| `valkey_haproxy_readonly_enabled` | Exposes a second HAProxy port for replicas | `false` |
| `valkey_haproxy_backend_nodes` | Backend nodes used by HAProxy | `valkey_cluster_nodes` |
| `valkey_disable_commands` | Commands disabled with `rename-command`; keep `INFO` enabled when using HAProxy health checks | `[FLUSHDB, FLUSHALL]` |
| `valkey_server_packages` | Override package names when your distro differs | `[valkey-server]` |

## Notes

- The role assumes Valkey is available from the target host package repositories.
- Override `valkey_server_packages`, `valkey_server_binary`, or the user/group variables if your distribution uses different names.
- Sentinel rewrites its own config during failover, so the sentinel config file is owned by the Valkey service account and is preserved on later role runs unless `valkey_sentinel_overwrite_config` is set to `true`.
- The supported auth model is a simple shared password (`requirepass`/`masterauth`). ACL-style username-based auth is out of scope for this role.
- Sentinel peer authentication is intentionally not enabled by this role yet; leave `valkey_sentinel_password` empty so Sentinel quorum traffic remains in the supported mode.
- If you enable HAProxy together with `valkey_requirepass`, the role requires `valkey_haproxy_allow_insecure_auth_transport: true` because HAProxy backend health checks use cleartext TCP unless you add your own secured transport layer.
