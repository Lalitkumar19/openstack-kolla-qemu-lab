# ✅ Operations, evidence and reboot verification

[← Persistence](07-reboot-persistence.md) · [Next: Troubleshooting →](09-troubleshooting.md)

## 1. 🖥️ HOST: inspect containers

~~~bash
docker ps -a --format 'table {{.Names}}\t{{.Status}}'
~~~

Most core containers expose health checks; some helpers do not. A container without a “healthy” label is not automatically faulty. Compare state, logs and API behavior.

The final clean reboot showed all core health-checked containers healthy after approximately two minutes. Checks at 16 seconds were too early.

Top-level `proxysql` and `haproxy` are deliberately stopped in this lab. These are separate from namespace-specific `neutron_*_haproxy_*` helper containers, which must remain operational.

## 2. 🔎 Check actual OpenStack heartbeats

~~~bash
export OS_CLIENT_CONFIG_FILE=/etc/kolla/clouds.yaml
export OS_CLOUD=kolla-admin
/usr/bin/openstack compute service list
/usr/bin/openstack network agent list
~~~

| Service | Expected |
|---|---|
| Nova scheduler/conductor/compute | enabled, up |
| Neutron metadata/OVS/DHCP/L3 | alive, UP |

A Docker health check and an OpenStack heartbeat answer different questions.

## 3. 🔎 Check host NAT state

~~~bash
systemctl is-active kolla-lab-host-nat.service
systemctl is-enabled kolla-proxysql-container.service kolla-haproxy-container.service
ip -br address show dev br-ex
sysctl net.ipv4.ip_forward
iptables -t nat -S POSTROUTING
~~~

Expected: NAT active; top-level proxy units disabled; bridge `10.0.0.91/24`; forwarding 1; exact MASQUERADE rule.

`systemctl is-enabled` returns a nonzero status for disabled units; the printed `disabled` is the intended result here.

## 4. 🔎 Guest API state

~~~bash
/usr/bin/openstack server show lab-vm01 \
  -c status -c OS-EXT-STS:power_state \
  -c OS-EXT-STS:task_state -c addresses
~~~

Immediately after reboot an API can briefly show stale ACTIVE metadata while actual power state is Shutdown. Wait for compute heartbeat/state synchronization before acting.

Observed final state settled to **SHUTOFF**. Start it deliberately:

~~~bash
/usr/bin/openstack server start lab-vm01
~~~

This changes the guest power state. Wait and verify ACTIVE / Running with task state clear. Do not use a state-reset command to pretend a powered-off guest is running.

## 5. 🐧 Verify from INSIDE the guest

Open the browser console and log in as the image's test user. Run:

~~~sh
hostname
ip route
cat /etc/resolv.conf
ping -c 3 1.1.1.1
ping -c 3 example.com
~~~

Expected hostname `lab-vm01`, gateway `192.168.50.254`, intended DNS and successful connectivity.

**Evidence still pending:** repeat this inside the guest after the final clean reboot. The last pasted successful pings used the HOST `root@horizon` shell. Do not mark guest persistence fully verified from host pings.

## 6. Final reboot checklist

- [x] NAT unit enabled and successful after reboot.
- [x] Bridge IP and forwarding retained.
- [x] Obsolete proxy boot services stayed disabled.
- [x] MariaDB and core services recovered without manual restarts after proxy fix.
- [x] Nova services and Neutron agents UP.
- [x] Guest manually started and API reported ACTIVE / Running.
- [ ] Guest internet + DNS repeated inside guest after that final reboot.
- [ ] Admin credentials rotated and access hardened.

## 7. Useful read-only diagnostics

~~~bash
ss -lntp 'sport = :3306'
ss -lntp 'sport = :6080'
systemctl --failed --no-pager
journalctl -u kolla-lab-host-nat.service -n 30 --no-pager
~~~

Use targeted container health output:

~~~bash
docker inspect --format \
  '{{range .State.Health.Log}}{{println .End .ExitCode .Output}}{{end}}' mariadb
~~~

Keep health/log output private if it contains sensitive details.

## 8. Capture evidence, not secrets

For GitHub, record the outcome and a short sanitized screenshot. Omit public addresses, console tokens, account IDs and passwords.

A useful milestone update:

> Clean host reboot: core APIs and service heartbeats recovered automatically. Guest power state was SHUTOFF; manual start succeeded. Guest connectivity recheck is recorded separately.

## 9. Planned automatic guest startup

Automatic guest recovery/startup after host reboot is not configured. Treat it as a future Nova configuration change with its own tests, not something implied by `ACTIVE` before the reboot.
