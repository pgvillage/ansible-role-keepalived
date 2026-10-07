# pgvillage.keepalived API

This document describes all variables that can be used to configure the
`pgvillage.keepalived` role. Defaults are defined in
[`defaults/main.yml`](../defaults/main.yml).

## Installation

| Variable | Default | Description |
|----------|---------|-------------|
| `keepalived_package_state` | `present` | State passed to the package module for all keepalived packages (e.g. `present`, `latest`, `absent`). |
| `keepalived_local_packages` | `[]` | List of local package files that are copied to `/tmp/` on the target and installed from there. |
| `keepalived_packages` | `[keepalived]` | List of packages to install from the configured package repositories. |

## Virtual IP / VRRP instance

These variables are rendered into `/etc/keepalived/keepalived.conf` as a single
VRRP instance named `balancer`.

| Variable | Default | Description |
|----------|---------|-------------|
| `keepalived_cluster_password` | first 8 chars of `md5(keepalived_vip_definition)` | Password used by VRRP peers to authenticate each other (`auth_type PASS`). Nodes derive the same password only when the full VIP definition (including interface, CIDR and label) is identical; otherwise set a shared password explicitly. |
| `keepalived_interface` | `eth0` | Network interface VRRP runs on and on which the VIP is configured. |
| `keepalived_vip` | `192.168.0.100` | The virtual IP address managed by keepalived. |
| `keepalived_vip_cidr` | `24` | Prefix length (CIDR) of the virtual IP address. |
| `keepalived_label` | `vip` | Label suffix for the VIP; the address is labeled `<interface>:<label>`. |
| `keepalived_vip_definition` | `{{ keepalived_vip }}/{{ keepalived_vip_cidr }} dev {{ keepalived_interface }} label {{ keepalived_interface }}:{{ keepalived_label }}` | Full `virtual_ipaddress` entry written to keepalived.conf. Override only if you need a custom definition. |
| `keepalived_virtual_cluster_name` | `keepalived` | Name of the virtual cluster; used to derive the virtual router id. |
| `keepalived_virtual_router_id` | `{{ keepalived_virtual_cluster_name \| hashnum(256) }}` | VRRP `virtual_router_id` (1-255). Must be identical on all nodes of a cluster and unique per network segment. The default hash can produce `0`; override it if this occurs. |

Notes on the generated config:

- Hosts whose FQDN contains `1` start as `MASTER` with priority `100`; all
  others start as `BACKUP` with priority `50`.
- A `check_haproxy` track script (`killall -0 haproxy`, weight `2`) is always
  configured.

## Kernel (sysctl) tuning

The individual values below are combined in `keepalived_sysctl_config`, which
is applied (and persisted) with `ansible.posix.sysctl`.

| Variable | Default | sysctl key | Description |
|----------|---------|------------|-------------|
| `keepalived_net_core_rmem_max` | `16777216` | `net.core.rmem_max` | Maximum socket receive buffer size (bytes). |
| `keepalived_net_ipv4_tcp_rmem` | `4096 87380 16777216` | `net.ipv4.tcp_rmem` | Min, default and max TCP receive buffer size (bytes). |
| `keepalived_net_core_wmem_max` | `16777216` | `net.core.wmem_max` | Maximum socket send buffer size (bytes). |
| `keepalived_net_ipv4_tcp_wmem` | `4096 16384 16777216` | `net.ipv4.tcp_wmem` | Min, default and max TCP send buffer size (bytes). |
| `keepalived_net_ipv4_tcp_fin_timeout` | `20` | `net.ipv4.tcp_fin_timeout` | Seconds to keep sockets in FIN-WAIT-2 state. |
| `keepalived_net_ipv4_tcp_tw_reuse` | `1` | `net.ipv4.tcp_tw_reuse` | Allow reuse of TIME-WAIT sockets for new connections. |
| `keepalived_net_core_netdev_max_backlog` | `10000` | `net.core.netdev_max_backlog` | Max packets queued on the input side. |
| `keepalived_net_ipv4_ip_local_port_range` | `15000 65001` | `net.ipv4.ip_local_port_range` | Range of local ports used for outgoing connections. |
| `keepalived_net_ipv4_ip_nonlocal_bind` | `1` | `net.ipv4.ip_nonlocal_bind` | Allow services to bind to the VIP while it is not configured on this node. |
| `keepalived_net_ipv4_ip_forward` | `1` | `net.ipv4.ip_forward` | Enable IPv4 packet forwarding. |
| `keepalived_net_ipv4_conf_all_forwarding` | `1` | `net.ipv4.conf.all.forwarding` | Enable forwarding on all interfaces. |

### `keepalived_sysctl_config`

List of `key`/`value` pairs applied as sysctl settings. By default it contains
all of the settings above. Override it to add, remove or change kernel
parameters:

```yaml
keepalived_sysctl_config:
  - key: net.ipv4.ip_nonlocal_bind
    value: "1"
```

## VRRP instances (validated only)

### `keepalived_vrrp_instances`

Default: `[]`

List of VRRP instance definitions. The role validates this list (see
[`tasks/assert.yml`](../tasks/assert.yml) and
[`tasks/assert_instances.yml`](../tasks/assert_instances.yml)), but the current
`keepalived.conf.j2` template does **not** render it; the generated config uses
the `keepalived_vip*` / `keepalived_interface` variables instead.

The descriptions below express intended behavior; these settings have no effect
on the generated config. The unicast and command assertions currently reference
`item` instead of `instance`, so their requirements are not reliably enforced.

| Key | Required | Description |
|-----|----------|-------------|
| `name` | yes | Name of the VRRP instance (string). |
| `state` | yes | Initial state: `MASTER` or `BACKUP`. |
| `interface` | yes | Interface VRRP runs on. |
| `virtual_router_id` | yes | Unique identifier, `1`-`255` for Keepalived; validation currently also accepts `0`. |
| `priority` | yes | Advertised priority, `1`-`255` (max `252` when `check_status_command` is set). |
| `unicast_src_ip` | no | Primary address used for unicast. |
| `secondary_private_ip` | with `unicast_src_ip` | The peer's unicast address. |
| `check_status_command` | no | Command that adds `+3` to the priority if it returns `0`. |
| `authentication.auth_type` | yes | `AH` or `PASS`. |
| `authentication.auth_pass` | yes | Validation accepts 1-20 characters; Keepalived uses only the first 8. |
| `virtual_ipaddresses` | yes | List of `{name, cidr}` entries; `cidr` must be `0`-`32`. |

You do not need to set the state to `MASTER`; all nodes can be `BACKUP`, in
which case one host will be elected to configure the virtual IP. `MASTER` only
sets the initial state; over time other nodes may become master.

Example:

```yaml
keepalived_vrrp_instances:
  - name: VI_1
    state: MASTER
    interface: ens192
    unicast_src_ip: "192.168.1.1"
    secondary_private_ip: "192.168.1.2"
    virtual_router_id: 51
    priority: 250
    check_status_command: /sbin/postfix status
    authentication:
      auth_type: PASS
      auth_pass: "12345"
    virtual_ipaddresses:
      - name: "192.168.122.200"
        cidr: 24
```

## Example playbook

```yaml
- hosts: pgvillage
  become: true
  vars:
    keepalived_interface: ens192
    keepalived_vip: 10.0.0.50
    keepalived_vip_cidr: 24
    keepalived_virtual_cluster_name: pgcluster1
  roles:
    - pgvillage.keepalived
```
