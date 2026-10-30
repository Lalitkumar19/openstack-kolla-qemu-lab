# 🎯 Roadmap — planned work

[← Troubleshooting](09-troubleshooting.md) · [References →](11-references.md)

> None of the unchecked capabilities below are claimed as implemented.

## Immediate completion

- [ ] Repeat guest IP + DNS tests inside the console after final clean reboot.
- [ ] Rotate exposed admin credentials while synchronizing saved Kolla and client configuration.
- [ ] Configure HTTPS and narrow administrative access.
- [ ] Capture sanitized evidence and record exact dependency/source versions.
- [ ] Reassess historical outer network/port-security changes for the final design.

## Phase 2 — Cinder

- [ ] Attach a separate blank disk/volume to the lab host.
- [ ] Verify device identity, mounts and contents before provisioning.
- [ ] Configure a Cinder LVM backend.
- [ ] Test create/attach/detach, persistence and boot-from-volume.
- [ ] Document capacity and cleanup.

**Do not format the existing root disk `/dev/sda`.** A blank disk must be positively identified before any destructive storage command. Cinder is not made operational by merely enabling a dashboard menu.

## Phase 3 — OS image catalog

| Image family | Validation needed |
|---|---|
| Ubuntu / Debian | Official cloud image, cloud-init, SSH, network |
| Rocky / AlmaLinux | Image compatibility, cloud-init, guest access |
| Windows | Valid licensing/evaluation terms, VirtIO, Cloudbase-Init, credentials/RDP |
| Firewall appliance | Selected product, compatible image, multi-NIC routing/NAT tests |

Uploading an image is not the same as validating it. QEMU software emulation may make Windows/heavy appliances slow.

## Phase 4 — users, projects and RBAC

- [ ] Separate admin and ordinary project users.
- [ ] Validate project-scoped roles and quota limits.
- [ ] Verify users cannot list/change another tenant's resources.
- [ ] Test authorization through APIs, not just hidden UI buttons.
- [ ] Implement safe credential/token handling.

## Phase 5 — custom self-service portal

~~~mermaid
flowchart TD
    B["Browser: client or admin"]
    API["Application backend"]
    K["Keystone scoped identity"]
    OS["OpenStack service APIs"]
    B --> API
    API --> K
    API --> OS
    K -. "Authorization context" .-> OS
~~~

Client scope: own VMs, approved images/flavors, own volumes/networks.

Admin scope: authorized users/projects, quotas, image/flavor publishing and infrastructure health.

**Do not place a shared OpenStack admin password in frontend JavaScript.** Backend authorization and OpenStack policy enforcement both need tests.

## Phase 6 — operational maturity

- [ ] Automatic guest recovery/start policy, if desired.
- [ ] Backup/restore tests and image lifecycle.
- [ ] Monitoring, logs, capacity alerts and audit trails.
- [ ] Deliberate inbound access design.
- [ ] A multi-node/KVM environment before production/HA claims.

## Resume wording

Current scope:

> Deployed a single-node OpenStack lab using Kolla Ansible on Ubuntu, provisioned QEMU instances, configured Neutron networking and persistent host NAT, and validated core service recovery after reboot.

Only after implementation/testing:

> Developed an OpenStack-integrated self-service portal with project-based access control, VM lifecycle management and Cinder storage workflows.

Keep the single-node learning scope explicit. Do not claim HA, production-scale performance or customer operations that were not tested.
