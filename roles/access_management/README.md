access_management
=================

This role manages local users, groups, SSH access, sudo rules and the root
account on Debian, Ubuntu and Enterprise Linux systems.

The role performs the following tasks:

- Validates group, user and SSH authorized-key definitions.
- Creates local groups before managing users and removes obsolete groups after
  user and sudo configuration is complete.
- Creates, modifies or removes local user accounts.
- Manages user attributes, password hashes and supplementary group memberships.
- Adds administrative users to the OS-specific `sudo` or `wheel` group.
- Adds SSH-enabled users to a dedicated access group.
- Adds or removes SSH authorized keys for local users.
- Creates validated sudoers fragments for users and groups.
- Optionally restricts SSH access with an `AllowGroups` configuration fragment.
- Optionally configures the root account and its SSH authorized keys.
- Validates the SSH daemon configuration before reloading the service.

The role treats the configured supplementary group list as authoritative for
each managed user. All tasks are tagged with `access_management`.

Requirements
------------

- Ansible Core 2.18.12 or newer.
- Managing Ubuntu 26.04 with its default Python 3.14 requires Ansible Core 2.20 or newer.
- The `ansible.posix` collection, installed automatically with `kazikb.linux`.
- Fact gathering must be enabled for the play.
- The target must use systemd.
- OpenSSH server must be installed when SSH access management is enabled.
- The `sudo` package and OS-specific `sudo` or `wheel` group must exist when
  sudo access or custom sudo rules are configured.
- The play must use privilege escalation or otherwise run with root privileges.

Role Variables
--------------

### SSH access management

- `access_management_sshd_manage_access` - restrict SSH access to members of the
  configured access group. Default: `false`.
- `access_management_sshd_access_group` - group whose members are allowed to
  connect through SSH. Cannot be `sudo` or `wheel`. Default: `sshusers`.

```yaml
access_management_sshd_manage_access: true
access_management_sshd_access_group: sshusers
```

When SSH access management is enabled, the role creates the access group and
writes `/etc/ssh/sshd_config.d/40-access-management.conf` with an `AllowGroups`
directive. The SSH daemon configuration is validated before the service is
reloaded.

The current Ansible connection user must be a non-root account managed by this
role with `state: present` and `ssh_access: true`. For initial provisioning over
root SSH, first create the non-root management account with SSH access management
disabled, reconnect as that account and then enable SSH access management.

Disabling SSH access management removes the configuration fragment and the
configured access group.

### Root account

- `access_management_root_account_manage` - manage the root account. Default:
  `false`.
- `access_management_root_account_password_lock` - lock password authentication
  for the root account. Default: `true`.
- `access_management_root_account_password` - encrypted root password hash.
  Default: `null`.
- `access_management_root_account_shell` - root login shell. Default:
  `/bin/bash`.
- `access_management_root_account_ssh_authorized_keys` - SSH authorized keys to
  add to or remove from the root account. Default: `[]`.

```yaml
access_management_root_account_manage: true
access_management_root_account_password_lock: true
access_management_root_account_shell: /bin/bash
access_management_root_account_ssh_authorized_keys:
  - pubkey: "ssh-ed25519 AAAA... root@example"
    state: present
```

Root-account changes are applied only when
`access_management_root_account_manage` is enabled. An encrypted password hash
must be provided when password locking is disabled.

When Ansible initially uses password-based `su` to become root, do not enable
root-account management in the same run that creates the replacement sudo
account. Locking the root password prevents subsequent `su` elevation, and the
role cannot reliably detect a become method supplied only through the
`ansible-playbook` command line.

For the initial provisioning run, temporarily override
`access_management_root_account_manage` to `false`. After the role creates the
management account, reconnect with that account using `sudo` and run the
playbook again with root-account management enabled. Do not use
`access_management_root_account_password_lock: false` to defer the change;
that value requests an unlocked root password and requires an encrypted root
password hash.

Each root authorized-key definition requires:

- `pubkey` - SSH public key.
- `state` - `present` or `absent`.

### Local groups

- `access_management_groups_list` - local groups to create, modify or remove.
  Default: `[]`.
- `access_management_groups_list_append` - additional group definitions appended
  to the primary group list. Default: `[]`.

Each group definition supports:

- `name` - local group name. Required.
- `state` - `present` or `absent`. Required.
- `gid` - numeric group ID.
- `system` - create a system group. Default: `false`.
- `sudo_rules` - sudoers rules applied to the group. Default: `[]`.

```yaml
access_management_groups_list:
  - name: platform
    state: present
    gid: 1500
    system: false

  - name: operators
    state: present
    sudo_rules:
      - "ALL=(root) /usr/bin/journalctl"
      - "ALL=(root) /usr/bin/systemctl status *"

  - name: obsolete_group
    state: absent
```

Group sudo rules are written to dedicated files in `/etc/sudoers.d` and
validated with `visudo` before installation. Each rule is rendered after the
`%group` identifier and must contain the remaining sudoers rule, beginning with
the host specification.

Group names must be unique across `access_management_groups_list` and
`access_management_groups_list_append`. The `sudo`, `wheel` and configured SSH
access groups are reserved and cannot be managed through these lists.

### Local users

- `access_management_users_list` - local user accounts to create, modify or
  remove. Default: `[]`.
- `access_management_users_list_append` - additional user definitions appended
  to the primary user list. Default: `[]`.

Each user definition supports:

- `username` - local account name. Required.
- `state` - `present` or `absent`. Required.
- `uid` - numeric user ID. UID 0 is not allowed.
- `password` - encrypted account password hash.
- `password_lock` - lock or unlock password authentication.
- `comment` - account description, also known as GECOS.
- `shell` - login shell.
- `create_home` - create the home directory. Default: `true`.
- `home` - home directory path.
- `move_home` - move the existing home directory when its path changes. Default:
  `false`.
- `group` - primary group.
- `groups` - supplementary groups to assign. Default: `[]`.
- `umask` - account umask.
- `system` - create a system account. Default: `false`.
- `remove` - remove the home directory and mail spool with an absent account.
  Default: `false`.
- `sudo` - add the account to the OS-specific `sudo` or `wheel` group. Default:
  `false`.
- `sudo_nopasswd` - allow unrestricted sudo access without a password. Requires
  `sudo: true`. Default: `false`.
- `sudo_rules` - custom sudoers rules applied to the account. Default: `[]`.
- `ssh_access` - add the account to the SSH access group when SSH access
  management is enabled. Default: `false`.
- `ssh_authorized_keys` - SSH authorized keys to add or remove. Default: `[]`.

```yaml
access_management_users_list:
  - username: ansible
    state: present
    comment: Ansible automation account
    shell: /bin/bash
    create_home: true
    groups:
      - platform
    sudo: true
    sudo_nopasswd: true
    ssh_access: true
    ssh_authorized_keys:
      - pubkey: "ssh-ed25519 AAAA... ansible@example"
        state: present

  - username: application
    state: present
    comment: Application service account
    system: true
    password_lock: true
    shell: /usr/sbin/nologin
    create_home: false

  - username: backup
    state: present
    sudo_rules:
      - "ALL=(root) NOPASSWD: /usr/local/sbin/run-backup"

  - username: obsolete_user
    state: absent
    remove: true
```

Supplementary membership is authoritative: managed users are removed from
groups that are not included in the computed group list. Groups managed with
`state: present` are created before users; other requested groups must already
exist. A missing group causes the role to fail. The `sudo`, `wheel` and
configured SSH access groups must be managed through the `sudo` and `ssh_access`
options rather than listed in `groups`.

Each authorized-key definition requires:

- `pubkey` - SSH public key.
- `state` - `present` or `absent`.

User sudo rules are written to dedicated files in `/etc/sudoers.d` and validated
with `visudo` before installation. Each custom rule is rendered after the
username and must contain the remaining sudoers rule, beginning with the host
specification. Enabling `sudo_nopasswd` creates an unrestricted
`ALL=(ALL) NOPASSWD: ALL` rule.

Linux password values must be encrypted hashes. Store password hashes in
Ansible Vault rather than plaintext inventory files. See the
[Ansible password FAQ](https://docs.ansible.com/ansible/latest/reference_appendices/faq.html#how-do-i-generate-encrypted-passwords-for-the-user-module)
for supported hash-generation methods.

Usernames must be unique across `access_management_users_list` and
`access_management_users_list_append`. The root account and accounts with UID 0
must be managed separately through the root-account variables.

Dependencies
------------

This role has no Ansible role dependencies. It uses the
`ansible.posix.authorized_key` module from the `ansible.posix` collection installed
with `kazikb.linux`.

Example Playbook
----------------

```yaml
---
- name: Apply access management
  hosts: servers
  remote_user: ansible
  become: true
  gather_facts: true

  vars:
    access_management_sshd_manage_access: true
    access_management_sshd_access_group: sshusers

    access_management_groups_list:
      - name: platform
        state: present

      - name: operators
        state: present
        sudo_rules:
          - "ALL=(root) /usr/bin/journalctl"

    access_management_users_list:
      - username: ansible
        state: present
        comment: Ansible automation account
        shell: /bin/bash
        groups:
          - platform
        sudo: true
        sudo_nopasswd: true
        ssh_access: true
        ssh_authorized_keys:
          - pubkey: "ssh-ed25519 AAAA... ansible@example"
            state: present

      - username: operator
        state: present
        comment: Operations account
        groups:
          - platform
        ssh_access: true
        ssh_authorized_keys:
          - pubkey: "ssh-ed25519 AAAA... operator@example"
            state: present

    access_management_root_account_manage: true
    access_management_root_account_password_lock: true

  roles:
    - role: kazikb.linux.access_management
```

Run only this role's tagged tasks with:

```bash
ansible-playbook site.yml --tags access_management
```

License
-------

MIT

Author Information
------------------

Kazimierz Biskup [GitHub](https://github.com/kazikb/ansible-collection-linux)
