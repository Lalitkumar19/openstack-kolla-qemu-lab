# 📝 Change log

## Lab milestone — 2026-10-30 session

Completed: single-node Kolla deployment, QEMU CirrOS guest, noVNC, DHCP, guest internet/DNS, host NAT, systemd persistence, proxy/MariaDB conflict diagnosis, disabled obsolete proxy boot services and clean recovery of OpenStack services after reboot.

After the final reboot the guest was manually started and reported ACTIVE / Running. Its final connectivity repeat **inside the guest** remains pending; the latest pasted ping output came from the outer host.

## Publication preparation — 2026-10-31

- Reorganized commands into beginner-friendly phases.
- Separated the working host-NAT design from historical direct-routing attempts.
- Sanitized public addresses, UUIDs and credentials.
- Added architecture, GitHub UI instructions, reusable configuration, troubleshooting and roadmap.
