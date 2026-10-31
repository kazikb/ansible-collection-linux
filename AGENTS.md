# Repository Guidelines

## Repository Layout

- The repository root is the `kazikb.linux` collection. `galaxy.yml` defines
  identity, version, dependencies, and packaging; `meta/runtime.yml` defines
  Ansible Core compatibility.
- `roles/` contains `system_baseline`, `access_management`, `host_hardening`,
  and `docker_engine`. Read the affected role's README and metadata before
  changing its behavior or public variables.
- `playbooks/` contains the operational playbooks `bootstrap_install_sudo.yml`
  and `install_updates.yml`. Their opening comments define requirements and
  usage.
- `tests/integration-test.yml` composes the base roles. The Docker composition
  adds `docker_engine` before `access_management`. Preserve composition order
  and role tags.
- `requirements.txt` contains controller dependencies; `requirements-dev.txt`
  adds tooling. Keep `requirements.yml` aligned with `galaxy.yml` dependencies.

## Development and Validation

Use Ansible from the repository's virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -r requirements-dev.txt
ansible-galaxy collection install -r requirements.yml
```

Development compositions run source roles using `ansible.cfg`. Consumers use
fully qualified `kazikb.linux` names. To test collection-name resolution locally,
refresh the installed copy after source changes:

```bash
ansible-galaxy collection install . -p ./collections --force --no-deps
```

For role-only changes, lint the affected role and run relevant syntax checks.
Run the full static checks when shared contracts change:

```bash
ansible-lint --offline roles playbooks tests/*.yml galaxy.yml meta/runtime.yml
ansible-playbook playbooks/bootstrap_install_sudo.yml -i localhost, --syntax-check
ansible-playbook playbooks/install_updates.yml -i localhost, --syntax-check
ansible-playbook tests/integration-test.yml -i localhost, --syntax-check
ansible-playbook tests/integration-test-docker.yml -i localhost, --syntax-check
```

Behavioral tests require a private inventory and an explicitly authorized
disposable host. There is no tracked automated host test runner. Use
`--check --diff --limit <host>` where meaningful, followed by a live run.
Report the exact distributions and versions exercised, or state that behavioral
validation was not performed. Static checks and platform metadata do not
establish host behavior.

## Implementation Conventions

- Make surgical changes and preserve unrelated work. Avoid new abstractions
  unless they solve a demonstrated problem.
- Use two-space YAML indentation, `---` document starts, `true`/`false`, fully
  qualified module names, and action-oriented task names.
- Follow standard role layout. Put public variables in `defaults/`, fixed
  values in `vars/`, behavior in `tasks/`, notifications in `handlers/`, and
  templates in `templates/`. Prefix public variables with the role name.
- Keep defaults, `meta/argument_specs.yml`, and the role README aligned. Update
  `meta/main.yml` when requirements, platforms, description, or license change.
- Use `ansible_facts[...]`; injected fact variables are disabled. Convert
  boolean inputs with `| bool`. Keep host key checking enabled and unmatched
  inventory patterns fatal.
- Treat variables as untrusted configuration. Reject invalid or lockout-prone
  values and validate rendered service configuration before replacement and
  reload or restart.
- Keep operational playbook requirements and examples in opening comments.
  Keep the main README's role and playbook links current. Keep documentation
  concise; use changelogs for release summaries, not implementation details.
- Prefer commits in the form `type(scope): imperative summary`.

## Behavior Contracts

- Roles require gathered facts, systemd, root privileges, and Ansible Core
  2.18.12 or newer. Ubuntu 26.04 with default Python 3.14 requires Core 2.20 or
  newer. Supported platforms are declared in role metadata.
- `access_management` owns supplementary group membership authoritatively.
  Preserve group-before-user ordering, delayed group removal, reserved-group
  ownership, and safeguards for root and the Ansible connection user.
- `system_baseline` owns a fixed package baseline. Mail relay and automatic
  updates are opt-in. Preserve detection of an installed Chrony or
  systemd-timesyncd service.
- `docker_engine` owns Docker CE installation and `/etc/docker/daemon.json`.
  Targets must reach Docker's official repository. Validate daemon configuration
  before replacement and restart Docker only through its change handler.
- `host_hardening` independently gates SSH, firewall, sysctl, and kernel module
  management. Validate firewall definitions whenever managed; discover SSH
  listeners and enforce access only when the managed firewall is enabled.
  Require rules for current listeners and an explicitly requested managed SSH
  port. Apply firewall changes before SSH changes; immediate nftables reload
  is intentional.
- SSH port management owns only the role's `Port` directive. Preserve other
  SSH port configuration. For `null`, fallback ports are the caller's firewall
  responsibility. Connection migration also belongs to the caller; use the
  role README's dedicated migration example and defer reboots until reconnecting.
- On Enterprise Linux, the role owns the selected SSH SELinux port mapping,
  including existing local customizations. Manage policy even when SELinux is
  disabled. Remove the previous local mapping after SSH validation and reload,
  preserving distribution policy. On Ubuntu, reload systemd and restart
  `ssh.socket` only when it was active before the configuration change.
- `access_management` and `host_hardening` share SSH validation and reload
  handler names. Keep names and ordering aligned to validate and reload once
  per handler flush. Keep SELinux cleanup after SSH reload.
- `bootstrap_install_sudo.yml` requires a non-root connection user with su
  access to root and Python at `/usr/bin/python3`. Preserve `append: true` and
  the distinction between SSH (`-k`) and su (`-K`) passwords.
- `install_updates.yml` mutates hosts and may reboot. Keep targeting explicit;
  Enterprise Linux reboot detection requires `dnf-plugins-core` from the baseline.

## Private Data and Releases

- Keep private inventories and host variables in ignored `tests/inventory/` or
  `local/` paths and pass them explicitly with `-i`. Never commit secrets,
  private inventories, `.venv/`, `collections/`, or generated artifacts. Use
  Ansible Vault for secrets.
- For releases, update `galaxy.yml` and `CHANGELOG.md` together and use a
  matching `vX.Y.Z` Git tag. Release automation builds the collection and uses
  the matching changelog section for GitHub release notes.
- Package all roles and both operational playbooks. Exclude private data,
  development compositions, controller configuration, caches, and generated
  files through `build_ignore`.
- For packaging changes or releases, inspect the archive and test installation
  outside the repository with fully qualified role and playbook names. Verify
  excluded files are absent. Routine role edits do not require rebuilding or
  reinstalling the collection.
