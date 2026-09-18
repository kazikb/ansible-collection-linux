# Changelog

## 0.1.0

- Package `system_baseline`, `access_management`, `host_hardening`, and `docker_engine` as the `kazikb.linux` collection, preserving role variables, tags, and behavior.
- Distribute the system updates playbook as `kazikb.linux.install_updates`.
- Add `kazikb.linux.bootstrap_install_sudo` to install sudo using su and add the non-root Ansible connection user to the sudo or wheel group.
- Require Ansible Core 2.18.12 or newer, with Core 2.20 or newer required to manage Ubuntu 26.04 with its default Python 3.14.
