# 💾 Persist host NAT and bridge addressing

[← Network design](06-router-and-host-nat.md) · [Next →](08-operations-and-verification.md)

> 🛠️ This writes host config and installs a systemd service. It does not deploy Kolla or create guest resources.

## 1. Files

| Repository source | Host destination | Purpose |
|---|---|---|
| [NAT script](../scripts/kolla-lab-host-nat) | `/usr/local/sbin/kolla-lab-host-nat` | Bridge IP, forwarding and exact NAT rules |
| [Systemd unit](../systemd/kolla-lab-host-nat.service) | `/etc/systemd/system/kolla-lab-host-nat.service` | Apply after Docker; retry while bridge unavailable |
| [Sysctl config](../configs/99-kolla-lab-forwarding.conf) | `/etc/sysctl.d/99-kolla-lab-forwarding.conf` | Persist IPv4 forwarding |

**Read all three files before installing.** Interfaces and addresses are this lab's values.

## 2. Make repository sources available on the HOST

The initial GitHub upload is from the browser. To install the script later, copy the extracted sources to the host, or clone your published repository:

~~~bash
git clone https://github.com/YOUR_USERNAME/openstack-kolla-qemu-lab.git \
  /root/openstack-kolla-qemu-lab
~~~

Replace `YOUR_USERNAME`. Do not clone over an existing working directory. A private repo needs your normal authorized GitHub access; do not put tokens in a command or commit them.

## 3. Install the reviewed files

~~~bash
REPO_DIR=/root/openstack-kolla-qemu-lab
install -m 750 "$REPO_DIR/scripts/kolla-lab-host-nat" \
  /usr/local/sbin/kolla-lab-host-nat
install -m 644 "$REPO_DIR/systemd/kolla-lab-host-nat.service" \
  /etc/systemd/system/kolla-lab-host-nat.service
install -m 644 "$REPO_DIR/configs/99-kolla-lab-forwarding.conf" \
  /etc/sysctl.d/99-kolla-lab-forwarding.conf
bash -n /usr/local/sbin/kolla-lab-host-nat
systemctl daemon-reload
systemctl enable --now kolla-lab-host-nat.service
~~~

Do not overwrite existing customized copies without backing them up/reviewing differences.

## 4. How the script works

1. Require `br-ex` to exist; fail if OVS is not ready.
2. Set the bridge interface UP.
3. Apply `10.0.0.91/24` to the host bridge interface.
4. Enable IPv4 forwarding.
5. Check each exact iptables rule before inserting it.
6. Preserve unrelated firewall chains/policies.

It does not flush rules, move the public NIC or configure a new default route.

## 5. How systemd handles startup

- `After=docker.service` orders startup after Docker.
- Docker ready does not necessarily mean OVS bridge ready.
- `Restart=on-failure` retries after 10 seconds if `br-ex` is not available yet.
- `Type=oneshot` with `RemainAfterExit=yes` means **active (exited)** is success.
- `PartOf=docker.service` propagates a Docker restart to the unit.
- This is a startup/restart mechanism, **not continuous monitoring** of arbitrary later firewall/OVS changes.

## 6. 🔎 Validate

~~~bash
systemctl status kolla-lab-host-nat.service --no-pager
ip -br address show dev br-ex
sysctl net.ipv4.ip_forward
iptables -t nat -S POSTROUTING
~~~

Expected: enabled/active (exited), exit status 0, `.91/24` on bridge, forwarding 1, exact MASQUERADE rule.

One initial retry was observed on boot before the bridge existed, followed by success.

## 7. Controlled reboot

Only after base services, proxy configuration and current guest connectivity are verified:

~~~bash
reboot
~~~

This interrupts the lab. Reconnect, wait for health checks, then use the next chapter's checklist. Do not reboot a provider controller or unrelated production VM.

## 8. Removing this customization

Disabling the unit prevents future application but does not automatically remove existing runtime IP/rules. A complete rollback must remove the exact three custom rules and bridge address and restore the inner router gateway deliberately. Do not flush firewall tables. This guide does not provide a blanket rollback for another environment.
