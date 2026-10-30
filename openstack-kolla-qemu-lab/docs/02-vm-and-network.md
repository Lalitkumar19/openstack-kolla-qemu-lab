# 🖥️ VM and networking

[← Architecture](01-architecture.md) · [Next →](03-installation.md)

> All HOST commands target a dedicated Ubuntu lab VM, never an existing provider controller.

## 1. Outer VM

Create Ubuntu 24.04 x86_64 with two NICs, adequate storage/RAM and sudo/SSH-key access. One NIC provides management/internet; the other is a private/provider link. The observed 64-vCPU lab is not a sizing minimum.

## 2. 🔎 Inspect

~~~bash
cat /etc/os-release
hostname -f
lscpu
free -h
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
df -h /
ip -br link
ip -br address
ip route
~~~

| Check | Why |
|---|---|
| CPU/RAM | Capacity and architecture |
| `lsblk` | Disk/partition identities |
| `df` | Actual usable filesystem space |
| Interface MACs | Map portal NICs to Linux names |
| Routes | Preserve management/default connectivity |

Do not assume every image uses the same NIC names.

## 3. 🔎 QEMU decision

~~~bash
grep -Ew 'svm|vmx' /proc/cpuinfo | head -n 1
ls -l /dev/kvm
~~~

No flags and no device were observed. We selected QEMU emulation. We did not modify an outer compute host. Large resource allocations do not replace hardware acceleration.

## 4. Final interface layout

| NIC | Host configuration |
|---|---|
| `enp3s0` | Existing management DHCP/IP/default route |
| `enp8s0` | No direct host IP/DHCP; later OVS bridge member |
| `br-ex` | Kolla-created OVS bridge; later host NAT service adds `.91` |

An early wildcard match `name: e*` enabled DHCP on both NICs and added unwanted defaults.

## 5. 🛠️ Back up before network edits

Have cloud console access and a second SSH session available.

~~~bash
install -d -m 700 /root/kolla-lab/network-backup
cp -a /etc/netplan /root/kolla-lab/network-backup/
~~~

If `/etc/cloud/cloud.cfg.d/99-network-config.cfg` exists, back it up before replacing it. Inspect other cloud-init network overrides too.

~~~bash
cat > /etc/cloud/cloud.cfg.d/99-network-config.cfg <<'EOF'
network:
  config: disabled
EOF
~~~

**Why:** prevent cloud-init from regenerating the lab's manually managed network configuration.

## 6. 🛠️ Explicit Netplan

Inspect existing definitions:

~~~bash
netplan get
ls -l /etc/netplan/
~~~

Intended `/etc/netplan/50-cloud-init.yaml` for this lab:

~~~yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:
      dhcp4: true
    enp8s0:
      dhcp4: false
      dhcp6: false
      accept-ra: false
      link-local: []
      optional: true
~~~

Edit/merge the existing configuration; do not add conflicting definitions across files.

~~~bash
chmod 600 /etc/netplan/50-cloud-init.yaml
netplan generate
netplan try --timeout 120
~~~

`generate` validates/renders. `try` applies with confirmation/timeout. Confirm only after checking SSH and routes. The later host NAT service manages the `br-ex` address separately.

## 7. 🔎 Validate

~~~bash
ip -br address
ip route
getent ahostsv4 quay.io
curl -I --connect-timeout 10 --max-time 30 https://quay.io/v2/
timedatectl status
getent ahostsv4 horizon
~~~

Expected: one intended default route, working registry DNS/TLS, synchronized clock and correct hostname resolution. Registry `401` can be the expected authentication challenge, not an internet failure.
