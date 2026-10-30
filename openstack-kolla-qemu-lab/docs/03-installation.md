# 🧰 Install Kolla tooling

[← VM setup](02-vm-and-network.md) · [Next →](04-configuration-and-deploy.md)

> 🛠️ For a NEW lab. Do not reinstall an already-working environment just to replay this guide.

## 1. Host dependencies

Run as root:

~~~bash
apt update
apt install -y \
  git curl tmux python3-dev python3-venv python3-pip \
  build-essential libffi-dev libssl-dev \
  pkg-config libdbus-1-dev libglib2.0-dev
~~~

Python/compiler packages support isolated dependencies. DBus/GLib development packages allow building `dbus-python`. `tmux` preserves long tasks across SSH disconnects.

## 2. Venv

~~~bash
python3 -m venv /opt/kolla-venv
source /opt/kolla-venv/bin/activate
python -m pip install --upgrade pip
~~~

The venv separates Kolla tooling from system Python.

## 3. Selected release

~~~bash
python -m pip install \
  -c https://releases.openstack.org/constraints/upper/2026.1 \
  'git+https://opendev.org/openstack/kolla-ansible@stable/2026.1'
~~~

The observed installation was `22.2.1.dev26` with Ansible Core `2.20.10`. A moving stable branch is **not an immutable source pin**. Record versions and exact source commit in your private build record; review release requirements when fresh versions differ.

~~~bash
kolla-ansible --version
ansible --version
python -m pip show kolla-ansible ansible-core
~~~

## 4. Interpreter-specific dependencies

~~~bash
python -m pip install \
  -c https://releases.openstack.org/constraints/upper/2026.1 \
  docker dbus-python
python -c "import docker, dbus; print('Docker SDK:', docker.__version__); print('DBus import: OK')"
~~~

Localhost checks used the venv interpreter. Installing modules only into system Python did not satisfy it. The Python Docker SDK is different from Docker Engine.

## 5. Collections

~~~bash
kolla-ansible install-deps
~~~

Expected: successful collection installation.

## 6. Private config and inventory

~~~bash
install -d -m 700 /etc/kolla /root/kolla-lab
cp -a --update=none \
  /opt/kolla-venv/share/kolla-ansible/etc_examples/kolla/. /etc/kolla/
cp --update=none \
  /opt/kolla-venv/share/kolla-ansible/ansible/inventory/all-in-one \
  /root/kolla-lab/all-in-one
chmod 600 /etc/kolla/passwords.yml
~~~

`--update=none` avoids overwriting files on the observed Ubuntu coreutils. Check your utility's help if unsupported.

## 7. Generate credentials ONCE

~~~bash
kolla-genpwd
~~~

This writes private passwords. Do not rerun it to troubleshoot services. Never commit the resulting file.

## 8. Inventory check

~~~bash
ansible -i /root/kolla-lab/all-in-one all -m ping
~~~

Expected: `SUCCESS` and `"ping": "pong"`. This verifies Python/module execution, not ICMP. Interpreter/group-name warnings were observed but the task passed.

## 9. tmux

~~~bash
tmux new -s kolla-deploy
~~~

Inside a new shell, reactivate the venv. Detach with **Ctrl+B, then D**. Reconnect:

~~~bash
tmux ls
tmux attach -t kolla-deploy
~~~

Before repeating work after a disconnect:

~~~bash
pgrep -af 'kolla-ansible|ansible-playbook'
~~~

Do not launch a second deployment while another is running. A new tmux/root shell does not automatically inherit all previous exports.
