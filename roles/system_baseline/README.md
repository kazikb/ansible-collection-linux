system_baseline
===============

This role configures a common system baseline for Debian, Ubuntu and
Enterprise Linux systems.

The role performs the following tasks:

- Gathers service and package facts and loads OS-family-specific variables.
- Configures the hostname and a managed block of static `/etc/hosts` entries.
- Installs a fixed set of common and distribution-specific administration tools.
- Installs EPEL, enables CRB and installs additional packages on Enterprise Linux.
- Detects, enables and starts Chrony or systemd-timesyncd.
- Configures NTP servers when a server list is provided.
- Configures the system timezone and default locale.
- Installs additional locales or language packs for the target OS family.
- Configures global Vim settings and selects Vim as the system editor.
- Adds or removes trusted root CA certificates when requested.
- Optionally configures Postfix as a send-only null client.
- Optionally configures and enables automatic updates using unattended-upgrades
  or dnf-automatic.

The package baseline is defined internally by the role and is intentionally not
configurable. Optional features are controlled through variables documented
below. All tasks are tagged with `system_baseline`.

Requirements
------------

- Ansible Core 2.18.12 or newer.
- Managing Ubuntu 26.04 with its default Python 3.14 requires Ansible Core 2.20 or newer.
- The `community.general` collection, installed automatically with `kazikb.linux`.
- Fact gathering must be enabled for the play.
- The target must use systemd.
- Chrony or systemd-timesyncd must already be installed on the target.
- The play must use privilege escalation or otherwise run with root privileges.

Role Variables
--------------

### Identity

- `system_baseline_manage_hostname` - set the hostname to
  `inventory_hostname_short`. Default: `true`.
- `system_baseline_hosts_entries` - list of complete lines managed in a dedicated
  block in `/etc/hosts`. Default: `[]`.
- `system_baseline_remove_debian_hostname_entry` - remove the Debian
  `127.0.1.1` hostname entry. Default: `false`.

```yaml
system_baseline_manage_hostname: true
system_baseline_remove_debian_hostname_entry: true
system_baseline_hosts_entries:
  - 192.0.2.10 server1.example.com server1
  - 192.0.2.11 server2.example.com server2
```

When `system_baseline_remove_debian_hostname_entry` is enabled, the managed host
entries should provide resolution for the local hostname to its real address.

### Time synchronization

- `system_baseline_timezone` - system timezone. Default: `Europe/Warsaw`.
- `system_baseline_ntp_server_list` - NTP servers used by the detected time
  service. Default: `[]`.

```yaml
system_baseline_timezone: Europe/Warsaw
system_baseline_ntp_server_list:
  - ntp1.example.com
  - ntp2.example.com
```

The role detects `chrony.service`, `chronyd.service` or
`systemd-timesyncd.service`, in that order, and ensures the detected service is
enabled and started. An empty server list leaves the service configuration
unchanged.

For Chrony, each list item is rendered as `server <name> iburst`. Existing
`server` and `pool` directives in the main Chrony configuration file are
disabled when a managed server list is configured. For systemd-timesyncd, the
list is written to the `NTP=` setting in `/etc/systemd/timesyncd.conf`.

### Locale

- `system_baseline_locale` - system-wide default locale. Default:
  `en_US.UTF-8`.
- `system_baseline_debian_locale_to_add` - locales to generate or remove on
  Debian and Ubuntu.
- `system_baseline_redhat_locale_to_add` - language-pack packages to install or
  remove on Enterprise Linux.

```yaml
system_baseline_locale: en_US.UTF-8

system_baseline_debian_locale_to_add:
  - name: en_US.UTF-8
    state: present
  - name: pl_PL.UTF-8
    state: present

system_baseline_redhat_locale_to_add:
  - name: glibc-langpack-en
    state: present
  - name: glibc-langpack-pl
    state: present
```

### Mail relay

- `system_baseline_mta_enabled` - install and configure Postfix as a send-only
  null client. Default: `false`.
- `system_baseline_mta_notification_email` - address that receives root,
  postmaster and host notifications.
- `system_baseline_mta_smtp_relayhost` - SMTP relay hostname or address.
- `system_baseline_mta_smtp_relay_port` - SMTP relay port.
- `system_baseline_mta_sender_domain` - domain used for rewritten sender
  addresses.
- `system_baseline_mta_smtp_tls_security_level` - Postfix SMTP TLS security
  level: `none`, `may` or `encrypt`. Default: `may`.

The relay, port, sender domain and notification address must be set when MTA
management is enabled.

```yaml
system_baseline_mta_enabled: true
system_baseline_mta_notification_email: alerts@example.com
system_baseline_mta_smtp_relayhost: smtp.example.com
system_baseline_mta_smtp_relay_port: 25
system_baseline_mta_sender_domain: example.com
system_baseline_mta_smtp_tls_security_level: may
```

### Automatic updates

- `system_baseline_automatic_updates_enabled` - configure automatic package
  updates. Default: `false`.
- `system_baseline_automatic_updates_upgrade_type` - install `security` updates
  or `all` available updates. Default: `security`.
- `system_baseline_automatic_updates_automatic_reboot` - automatically reboot
  when required after updates. Default: `false`.
- `system_baseline_automatic_updates_upgrade_time` - daily update time used by
  the systemd timer. Default: `02:00`.
- `system_baseline_automatic_updates_upgrade_randomized_delay_sec` - randomized
  delay for the update timer. Default: `15m`.

```yaml
system_baseline_automatic_updates_enabled: true
system_baseline_automatic_updates_upgrade_type: security
system_baseline_automatic_updates_automatic_reboot: true
system_baseline_automatic_updates_upgrade_time: "02:00"
system_baseline_automatic_updates_upgrade_randomized_delay_sec: "15m"
```

On Debian and Ubuntu, the role installs unattended-upgrades and enables the
`apt-daily.timer` and `apt-daily-upgrade.timer` units. These additional variables
control APT behavior:

- `system_baseline_automatic_updates_apt_mail_report` - mail report policy:
  `always`, `on-change` or `only-on-error`. Default: `on-change`.
- `system_baseline_automatic_updates_apt_download_time` - daily package-list
  download times. Default: `05,22:00`.
- `system_baseline_automatic_updates_apt_download_randomized_delay_sec` -
  randomized delay for package-list downloads. Default: `10m`.
- `system_baseline_automatic_updates_apt_package_blacklist` - package-name
  regular expressions excluded from unattended upgrades. Default: `[]`.

```yaml
system_baseline_automatic_updates_apt_mail_report: on-change
system_baseline_automatic_updates_apt_download_time: "05,22:00"
system_baseline_automatic_updates_apt_download_randomized_delay_sec: "10m"
system_baseline_automatic_updates_apt_package_blacklist:
  - linux-image-.*
```

On Enterprise Linux, the role installs dnf-automatic and enables
`dnf-automatic.timer`.

- `system_baseline_automatic_updates_dnf_package_exclude` - package names or
  globs excluded from automatic updates. Default: `[]`.

```yaml
system_baseline_automatic_updates_dnf_package_exclude:
  - kernel*
  - podman*
```

Email reports are enabled only when `system_baseline_mta_enabled` is also true.

### Root CA certificates

- `system_baseline_root_ca_list` - root CA certificates to add to or remove from
  the system trust store. Default: `[]`.

```yaml
system_baseline_root_ca_list:
  - filename: example-root-ca.crt
    state: present
    content: "{{ lookup('file', 'files/example-root-ca.crt') }}"
  - filename: obsolete-root-ca.crt
    state: absent
```

Trust-store changes are applied immediately after certificate files change.

Dependencies
------------

This role has no Ansible role dependencies. It uses modules from the
`community.general` collection installed with `kazikb.linux`.

Example Playbook
----------------

```yaml
---
- name: Apply system baseline
  hosts: servers
  become: true
  gather_facts: true

  vars:
    system_baseline_hosts_entries:
      - 192.0.2.10 server1.example.com server1
      - 192.0.2.11 server2.example.com server2

    system_baseline_ntp_server_list:
      - ntp1.example.com
      - ntp2.example.com

    system_baseline_mta_enabled: true
    system_baseline_mta_notification_email: alerts@example.com
    system_baseline_mta_smtp_relayhost: smtp.example.com
    system_baseline_mta_smtp_relay_port: 25
    system_baseline_mta_sender_domain: example.com

    system_baseline_automatic_updates_enabled: true
    system_baseline_automatic_updates_upgrade_type: security

  roles:
    - role: kazikb.linux.system_baseline
```

Run only this role's tagged tasks with:

```bash
ansible-playbook site.yml --tags system_baseline
```

License
-------

MIT

Author Information
------------------

Kazimierz Biskup [GitHub](https://github.com/kazikb/ansible-collection-linux)
