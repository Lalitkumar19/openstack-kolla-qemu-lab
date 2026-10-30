# 🐧 Create the first QEMU instance

[← Deployment](04-configuration-and-deploy.md) · [Next →](06-router-and-host-nat.md)

> 🛠️ Commands create new lab resources. Do not repeat create commands against existing names without checking first.

## 1. Select the private client configuration

HOST:

~~~bash
export OS_CLIENT_CONFIG_FILE=/etc/kolla/clouds.yaml
export OS_CLOUD=kolla-admin
/usr/bin/openstack image list
/usr/bin/openstack flavor list
/usr/bin/openstack network list
~~~

Initial lists were empty.

## 2. Download a small test image

~~~bash
install -d /root/kolla-lab/images
cd /root/kolla-lab/images
curl -fL --retry 3 \
  https://download.cirros-cloud.net/0.6.3/cirros-0.6.3-x86_64-disk.img \
  -o cirros-0.6.3-x86_64-disk.img
curl -fL --retry 3 \
  https://download.cirros-cloud.net/0.6.3/SHA256SUMS \
  -o SHA256SUMS
sha256sum --ignore-missing -c SHA256SUMS
~~~

Check that the downloaded image's checksum reports **OK** before uploading. Stop on a download/checksum failure. CirrOS is a cloud test image, not a production application OS.

## 3. Upload to Glance

~~~bash
/usr/bin/openstack image create cirros-0.6.3 \
  --file /root/kolla-lab/images/cirros-0.6.3-x86_64-disk.img \
  --disk-format qcow2 \
  --container-format bare \
  --private
/usr/bin/openstack image show cirros-0.6.3 \
  -c name -c status -c size -c disk_format
~~~

| Field | Expected |
|---|---|
| Status | `active` after upload |
| Format | `qcow2` |
| Observed size | 21,692,416 bytes, approximately 21 MB |

Initial create output briefly showed `queued`; a subsequent check showed `active`. Do not create duplicates merely because of that transient state.

## 4. Create a flavor

~~~bash
/usr/bin/openstack flavor create lab.small \
  --vcpus 1 --ram 512 --disk 1
~~~

A flavor defines resource sizing: **1 vCPU, 512 MB RAM, 1 GB local root disk**. This does not create a Cinder volume.

## 5. Create the internal network

This clean recipe uses the **final address plan from the start**, rather than repeating the earlier gateway/pool errors.

~~~bash
/usr/bin/openstack network create lab-net
/usr/bin/openstack subnet create lab-subnet \
  --network lab-net \
  --subnet-range 192.168.50.0/24 \
  --allocation-pool start=192.168.50.1,end=192.168.50.253 \
  --gateway 192.168.50.254 \
  --dns-nameserver 1.1.1.1 \
  --dns-nameserver 8.8.8.8 \
  --dhcp
~~~

The gateway address is reserved for the later router and excluded from the allocation pool. The network remains isolated until the router/uplink phase is complete.

Historical note: the session initially used `--gateway none` and later added the gateway. See troubleshooting if updating an existing subnet; do not recreate it.

## 6. Launch the guest

~~~bash
/usr/bin/openstack server create lab-vm01 \
  --image cirros-0.6.3 \
  --flavor lab.small \
  --network lab-net \
  --config-drive true
~~~

- `--image` selects Glance boot media.
- `--flavor` selects sizing.
- `--network` requests an inner virtual port.
- `--config-drive true` supplies supported metadata via an attached config drive.
- No Cinder boot volume or SSH key was configured for this initial console-only test.

## 7. 🔎 Check scheduling and boot

~~~bash
/usr/bin/openstack server show lab-vm01 \
  -c status -c OS-EXT-STS:task_state \
  -c OS-EXT-STS:power_state -c addresses -c fault
/usr/bin/openstack console log show lab-vm01 --lines 60
~~~

Expected sequence: `BUILD/scheduling` → `ACTIVE/Running`, DHCP lease and guest login prompt. QEMU may need time. A fresh lease can differ from observed `192.168.50.65`.

`console log show` **only displays logs**; it does not log you into the guest.

## 8. 🌐 Browser console login

Horizon → **Project → Compute → Instances → lab-vm01 → Console**.

Public CirrOS test credentials:

| Item | Value |
|---|---|
| Username | `cirros` |
| Password | `gocubsgo` |

These are the image's well-known test defaults, not Horizon admin credentials and not Nova's generated `adminPass`. Do not attach an openly exposed public address to an unchanged default-password test guest.

## 9. 🐧 Inside the guest only

~~~sh
hostname
whoami
ip -4 address show
ip route
df -h /
~~~

Expected observed values: hostname `lab-vm01`, user `cirros`, address `192.168.50.65`, root filesystem approximately 960.8 MB.

If hostname is `horizon` and disk is approximately 484 GB, you are running on the HOST, not the guest.

At this phase, internet is not expected until [router + host NAT](06-router-and-host-nat.md) is configured.
