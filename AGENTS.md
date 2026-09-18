# Repository Guidelines

## Scope and Structure

- The repository root is the `kazikb.linux` collection. `galaxy.yml` declares
  collection identity and dependencies; `meta/runtime.yml` declares Core
  compatibility. Keep `requirements.yml` aligned with collection dependencies.
- Package the roles and both operational playbooks. Exclude private data,
  development compositions, controller configuration, caches, and generated
  artifacts using `build_ignore`. Record releases in `CHANGELOG.md` and match
  Git release tags to the version in `galaxy.yml`.
- `roles/` contains four reusable roles: `system_baseline`,
  `access_management`, `host_hardening`, and `docker_engine`. Follow the
  standard role layout: public defaults in `defaults/`, fixed implementation
  values in `vars/`, behavior in `tasks/`, notifications in `handlers/`,
  templates in `templates/`, and contracts in `meta/` and `README.md`.
- `playbooks/bootstrap_install_sudo.yml` installs sudo using su and adds the
  non-root Ansible connection user to the sudo or wheel group.
- `playbooks/install_updates.yml` installs system updates, removes unused
  packages, and can reboot managed hosts.
- `tests/integration-test.yml` composes the three base roles.
  `tests/integration-test-docker.yml` adds `docker_engine` before
  `access_management` so authoritative user group management can include the
  Docker group.
- `requirements.txt` contains controller runtime packages,
  `requirements-dev.txt` adds development tooling, and `requirements.yml`
  contains required Galaxy collections.
- `ansible.cfg` defines repository-local role and collection paths, disables
  injected fact variables, keeps host key checking enabled, and treats unmatched
  inventory host patterns as errors.
- Private inventories and host variables belong in ignored `tests/inventory/` or
  `local/` paths and must be passed explicitly with `-i`.
- Do not commit `.venv/`, `collections/`, `tests/inventory/`, `local/`, secrets, or
  generated files. Behavioral runs require an external inventory and an
  explicitly authorized disposable host; there is no tracked automated host
  test runner.

## Platform and Execution Contracts

- Roles require Ansible Core 2.18.12 or newer, gathered facts, systemd, and root
  privileges on managed hosts. Ubuntu 26.04 with its default Python 3.14 requires
  Ansible Core 2.20 or newer. Role metadata declares Debian 13, Ubuntu 24.04
  and 26.04, and Enterprise Linux 10. Report only the distributions and
  versions actually exercised for a change; do not infer behavioral verification
  from metadata.
- Every role's top-level task block is tagged with its role name. Preserve these
  tags and the role order in the integration compositions.
- `access_management` treats supplementary groups as authoritative. Preserve
  group-before-user ordering, delayed group removal, reserved-group ownership,
  and safeguards for root and the current Ansible connection user.
- `host_hardening` independently gates SSH, firewall, sysctl, and kernel module
  management. Validate firewall definitions whenever firewall management is
  enabled, but discover SSH listeners and enforce SSH firewall access only when
  the managed firewall is also enabled.
- `access_management` and `host_hardening` share identically named SSH validate
  and reload handlers. Keep their names and ordering aligned so a handler flush
  validates and reloads SSH only once.
- `system_baseline` owns a fixed package baseline. Mail relay and automatic
  updates remain opt-in. Chrony or systemd-timesyncd must already be installed;
  retain runtime detection of the available service.
- `docker_engine` owns Docker CE installation and `/etc/docker/daemon.json`.
  The target must reach Docker's official package repository. Validate rendered
  daemon configuration before replacement and restart Docker through its handler
  only when the file changes.
- `playbooks/install_updates.yml` is intentionally mutating and may reboot.
  Keep host targeting explicit. Enterprise Linux reboot detection requires
  `dnf-plugins-core`, which `system_baseline` installs.
- `playbooks/bootstrap_install_sudo.yml` targets fresh hosts without sudo.
  It requires Python 3 at `/usr/bin/python3`, a non-root connection user with
  su access to root, and the default sudo policy for the sudo or wheel group.
  Preserve `append: true` so existing group memberships remain intact. Keep
  host targeting explicit and distinguish the SSH password (`-k`) from the
  root password used by su (`-K`).

## Development Environment

Use Ansible from the repository's Python virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -r requirements-dev.txt
ansible-galaxy collection install -r requirements.yml
```

Development compositions use short role names and the repository's `roles_path`
to run source roles directly. Operational playbooks can also run by file path.
Consumers use fully qualified `kazikb.linux` role and playbook names. Running
source files does not require installing this collection.

To run playbooks by collection name locally, install or refresh the collection
from the repository root after installing its dependencies:

```bash
ansible-galaxy collection install . -p ./collections --force --no-deps
```

Repeat after source changes; the installed copy does not update automatically.

## Validation

Run repository-wide static validation when shared contracts change. For a small
role-only change, linting the affected role is sufficient in addition to the
relevant syntax checks.

```bash
ansible-lint --offline roles playbooks tests/*.yml galaxy.yml meta/runtime.yml
ansible-playbook playbooks/bootstrap_install_sudo.yml \
  -i localhost, --syntax-check
ansible-playbook playbooks/install_updates.yml \
  -i localhost, --syntax-check
ansible-playbook tests/integration-test.yml \
  -i localhost, --syntax-check
ansible-playbook tests/integration-test-docker.yml \
  -i localhost, --syntax-check
```

For packaging changes or releases, check archive contents and installation from
outside the repository using fully qualified role and playbook names. Private
and development-only files must be absent. Routine role edits do not require a
collection build or reinstall.

Static validation does not establish host behavior. For behavior changes, when
an explicitly authorized disposable host is available, use a private inventory
and run the relevant playbook with `--check --diff --limit <host>` where check
mode is meaningful, followed by a live run. Otherwise, report that behavioral
validation was not performed. Report the exact distributions and versions used
for every behavioral test.

## Ansible Conventions

- Use two-space YAML indentation, `---` document starts, `true`/`false`, fully
  qualified module names, and action-oriented task names.
- Prefix public variables with the role name. Keep overridable values in
  `defaults/` and fixed implementation values in `vars/`.
- Keep public defaults, `meta/argument_specs.yml`, and the role README aligned.
  Update `meta/main.yml` when the role description, Ansible requirement, license,
  or platform matrix changes.
- Use `ansible_facts[...]` instead of injected `ansible_*` fact variables, and
  convert boolean inputs with `| bool` in conditions and assertions.
- Treat inventory and role variables as untrusted configuration. Assert invalid
  or lockout-prone values and validate generated service configuration before
  notifying reload or restart handlers.
- Keep operational playbook requirements, usage examples, and safety notes in
  comments at the beginning of the playbook. The main `README.md` lists roles
  with links to their READMEs and playbooks with links to their source files.

## Changes and Security

- Make surgical changes, preserve unrelated work, and avoid new abstractions
  unless they solve a demonstrated problem.
- Update role documentation when behavior, public variables, requirements, or
  compatibility changes.
- Never commit passwords, private keys, tokens, decrypted variable files, or
  private inventory data. Use Ansible Vault for secrets.
- Prefer commits in the form `type(scope): imperative summary`.
