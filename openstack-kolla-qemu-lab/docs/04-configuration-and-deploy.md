# ⚙️ Configure and deploy

[← Installation](03-installation.md) · [Next →](05-first-instance.md)

## 1. 🛠️ Set globals

Edit `/etc/kolla/globals.yml` using the [sanitized example](../configs/globals.yml.example).

| Setting | Purpose |
|---|---|
| `kolla_base_distro` | Ubuntu container base |
| `openstack_release` | Match 2026.1 tooling/images |
| `network_interface / api_interface` | Management NIC |
| `neutron_external_interface` | Provider NIC handed to OVS |
| `kolla_internal_vip_address` | Host management IP for this no-proxy lab |
| `enable_haproxy / keepalived / proxysql` | All explicitly false |
| `nova_compute_virt_type` | QEMU software emulation |
| `enable_horizon` | Dashboard |

**Replace `192.0.2.10` with your actual management IP.** It is not a spare address or an additional VIP to allocate.

This host-IP/no-load-balancer design is a lab workaround. It has no HA and should not be generalized to production.

## 2. 🛠️ Separate YAML and INI overrides

These are **three different files**. Do not paste `[DEFAULT]` / `[ml2]` into `globals.yml`.

~~~bash
install -d -m 700 /etc/kolla/config/neutron
cat > /etc/kolla/config/neutron.conf <<'EOF'
[DEFAULT]
global_physnet_mtu = 1442
EOF
cat > /etc/kolla/config/neutron/ml2_conf.ini <<'EOF'
[ml2]
path_mtu = 1442
EOF
~~~

Why: conservative physical/path MTU for the observed private link. Neutron accounts for overlay overhead.

## 3. 🔎 Validate private configuration without printing secrets

~~~bash
/opt/kolla-venv/bin/python - <<'PY'
import yaml
with open('/etc/kolla/globals.yml') as f:
    cfg = yaml.safe_load(f)
assert isinstance(cfg, dict)
for key in ('enable_haproxy', 'enable_keepalived', 'enable_proxysql'):
    assert cfg.get(key) is False, key
print('Globals parsed; lab proxies disabled.')
PY
~~~

## 4. 🛠️ Bootstrap host services

Inside the venv/tmux session:

~~~bash
source /opt/kolla-venv/bin/activate
kolla-ansible bootstrap-servers \
  --configdir /etc/kolla \
  -i /root/kolla-lab/all-in-one \
  -e @/etc/kolla/globals.yml
~~~

Bootstrap prepares the host, including its container engine. Observed checks:

~~~bash
docker --version
systemctl is-active docker
~~~

Expected: Docker active. The session used Docker 29.9.2; this is an observation, not a package pin.

## 5. 🔎 Prechecks

~~~bash
kolla-ansible prechecks \
  --configdir /etc/kolla \
  --passwords /etc/kolla/passwords.yml \
  -i /root/kolla-lab/all-in-one \
  -e @/etc/kolla/globals.yml \
  --use-test-images
~~~

The observed public `quay.io/openstack.kolla` namespace required the explicit test-image acknowledgement. It is appropriate to this learning lab, not a statement of production image suitability.

Expected: `failed=0` and `unreachable=0`.

## 6. 🛠️ Pull images

~~~bash
kolla-ansible pull \
  --configdir /etc/kolla \
  -i /root/kolla-lab/all-in-one \
  -e @/etc/kolla/globals.yml
~~~

A download interrupted by an SSH disconnect does not require deleting images.

~~~bash
pgrep -af 'kolla-ansible|ansible-playbook'
docker image ls --format 'table {{.Repository}}\t{{.Tag}}\t{{.Size}}'
~~~

If no task is still running, rerun the same pull in tmux. Docker reuses already-present layers. Only a successful recap establishes the pull phase completed.

## 7. 🛠️ Deploy with explicit configuration

~~~bash
kolla-ansible deploy \
  --configdir /etc/kolla \
  --passwords /etc/kolla/passwords.yml \
  -i /root/kolla-lab/all-in-one \
  -e @/etc/kolla/globals.yml \
  -e '{"enable_haproxy": false, "enable_keepalived": false, "enable_proxysql": false}'
~~~

Why explicit flags? The session saw obsolete proxies remain/start despite saved false settings. Passing globals as extra vars and explicitly disabling proxies made the intended lab configuration authoritative. The exact earlier configuration-loading cause was not established.

A successful deployment is **not enough** if stale proxy boot units still exist. See below before reboot.

Disabled optional service plays can show “no hosts matched”; inspect the final recap and actual enabled services.

## 8. 🛠️ Generate client configuration

~~~bash
kolla-ansible post-deploy \
  --configdir /etc/kolla \
  --passwords /etc/kolla/passwords.yml \
  -i /root/kolla-lab/all-in-one \
  -e @/etc/kolla/globals.yml \
  -e '{"enable_haproxy": false, "enable_keepalived": false, "enable_proxysql": false}'
~~~

Observed generated client file: `/etc/kolla/clouds.yaml`, including `kolla-admin`. Keep this private.

Install/use the system client:

~~~bash
apt install -y python3-openstackclient
export OS_CLIENT_CONFIG_FILE=/etc/kolla/clouds.yaml
export OS_CLOUD=kolla-admin
/usr/bin/openstack compute service list
/usr/bin/openstack network agent list
~~~

A new login/tmux shell may need these exports again. The session used system OpenStackClient 6.6.0; a future toolchain should be reviewed against the release rather than assumed identical.

## 9. 🔎 Dashboard and console

Using your real management address locally:

~~~bash
MANAGEMENT_IP=192.0.2.10
curl -I --connect-timeout 10 --max-time 30 "http://$MANAGEMENT_IP/"
~~~

**Replace the example before running.** A `302` to `/auth/login/` is the expected dashboard redirect.

Prefer an SSH tunnel from your **LAPTOP**, using the SSH key that actually works:

~~~bash
ssh -i /path/to/private-key -N \
  -L 8080:192.0.2.10:80 ubuntu@192.0.2.10
~~~

Replace both example addresses and key path. Open `http://localhost:8080`. The tunnel must run on the laptop, not by SSHing from the VM into itself.

noVNC console URLs may still point to port 6080 on the management IP. A dashboard-only tunnel does not automatically tunnel that separate console endpoint. Use a properly configured console tunnel or narrowly scoped port-6080 ingress; do not expose broad admin access.

Login credentials come from private generated config. Never paste them into GitHub or screenshots. Credential rotation remains pending.

## 10. 🔄 Disable obsolete top-level proxy boot units

Only for the already-selected **no-proxy host-IP design**, inspect:

~~~bash
systemctl list-unit-files --type=service --no-pager |
  grep -Ei 'kolla.*(proxysql|haproxy)'
~~~

If these obsolete units exist, stop/disable them:

~~~bash
systemctl disable --now \
  kolla-proxysql-container.service \
  kolla-haproxy-container.service
~~~

If they do not exist in a fresh deployment, there is nothing to disable. Do not disable Neutron's namespace-specific metadata HAProxy helpers.

Verify `3306` belongs to MariaDB, not ProxySQL:

~~~bash
ss -lntp 'sport = :3306'
~~~

Do not change Docker restart policies to fix these units; the observed containers had `restart=no` and were started by systemd.
