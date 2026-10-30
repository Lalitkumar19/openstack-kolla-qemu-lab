# 🌐 Working router and host NAT design

[← First instance](05-first-instance.md) · [Next →](07-reboot-persistence.md)

> This chapter describes the working design, not the unsuccessful direct outer-router path. Commands change lab networking.

## 1. Preconditions

- Provider NIC is `enp8s0` and is attached to `br-ex`.
- Original outer private port owns `10.0.0.91`.
- Host has no other direct `10.0.0.91` address.
- `10.0.0.20` is reserved in the outer cloud to prevent allocation to another machine.
- These are dedicated lab subnets. Replace the entire address plan for another environment.
- Original management/default route through `enp3s0` remains intact.

In the outer portal, the session created an unattached reservation port for `.20` and kept it DOWN/unattached. This is an address reservation, not an API VIP.

The outer provider port was set to port-security disabled during earlier troubleshooting. That did not alone fix the return path. It is **not asserted as a necessary blanket setting for every host-NAT deployment**; reassess its scope.

## 2. 🔎 Verify OVS mapping

HOST:

~~~bash
docker exec neutron_openvswitch_agent \
  grep -n bridge_mappings /etc/neutron/plugins/ml2/openvswitch_agent.ini
docker exec openvswitch_vswitchd ovs-vsctl show
~~~

Expected `bridge_mappings = physnet1:br-ex`, and `enp8s0` as a `br-ex` member. Stop and adapt if your mapping differs.

## 3. 🛠️ Create inner external network

~~~bash
export OS_CLIENT_CONFIG_FILE=/etc/kolla/clouds.yaml
export OS_CLOUD=kolla-admin
/usr/bin/openstack network create lab-external \
  --external \
  --provider-network-type flat \
  --provider-physical-network physnet1 \
  --mtu 1442
/usr/bin/openstack subnet create lab-external-subnet \
  --network lab-external \
  --subnet-range 10.0.0.0/24 \
  --gateway 10.0.0.91 \
  --allocation-pool start=10.0.0.20,end=10.0.0.20 \
  --no-dhcp
~~~

“External” means external from the **inner** cloud. The subnet is still private IPv4. DHCP is disabled because gateway addresses are explicitly assigned.

## 4. 🛠️ Create and attach router

~~~bash
/usr/bin/openstack router create lab-router
/usr/bin/openstack router set lab-router \
  --external-gateway lab-external \
  --fixed-ip subnet=lab-external-subnet,ip-address=10.0.0.20 \
  --enable-snat
/usr/bin/openstack router add subnet lab-router lab-subnet
/usr/bin/openstack router show lab-router \
  -c status -c external_gateway_info -c interfaces_info
~~~

Expected: external `10.0.0.20`, internal `192.168.50.254` and SNAT enabled.

If using the existing session's resources, do not recreate them. Inspect their current values and use `set` only for a required correction.

## 5. 🛠️ Host bridge address

First verify no duplicate assignment:

~~~bash
ip -4 -br address
~~~

Apply runtime settings:

~~~bash
ip link set dev br-ex up
ip address replace 10.0.0.91/24 dev br-ex
~~~

The final host gateway is on the OVS **internal bridge interface**, not on `enp8s0`.

`br-ex UNKNOWN` can be normal for a virtual interface; inspect flags and connectivity rather than treating it as a failure automatically.

## 6. 🔎 Router-to-host test

Retrieve your actual router UUID, not one from a screenshot:

~~~bash
LAB_ROUTER_ID=$(/usr/bin/openstack router show lab-router -f value -c id)
docker exec --privileged -u 0 neutron_l3_agent \
  ip netns exec "qrouter-$LAB_ROUTER_ID" \
  ping -c 3 -W 2 10.0.0.91
~~~

Expected: replies. The diagnostic process receives elevated namespace access; this does not change the container's saved privilege configuration.

If the host gateway cannot be reached, fix it before adding NAT. Use `ip route` and `ip neigh` in the same namespace.

## 7. 🛠️ Forward only the intended traffic

The following runtime rules were used once during initial diagnosis:

~~~bash
iptables -w -I FORWARD 1 \
  -i br-ex -o enp3s0 -s 10.0.0.20/32 \
  -m conntrack --ctstate NEW,ESTABLISHED,RELATED -j ACCEPT
iptables -w -I FORWARD 1 \
  -i enp3s0 -o br-ex -d 10.0.0.20/32 \
  -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -w -t nat -A POSTROUTING \
  -s 10.0.0.20/32 -o enp3s0 -j MASQUERADE
sysctl -w net.ipv4.ip_forward=1
~~~

Do not repeatedly append duplicates. Use the idempotent persistence script in the next chapter for ongoing application.

| Rule | Why |
|---|---|
| Forward NEW/ESTABLISHED outbound | Permit the inner router's NAT source |
| Return ESTABLISHED/RELATED | Permit responses, not arbitrary new inbound traffic |
| MASQUERADE | Translate source to host uplink address |
| IPv4 forwarding | Enable host routing |

No firewall flush or global policy change is required. Existing Neutron/Docker chains are preserved.

## 8. 🔎 Router internet test

~~~bash
docker exec --privileged -u 0 neutron_l3_agent \
  ip netns exec "qrouter-$LAB_ROUTER_ID" \
  ping -c 3 -W 2 1.1.1.1
~~~

Expected: replies, observed 0% packet loss.

## 9. 🐧 Refresh the guest DHCP lease and test

Run in the **browser guest console**, not the HOST:

~~~sh
sudo dhcpcd -n eth0
~~~

Wait briefly, then:

~~~sh
hostname
ip route
cat /etc/resolv.conf
ping -c 3 1.1.1.1
ping -c 3 example.com
~~~

Expected: guest hostname, default via `192.168.50.254`, intended DNS, successful IP/domain tests. If a destination blocks ICMP, distinguish DNS resolution from ICMP filtering rather than assuming all internet is down.

## 10. What this does NOT implement

- No floating-IP public inbound access per guest.
- No generic firewall forwarding for all sources.
- No Cinder storage.
- No automatic guest startup after host reboot.
- No production routing/HA design.
