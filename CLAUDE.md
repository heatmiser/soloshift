# soloshift — Claude Code Project Guide

## Project Overview

soloshift automates the deployment of a single-node OpenShift (OCP) cluster on a KVM hypervisor using Ansible. It provisions KVM VMs, configures DNS (BIND), DHCP, a utility node, NFS storage, and the OpenShift bootstrap sequence — all on a single bare-metal or VM host.

Current target: OCP 4.10 on RHEL 8. Modernization goal: bring the project up to current OCP and Ansible practices.

## Environment

This project is developed inside a Podman container. The Python virtualenv for Ansible operations is at:

```
/home/claude/workspace/ansible2.16.17/bin/activate
```

Always activate it before running `ansible-playbook` or `ansible-lint`:

```bash
source /home/claude/workspace/ansible2.16.17/bin/activate
```

## Project Layout

```
soloshift/
├── ansible.cfg
├── cluster-up.yaml          # Top-level entry playbook
├── requirements.yaml        # Galaxy role dependencies
├── inventory/
│   ├── group_vars/all/      # Global variables (default_vars.yaml)
│   ├── hypervisor           # Hypervisor host inventory
│   └── localhost            # Localhost inventory
├── playbooks/               # Numbered playbooks executed in order
│   ├── files/
│   ├── includes/
│   └── templates/
└── roles/
    ├── ocp4-solo-hypervisor/
    ├── ocp4-solo-vmprovision/
    ├── ocp4-solo-utilnode/
    ├── ocp4-solo-util-bind-cntr/
    ├── ocp4-solo-util-dhcp-cntr/
    ├── ocp4-solo-post-install/
    └── ...
```

## Coding Standards

Standards are based on [Red Hat CoP Automation Good Practices](https://redhat-cop.github.io/automation-good-practices/).

### Module Naming — FQCN Required

Always use fully qualified collection names. Prefer the `ansible.builtin` prefix over short names.

```yaml
# Correct
- ansible.builtin.template:
- ansible.builtin.copy:
- ansible.builtin.service:
- ansible.builtin.debug:
- ansible.posix.firewalld:
- community.general.nmcli:

# Wrong
- template:
- copy:
- service:
- debug:
```

### Variable Management

- **Prefer `vars_files` over inline `vars:`** in playbooks and roles
- Role defaults go in `defaults/main.yml` — these are overridable by inventory
- Role internal/computed variables go in `vars/main.yml` — higher precedence, not meant to be overridden
- Prefix role variables with the role name: `ocp4_solo_hypervisor_packages`, not `packages`
- Prefix internal-only role variables with double underscores: `__ocp4_solo_hypervisor_temp`
- Use `snake_case` for all variable names

### YAML Style

- Indent with **2 spaces** — no tabs
- File extension: `.yaml` (this project uses `.yaml` consistently — maintain it)
- Booleans: use `true` / `false` — never `yes`/`no`, `True`/`False`, or `"yes"`/`"no"`
- Use double quotes for YAML strings; single quotes inside Jinja2 expressions
- Omit quotes for Ansible keywords (`present`, `absent`, `started`, etc.)
- Keep lines under 160 characters; use YAML block scalar `>-` for long expressions
- Break multi-condition `when:` into a list (implicit `and`)

```yaml
when:
  - ansible_facts['distribution'] == 'RedHat'
  - ansible_facts['distribution_major_version'] | int >= 8
```

### Task Naming

- **Name every task, block, and play** — no unnamed tasks
- Use imperative mood: "Ensure firewalld is running", not "firewalld running"
- Prefix tasks in included sub-files: `rolename | Task description`
- Use bracket notation for facts: `ansible_facts['distribution']` not `ansible_distribution`

### Comments in YAML

Comments are acceptable and encouraged where they add context: explaining *why* a value is set, documenting non-obvious conditionals, or noting known workarounds.

### Roles

- Keep roles idempotent — applying twice must produce no changes on the second run
- Use `changed_when:` with `command`/`shell` tasks to accurately report changes
- Add `check_mode: true` support where feasible
- Use `include_tasks` for platform-specific task files; use `import_tasks` for static inclusion
- Use `lookup('ansible.builtin.first_found', ...)` for platform-specific variable loading

### Handlers

- Name handlers descriptively
- Notify handlers by name from tasks; avoid triggering handlers from other handlers when possible

### Tags

- Tag tasks with role name or meaningful purpose
- Ensure tagged subsets are independently runnable without breaking state

### Playbooks

- Keep playbooks thin — delegate logic to roles
- Use `ansible.builtin.import_playbook` (FQCN) in master playbooks
- Use `ansible.builtin.import_role` for static role inclusion, `ansible.builtin.include_role` for dynamic

## Pre-completion Checklist

**Always run ansible-lint before finishing any task:**

```bash
source /home/claude/workspace/ansible2.16.17/bin/activate
ansible-lint
```

Fix all reported issues before considering a task complete. Lint profile: `production` is the target standard.

## Current Modernization Goals

1. Update OCP target version from 4.10 to current stable (4.16+)
2. Migrate all modules to FQCN with `ansible.builtin` prefix
3. Replace `True`/`False`/`yes`/`no` booleans with `true`/`false`
4. Convert inline `vars:` blocks to `vars_files:` where appropriate
5. Add missing task names
6. Update RHEL base image references to current supported versions
7. Improve idempotency across roles
8. Add `changed_when:` to all `command`/`shell` tasks
