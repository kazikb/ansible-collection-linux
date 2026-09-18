# ansible-collection-linux

Reusable Ansible roles and playbooks for Debian, Ubuntu and Enterprise Linux hosts,
distributed as the `kazikb.linux` collection.

The roles and playbooks are developed for my personal lab environment and are
provided "as is". Review and test them on disposable hosts before using them
in your own environment.

Roles
-----

| Role | Description |
| --- | --- |
| [`access_management`](roles/access_management/README.md) | Manages local users, groups, SSH access, sudo rules and the root account. |
| [`docker_engine`](roles/docker_engine/README.md) | Installs and configures Docker Engine. |
| [`host_hardening`](roles/host_hardening/README.md) | Manages OpenSSH hardening, the host firewall, sysctl parameters and kernel modules. |
| [`system_baseline`](roles/system_baseline/README.md) | Configures a common operating system baseline. |

Playbooks
---------

| Playbook | Description |
| --- | --- |
| [`bootstrap_install_sudo`](playbooks/bootstrap_install_sudo.yml) | Uses su to install sudo and add the non-root Ansible connection user to the sudo or wheel group. |
| [`install_updates`](playbooks/install_updates.yml) | Installs system updates, removes unused packages and reboots hosts when required. |

Each playbook includes requirements and usage examples in its opening comments.

Requirements
------------

- Ansible Core 2.18.12 or newer.
- Managing Ubuntu 26.04 with its default Python 3.14 requires Ansible Core 2.20 or newer.
- Python `netaddr` 0.10.1 or newer on the controller.
- Roles require fact gathering, systemd and root privileges on managed hosts.

See the role READMEs and playbook comments for additional requirements.

Local Setup
-----------

From this repository, create a virtual environment and install the runtime
requirements and collection dependencies:

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml
```

Use in Another Repository
-------------------------

Create a consumer `requirements.yml` referencing a release tag:

```yaml
---
collections:
  - name: https://github.com/kazikb/ansible-collection-linux.git
    type: git
    version: v0.1.0
```

Replace `v0.1.0` with the published release tag you want to use.

With the controller environment prepared above, install the collection from
the consuming repository:

```bash
ansible-galaxy collection install -r requirements.yml
```

Collection dependencies are installed automatically by `ansible-galaxy`;
install Python dependencies separately from this repository's `requirements.txt`.

Reference roles by their fully qualified names:

```yaml
---
- name: Configure Linux hosts
  hosts: all
  become: true
  gather_facts: true
  roles:
    - role: kazikb.linux.system_baseline
    - role: kazikb.linux.access_management
    - role: kazikb.linux.host_hardening
```

For Docker hosts, add `kazikb.linux.docker_engine` before
`kazikb.linux.access_management` so managed users can be assigned to the Docker
group.

Keep inventories and host variables outside the tracked repository and pass
the inventory explicitly with `-i`.

Run packaged playbooks by their fully qualified names, for example:

```bash
ansible-playbook kazikb.linux.install_updates -i <inventory> --limit <host>
```

Development
-----------

Create and activate a Python virtual environment, then install the development
requirements and Ansible Galaxy collections:

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -r requirements-dev.txt
ansible-galaxy collection install -r requirements.yml
```

Tests playbooks use the source roles directly, so edits are available
immediately without reinstalling the collection. Run lint with:

```bash
ansible-lint --offline roles playbooks tests/*.yml galaxy.yml meta/runtime.yml
```

To install or refresh the local collection copy for calls such as
`kazikb.linux.install_updates`, run from the repository root:

```bash
ansible-galaxy collection install . -p ./collections --force --no-deps
```

Repeat after source changes; the installed copy does not update automatically.

Releases
--------

Update the version in [`galaxy.yml`](galaxy.yml) and record the release in
[`CHANGELOG.md`](CHANGELOG.md), then commit the changes and push a matching Git
tag such as `v0.1.0`.
