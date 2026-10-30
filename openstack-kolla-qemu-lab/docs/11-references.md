# 📖 Primary references and reproducibility

[← Roadmap](10-roadmap.md) · [Home](../README.md)

This repository is a sanitized case study derived from the observed lab session. Upstream commands and defaults may evolve; use release-specific docs and inspect your installed CLI.

## OpenStack

- [Kolla Ansible 2026.1 quickstart](https://docs.openstack.org/kolla-ansible/2026.1/user/quickstart.html)
- [Kolla Ansible 2026.1 operations](https://docs.openstack.org/kolla-ansible/2026.1/user/operating-kolla.html)
- [Neutron networking in Kolla](https://docs.openstack.org/kolla-ansible/2026.1/reference/networking/neutron.html)
- [Cinder backend guide](https://docs.openstack.org/kolla-ansible/2026.1/reference/storage/cinder-guide.html)
- [OpenStack image guide](https://docs.openstack.org/image-guide/obtain-images.html)
- [CirrOS releases](https://github.com/cirros-dev/cirros/releases)
- [OpenStackClient subnet commands](https://docs.openstack.org/python-openstackclient/2026.1/cli/command-objects/subnet.html)
- [OpenStackClient router commands](https://docs.openstack.org/python-openstackclient/2026.1/cli/command-objects/router.html)
- [OpenStackClient port commands](https://docs.openstack.org/python-openstackclient/2026.1/cli/command-objects/port.html)
- [Keystone default roles](https://docs.openstack.org/keystone/2026.1/admin/service-api-protection.html)

## Host tools

- [Docker container listing](https://docs.docker.com/reference/cli/docker/container/ls/)
- [systemd service documentation](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html)
- [Netplan documentation](https://netplan.readthedocs.io/)
- [Ansible variable precedence](https://docs.ansible.com/ansible/latest/reference_appendices/general_precedence.html)

## GitHub

- [Create a repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-new-repository)
- [Upload files](https://docs.github.com/en/repositories/working-with-files/managing-files/adding-a-file-to-a-repository)
- [Mermaid diagrams](https://docs.github.com/en/get-started/writing-on-github/working-with-advanced-formatting/creating-diagrams)
- [Ignoring files](https://docs.github.com/en/get-started/getting-started-with-git/ignoring-files)

## What these references do and do not prove

Upstream docs explain tool behavior; the lab's own output establishes what happened in this environment. Do not use this case study as evidence that all combinations of current packages, provider networking or OS images are certified.

The recorded `22.2.1.dev26` version, moving release branch and image tags are not immutable locks. Exact source commit and image digests still need recording for a fully reproducible build.
