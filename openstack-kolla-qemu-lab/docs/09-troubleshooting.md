# 🔧 Troubleshooting — real failures and corrected lessons

[← Operations](08-operations-and-verification.md) · [Roadmap →](10-roadmap.md)

> Start with read-only evidence. Match the error before applying a fix. Historical failed approaches below are lessons, not steps to replay.

<details>
<summary><strong>1. Docker SDK / DBus import failed in prechecks</strong></summary>

**Symptoms:** `ModuleNotFoundError: docker` or `dbus`.

**Cause:** Ansible executed with the venv Python; system-only modules did not satisfy it.

**Fix:** install the venv dependencies from [installation](03-installation.md), then verify imports using that same interpreter. For DBus, install the OS development headers first.

**Lesson:** identify the interpreter from the failing task before installing packages.

</details>

<details>
<summary><strong>2. Public Kolla images required test acknowledgement</strong></summary>

**Symptom:** precheck explicitly required `--use-test-images` for `quay.io/openstack.kolla`.

**Fix:** pass that acknowledgement to prechecks for this test lab. Do not reinterpret it as a production image endorsement.

</details>

<details>
<summary><strong>3. Pull interrupted by SSH disconnect</strong></summary>

Inspect:

~~~bash
pgrep -af 'kolla-ansible|ansible-playbook'
docker image ls
tmux ls
~~~

Reattach to the existing session if running. Otherwise rerun the same pull in tmux; cached layers are reused. Do not delete images or launch duplicate playbooks as a first response.

</details>

<details>
<summary><strong>4. MariaDB could not bind port 3306</strong></summary>

**Evidence:** `Address already in use`; `ss` showed ProxySQL on the same host IP/port.

~~~bash
ss -lntp 'sport = :3306'
tail -n 60 /var/log/kolla/mariadb/mariadb.log
systemctl list-unit-files --type=service --no-pager |
  grep -Ei 'kolla.*(proxysql|haproxy|mariadb)'
~~~

This was a local bind conflict, not a missing security-group ingress rule.

For this no-proxy lab only:

~~~bash
systemctl disable --now \
  kolla-proxysql-container.service \
  kolla-haproxy-container.service
systemctl reset-failed kolla-mariadb-container.service
systemctl restart kolla-mariadb-container.service
~~~

Verify MariaDB healthy and `mariadbd` owns `3306`. Merely `docker stop` was not persistent: systemd enabled the proxies at boot despite Docker restart policy being `no`.

Do not delete database volumes, regenerate passwords or run cluster recovery based only on this bind error.

</details>

<details>
<summary><strong>5. MariaDB monitor healthcheck access denied</strong></summary>

**Symptom:** healthcheck denied `monitor@localhost`.

Read-only root diagnostics, using the **saved** database password without printing it:

~~~bash
/opt/kolla-venv/bin/python - <<'PY'
import os
import subprocess
import yaml
with open('/etc/kolla/passwords.yml') as f:
    password = yaml.safe_load(f)['database_password']
env = os.environ.copy()
env['MYSQL_PWD'] = password
query = (
    "SELECT User,Host,plugin FROM mysql.user "
    "WHERE User IN ('root','monitor'); "
    "SHOW GLOBAL STATUS WHERE Variable_name IN "
    "('wsrep_ready','wsrep_cluster_status','wsrep_connected');"
)
result = subprocess.run([
    'docker', 'exec', '-e', 'MYSQL_PWD', 'mariadb',
    'mariadb', '--protocol=socket', '-uroot', '-e', query
], env=env)
raise SystemExit(result.returncode)
PY
~~~

Observed: root authenticated using its saved password; cluster was Primary/connected/ready. No database reset was warranted.

The monitor account was initially absent; a later CREATE USER returned error 1396 while health had become healthy. That later attempt did not prove its following GRANT executed. Do not replay CREATE blindly, reset root, or overwrite credentials.

Inspect the installed Kolla role and actual account/health state before any targeted account repair. This repository intentionally does not ship an automatic SQL user-repair script.

</details>

<details>
<summary><strong>6. Dashboard worked, but guest console timed out</strong></summary>

HOST:

~~~bash
ss -lntp 'sport = :6080'
docker ps --filter name=nova_novncproxy
~~~

A local HTTP 200 from `http://YOUR_MANAGEMENT_IP:6080/vnc_lite.html` confirmed the console web service was reachable inside the host. The outer firewall/security group needed a narrowly scoped 6080 path for the administrator browser.

Guest internet is not needed for noVNC. Console access and guest networking are separate.

A dashboard tunnel on 8080 does not automatically tunnel a console URL pointing to another address/port.

</details>

<details>
<summary><strong>7. Cloud kolla-admin was not found</strong></summary>

A new root/tmux session lacked the config export.

~~~bash
export OS_CLIENT_CONFIG_FILE=/etc/kolla/clouds.yaml
export OS_CLOUD=kolla-admin
/usr/bin/openstack compute service list
~~~

Do not re-run post-deploy or print the full credential file merely to fix a missing environment variable.

</details>

<details>
<summary><strong>8. Gateway conflicts with allocation pool</strong></summary>

Historical subnet had pool `.1–.254` and no gateway. Adding gateway `.254` conflicted.

**Correct update for that existing lab subnet:**

~~~bash
/usr/bin/openstack subnet set lab-subnet \
  --no-allocation-pool \
  --allocation-pool start=192.168.50.1,end=192.168.50.253 \
  --gateway 192.168.50.254 \
  --dns-nameserver 1.1.1.1 \
  --dns-nameserver 8.8.8.8
~~~

The `--no-allocation-pool` flag is essential: without it the CLI added a second overlapping pool instead of replacing the first.

The observed `.1` was a DHCP port, not a free router gateway. An interface cannot be added to the router when the subnet has no gateway IP.

</details>

<details>
<summary><strong>9. ip netns exec: Operation not permitted</strong></summary>

Use a targeted elevated diagnostic process:

~~~bash
LAB_ROUTER_ID=$(/usr/bin/openstack router show lab-router -f value -c id)
docker exec --privileged -u 0 neutron_l3_agent \
  ip netns exec "qrouter-$LAB_ROUTER_ID" ip route
~~~

The original error meant namespace entry failed; ping had not actually executed. Do not call it a connectivity failure.

</details>

<details>
<summary><strong>10. ARP requests left enp8s0, but no reply returned</strong></summary>

Read-only namespace checks:

~~~bash
docker exec --privileged -u 0 neutron_l3_agent \
  ip netns exec "qrouter-$LAB_ROUTER_ID" ip -br address
docker exec --privileged -u 0 neutron_l3_agent \
  ip netns exec "qrouter-$LAB_ROUTER_ID" ip route
docker exec --privileged -u 0 neutron_l3_agent \
  ip netns exec "qrouter-$LAB_ROUTER_ID" ip neigh
~~~

An outer subnet gateway field did not mean a router interface existed. Attaching the missing outer interface restored ARP to `10.0.0.1`; ICMP replies still did not return.

Provider port security was confirmed disabled. The exact remaining provider return-path cause was **not proven**; possible reservation/MAC mapping behavior was a hypothesis.

The final working solution was **host NAT through br-ex**, not a claim that one broad security-group rule fixed nested routing.

</details>

<details>
<summary><strong>11. Changing the inner gateway MAC failed</strong></summary>

Two different restrictions were encountered:

1. OVS-bound port rejected MAC update.
2. After explicit API unbind, Neutron still rejected MAC changes for `device_owner network:router_gateway`.

The experiment restored the original host binding. The router MAC was **not** changed. The final design does not require forcing a gateway MAC to match the outer NIC.

Do not repeat unbind/MAC experiments or edit Neutron database records. Use the documented working host-gateway design.

</details>

<details>
<summary><strong>12. Host gateway .91 unreachable after adding IP</strong></summary>

Adding an address did not make the bridge's host interface administratively UP.

~~~bash
ip link set dev br-ex up
ip -br address show dev br-ex
~~~

Router-to-host ping then passed. `UNKNOWN` operational state can be normal for OVS internal interfaces.

</details>

<details>
<summary><strong>13. APIs returned 500 after the first reboot</strong></summary>

Startup ProxySQL occupied MariaDB's port and denied service DB connections. Neutron logs contained an old startup authentication error. Nova conductor could not discover Placement versions because Placement was failing, not because a version mismatch had been established.

After MariaDB was healthy and obsolete proxies were disabled, targeted recovery used:

~~~bash
systemctl restart \
  kolla-placement_api-container.service \
  kolla-neutron_server-container.service \
  kolla-nova_api-container.service \
  kolla-nova_metadata-container.service
~~~

After Placement returned a healthy discovery response, Nova conductor/scheduler were restarted:

~~~bash
systemctl restart \
  kolla-nova_conductor-container.service \
  kolla-nova_scheduler-container.service
~~~

These are **service-impacting recovery commands**, not a routine boot script. Do not add repeated blanket restarts as a substitute for diagnosis.

A subsequent clean reboot recovered core services without these manual restarts.

</details>

<details>
<summary><strong>14. Host ping passed, but guest connectivity was not verified</strong></summary>

Check `hostname`. `horizon` = HOST; `lab-vm01` = GUEST.

Run final IP/DNS tests in the browser guest console. Do not use host results as evidence of guest routing, tenant isolation or firewall behavior.

</details>

<details>
<summary><strong>15. Guest was ACTIVE briefly, then SHUTOFF after reboot</strong></summary>

Wait for nova-compute heartbeat and power-state synchronization. The observed guest settled to SHUTOFF and was started manually. Automatic guest startup is not configured.

Do not reset the API state to ACTIVE while the guest is physically powered off.

</details>
