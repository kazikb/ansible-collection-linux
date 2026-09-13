host_hardening
==============

This role manages host security settings on Debian, Ubuntu and Enterprise Linux
systems.

The role performs the following tasks:

- Optionally hardens the OpenSSH server.
- Optionally manages the native host firewall using nftables, UFW or firewalld.
- Optionally manages sysctl parameters.
- Optionally manages kernel module runtime and persistent state.
- Validates managed firewall, sysctl and kernel module definitions.
- Checks that an enabled firewall permits every listening SSH port.

SSH, firewall, sysctl and kernel module management are independently opt-in.
Disabling a management flag leaves the corresponding existing configuration
unchanged. All tasks are tagged with `host_hardening`.

Requirements
------------

- Ansible Core 2.18.12 or newer.
- Managing Ubuntu 26.04 with its default Python 3.14 requires Ansible Core 2.20 or newer.
- The `community.general`, `ansible.posix` and `ansible.utils` collections,
  installed automatically with `kazikb.linux`.
- Python `netaddr` 0.10.1 or newer on the Ansible controller.
- Fact gathering must be enabled for the play.
- The target must use systemd.
- The play must use privilege escalation or otherwise run with root privileges.
- OpenSSH server must be installed when SSH hardening is enabled.
- A running SSH daemon and either `ss` or `netstat` are required when the
  firewall is managed and enabled.
- `update-initramfs` must be available on Debian and Ubuntu, and `dracut` on
  Enterprise Linux, when kernel module management changes persistent state.

Role Variables
--------------

### OpenSSH hardening

- `host_hardening_sshd_manage` - manage the SSH hardening configuration.
  Default: `false`.
- `host_hardening_sshd_permit_root_login` - root login policy:
  `forced-commands-only`, `no` or `prohibit-password`. Default: `no`.
- `host_hardening_sshd_max_auth_tries` - maximum authentication attempts per
  connection. Default: `3`.
- `host_hardening_sshd_login_grace_time` - time allowed to authenticate.
  Default: `1m`.
- `host_hardening_sshd_jump_host` - allow local TCP forwarding for
  `ProxyJump` or `ssh -J`. Default: `false`.
- `host_hardening_sshd_algorithm_policy` - `distribution`, `remove` or
  `replace`. Default: `distribution`.
- `host_hardening_sshd_kex_algorithms` - SSH key-exchange algorithms. Default:
  `[]`.
- `host_hardening_sshd_ciphers` - SSH encryption algorithms. Default: `[]`.
- `host_hardening_sshd_macs` - SSH message authentication algorithms. Default:
  `[]`.
- `host_hardening_sshd_host_key_algorithms` - server host-key signature
  algorithms. Default: `[]`.
- `host_hardening_sshd_pubkey_accepted_algorithms` - accepted user public-key
  signature algorithms. Default: `[]`.

```yaml
host_hardening_sshd_manage: true
host_hardening_sshd_permit_root_login: "no"
host_hardening_sshd_max_auth_tries: 3
host_hardening_sshd_login_grace_time: 1m
host_hardening_sshd_jump_host: false

host_hardening_sshd_algorithm_policy: remove
host_hardening_sshd_kex_algorithms:
  - diffie-hellman-group14-sha1
host_hardening_sshd_ciphers:
  - aes128-cbc
host_hardening_sshd_macs:
  - hmac-sha1
```

When enabled, the role writes
`/etc/ssh/sshd_config.d/10-host-hardening.conf`. It requires public-key
authentication, disables password authentication and most forwarding features,
and validates the SSH configuration before reloading the service. Enabling jump
host support permits local TCP forwarding only.

The `distribution` algorithm policy leaves distribution defaults unchanged.
The `remove` policy removes listed algorithms from those defaults, while
`replace` replaces a default with each corresponding non-empty list. Empty
lists do not generate directives.

The role also removes active Diffie-Hellman moduli shorter than 3071 bits from
`/etc/ssh/moduli`.

### Host firewall

- `host_hardening_firewall_manage` - manage the host firewall. Default:
  `false`.
- `host_hardening_firewall_enabled` - enable and start the firewall service.
  Default: `true`.
- `host_hardening_firewall_logging` - log denied incoming traffic. Default:
  `false`.
- `host_hardening_firewall_default_incoming_policy` - `drop` or `reject`
  unmatched incoming traffic. Default: `drop`.
- `host_hardening_firewall_ingress_rules_list` - managed ingress rules.
  Default: `[]`.
- `host_hardening_firewall_ingress_rules_list_append` - additional ingress
  rules appended to the primary list. Default: `[]`.

Each firewall rule requires:

- `name` - unique rule name.
- `state` - `present` or `absent`.
- `protocol` - `tcp` or `udp`.
- `destination_port` - a port from 1 through 65535 or an inclusive range using
  `start-end` syntax.
- `source` - `any`, an IP address or a CIDR network.

```yaml
host_hardening_firewall_manage: true
host_hardening_firewall_enabled: true
host_hardening_firewall_logging: true
host_hardening_firewall_default_incoming_policy: drop

host_hardening_firewall_ingress_rules_list:
  - name: ssh-access
    state: present
    protocol: tcp
    destination_port: "22"
    source: any

  - name: https-access
    state: present
    protocol: tcp
    destination_port: "443"
    source: 192.0.2.0/24
```

Rule names and source, protocol and port combinations must be unique across the
primary and append lists.

When firewall management is enabled, the role validates all rule definitions.
When the firewall is also enabled, the role discovers listening `sshd` ports
and requires each port to be covered by a `present` TCP rule. This check does
not verify that the rule source permits the Ansible controller.

The firewall backend depends on the target:

- Debian uses nftables. The role replaces `/etc/nftables.conf` with its managed
  configuration. Disabling nftables stops the service and flushes the complete
  active nftables ruleset.
- Ubuntu uses UFW. The role enables IPv6 and manages incoming rules, default
  policies and logging.
- Enterprise Linux uses firewalld and the `public` zone. The role removes the
  `cockpit`, `dhcpv6-client` and `ssh` services from that zone and manages its
  target, rules and logging.

On Debian, the rendered nftables configuration is authoritative. On Ubuntu and
Enterprise Linux, include a previously managed rule with `state: absent` to
remove it. Omitting the rule does not remove it.

Setting `host_hardening_firewall_manage` to `false` leaves the firewall
unchanged. Setting `host_hardening_firewall_enabled` to `false` while management
remains enabled installs the native backend if needed and disables it.

### Sysctl parameters

- `host_hardening_sysctl_manage` - manage sysctl parameters. Default: `false`.
- `host_hardening_sysctl_params_list` - managed sysctl parameters. Default:
  `[]`.
- `host_hardening_sysctl_params_list_append` - additional parameters appended
  to the primary list. Default: `[]`.

Each definition requires a unique `name` and a `state` of `present` or
`absent`. A string `value` is required when `state` is `present`.

```yaml
host_hardening_sysctl_manage: true
host_hardening_sysctl_params_list:
  - name: kernel.randomize_va_space
    value: "2"
    state: present
  - name: kernel.panic
    state: absent
```

The role manages `/etc/sysctl.d/90-host-hardening.conf` and applies present
values to the running kernel. Omitted parameters are not changed. Use
`state: absent` to remove a previously managed parameter from the file.

Setting `host_hardening_sysctl_manage` to `false` leaves persistent and runtime
values unchanged.

### Kernel modules

- `host_hardening_kernel_modules_manage` - manage kernel modules. Default:
  `false`.
- `host_hardening_kernel_modules_list` - managed kernel modules. Default: `[]`.
- `host_hardening_kernel_modules_list_append` - additional modules appended to
  the primary list. Default: `[]`.
- `host_hardening_kernel_modules_reboot_on_change` - reboot after persistent
  module configuration changes. Default: `false`.

Each definition requires a unique `name` and one of these states:

- `enabled` - load the module now and at boot. Optional `options` are written to
  the role-managed modprobe file.
- `configured` - write required module `options` without changing runtime state.
- `disabled` - unload and block the module. `options` must be empty.

```yaml
host_hardening_kernel_modules_manage: true
host_hardening_kernel_modules_list:
  - name: dummy
    options: numdummies=1
    state: enabled
  - name: cramfs
    state: disabled

host_hardening_kernel_modules_reboot_on_change: false
```

The role owns `/etc/modules-load.d/host-hardening.conf` and
`/etc/modprobe.d/host-hardening.conf`. It removes either file when the combined
module list no longer requires it. Persistent changes rebuild all initramfs
images and optionally reboot the target. Runtime-only changes do not trigger a
reboot.

Setting `host_hardening_kernel_modules_manage` to `false` leaves persistent and
runtime state unchanged.

Dependencies
------------

This role has no Ansible role dependencies. It uses the `community.general`,
`ansible.posix` and `ansible.utils` collections installed with `kazikb.linux`.

Example Playbook
----------------

```yaml
---
- name: Apply host hardening
  hosts: servers
  become: true
  gather_facts: true

  vars:
    host_hardening_sshd_manage: true
    host_hardening_sshd_permit_root_login: "no"

    host_hardening_firewall_manage: true
    host_hardening_firewall_enabled: true
    host_hardening_firewall_ingress_rules_list:
      - name: ssh-access
        state: present
        protocol: tcp
        destination_port: "22"
        source: any

    host_hardening_sysctl_manage: true
    host_hardening_sysctl_params_list:
      - name: kernel.randomize_va_space
        value: "2"
        state: present

    host_hardening_kernel_modules_manage: true
    host_hardening_kernel_modules_list:
      - name: cramfs
        state: disabled

  roles:
    - role: kazikb.linux.host_hardening
```

The SSH rule must cover the target's actual listening SSH port before enabling
firewall management.

Run only this role's tagged tasks with:

```bash
ansible-playbook site.yml --tags host_hardening
```

License
-------

MIT

Author Information
------------------

Kazimierz Biskup [GitHub](https://github.com/kazikb/ansible-collection-linux)
