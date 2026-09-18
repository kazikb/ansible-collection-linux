docker_engine
=============

This role installs and configures Docker Engine on Debian, Ubuntu and
Enterprise Linux systems.

The role performs the following tasks:

- Removes packages that conflict with Docker Engine.
- Adds the official Docker package repository for the target distribution.
- Installs Docker Engine, the Docker CLI, containerd, Buildx and the Docker
  Compose plugin.
- Enables and starts the Docker service.
- Creates and validates `/etc/docker/daemon.json`.
- Restarts Docker when the daemon configuration changes.

The daemon configuration is authoritative. All tasks are tagged with
`docker_engine`.

Requirements
------------

- Ansible Core 2.18.12 or newer.
- Managing Ubuntu 26.04 with its default Python 3.14 requires Ansible Core 2.20 or newer.
- Fact gathering must be enabled for the play.
- The target must use systemd.
- The play must use privilege escalation or otherwise run with root privileges.
- The target must be able to access the official Docker package repository.

Role Variables
--------------

- `docker_engine_daemon_bip` - IPv4 gateway address and subnet for the default
  `docker0` bridge. Default: `null`.
- `docker_engine_daemon_default_address_pool_list` - address pools used to
  allocate subnets for user-defined networks. Default: `[]`.
- `docker_engine_daemon_log_driver` - default logging driver for newly created
  containers. Default: `local`.
- `docker_engine_daemon_log_format` - Docker daemon log format: `text` or
  `json`. Default: `text`.
- `docker_engine_daemon_log_options` - options passed to the default container
  logging driver. Values must be strings. Default: `max-size: "20m"` and
  `max-file: "5"`.
- `docker_engine_daemon_log_level` - Docker daemon logging level: `debug`,
  `info`, `warn`, `error` or `fatal`. Default: `info`.
- `docker_engine_daemon_live_restore` - keep standalone containers running
  while the Docker daemon is unavailable. Default: `true`.

```yaml
docker_engine_daemon_bip: 172.18.0.1/16

docker_engine_daemon_default_address_pool_list:
  - base: 172.20.0.0/16
    size: 24
  - base: 172.21.0.0/16
    size: 24

docker_engine_daemon_log_driver: local
docker_engine_daemon_log_format: text
docker_engine_daemon_log_options:
  max-size: "20m"
  max-file: "5"
docker_engine_daemon_log_level: info
docker_engine_daemon_live_restore: true
```

Each address pool requires a `base` network in CIDR notation and the prefix
`size` of the subnets allocated from that network. When
`docker_engine_daemon_bip` is `null`, the `bip` setting is omitted. An empty
address pool list or logging options dictionary is also omitted from the
generated configuration.

The generated daemon configuration is validated with `dockerd` before it
replaces the existing file. Container logging changes apply only to containers
created after the configuration change.

Dependencies
------------

This role has no Ansible role dependencies and uses only modules included with
Ansible Core.

Example Playbook
----------------

```yaml
---
- name: Install and configure Docker Engine
  hosts: docker_hosts
  become: true
  gather_facts: true

  vars:
    docker_engine_daemon_bip: 172.18.0.1/16
    docker_engine_daemon_default_address_pool_list:
      - base: 172.20.0.0/16
        size: 24

  roles:
    - role: kazikb.linux.docker_engine
```

Run only this role's tagged tasks with:

```bash
ansible-playbook site.yml --tags docker_engine
```

License
-------

MIT

Author Information
------------------

Kazimierz Biskup [GitHub](https://github.com/kazikb/ansible-collection-linux)
