# 🗺️ Architecture and packet flow

[← Home](../README.md) · [Next →](02-vm-and-network.md)

## 1. Layers

~~~mermaid
flowchart TD
    O["Outer OpenStack cloud"] --> H["Ubuntu VM: horizon"]
    H --> D["Docker: OpenStack services"]
    H --> Q["Libvirt / QEMU guest"]
    D -->|"Nova provisions"| Q
~~~

- **Outer cloud:** provides the original host VM, NICs and disks.
- **Lab host:** Ubuntu VM running the Kolla deployment.
- **Inner cloud:** the newly deployed OpenStack APIs/dashboard.
- **Guest:** VM created through the inner cloud; not a Docker container.

## 2. Components

| Component | Purpose |
|---|---|
| Kolla Ansible | Deploy/configure OpenStack |
| Docker | Run service containers |
| Horizon | Browser dashboard |
| Keystone | Authentication and scoped authorization |
| Nova | Scheduling and instance lifecycle |
| Placement | Resource inventory/allocations used by Nova |
| Glance | Image catalog and data |
| Neutron + OVS | Virtual networks, DHCP, routing and switching |
| MariaDB | Service databases |
| RabbitMQ | RPC/message transport |
| Libvirt / QEMU | Guest management and software emulation |
| noVNC | Browser console |
| Cinder | Persistent block storage — not deployed yet |

## 3. Management flow

~~~mermaid
flowchart TD
    B["Browser / CLI"] --> K["Keystone authentication"]
    B --> A["Horizon / OpenStack APIs"]
    K -. "Scoped identity + policy" .-> A
    A --> N["Nova + Placement"]
    N --> R["Neutron + Glance"]
    N --> G["Libvirt / QEMU guest"]
~~~

Keystone authenticates requests; it is not a guest packet-forwarding hop. API success does not alone prove guest boot/network health.

## 4. Actual working outbound flow

~~~mermaid
flowchart TD
    V["Guest 192.168.50.65"] --> RI["Inner gateway 192.168.50.254"]
    RI --> RE["Router SNAT 10.0.0.20"]
    RE --> HG["Host br-ex gateway 10.0.0.91"]
    HG --> U["Host MASQUERADE via enp3s0"]
    U --> I["Internet"]
~~~

1. Guest sends off-subnet packets to `192.168.50.254`.
2. Inner router SNAT translates source to `10.0.0.20`.
3. Router's external default gateway is **host `10.0.0.91`**.
4. Host forwards from `br-ex` to public uplink `enp3s0`.
5. Host MASQUERADE translates source to its uplink address.
6. Established NAT state handles replies.

This is a **double-NAT lab**. Outbound internet does not imply public inbound SSH/RDP access per guest.

## 5. Bridge membership is not routing

| Relationship | Meaning |
|---|---|
| `enp8s0 ↔ br-ex` | OVS provider bridge membership |
| `10.0.0.91/24` on `br-ex` | Host Layer 3 gateway |
| `br-ex → host IP stack → enp3s0` | Host routing + NAT |
| `enp3s0` | Public/management NIC; **not a br-ex member** |

Do not move the public NIC into `br-ex`. Do not put `.91` on both `enp8s0` and `br-ex`.

## 6. Address plan

| Address | Purpose |
|---|---|
| `192.0.2.10` | Sanitized management example; replace locally |
| `192.168.50.65` | Observed guest lease; a new lab may differ |
| `192.168.50.1` | Observed DHCP service |
| `192.168.50.254` | Inner router |
| `10.0.0.20` | Inner router external/SNAT IP |
| `10.0.0.91` | Host gateway, allocated to original outer private port |
| `10.0.0.1` | Earlier outer gateway; **not the final inner external gateway** |

The outer reservation for `.20` prevents duplicate allocation; it is not an attached router. No separate API VIP was created.

## 7. MTU

Outer provider MTU = **1442**. Neutron `global_physnet_mtu` and `path_mtu` were set to 1442. Observed overlay guest MTU = **1392**, accounting for tunnel overhead. Management NIC retained its original MTU.

## 8. Reboot

Core services recover through Kolla systemd units. A custom unit restores host bridge/NAT. Obsolete top-level ProxySQL/HAProxy units are disabled. **Guest startup remains manual** after host reboot.
