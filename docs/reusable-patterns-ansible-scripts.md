# Reusable Patterns: Ansible & Scripts

Reference patterns extracted from `mcp/exarp-go`, `trading/ib_box_spread_full_universal`, `david_cv`, and `davidblog`. Use these when adding new projects or roles.

---

## Ansible

### 1. Directory layout

```
ansible/
├── ansible.cfg
├── requirements.yml
├── run-dev-setup.sh           # One-entry script (see below)
├── inventories/
│   ├── development/
│   │   ├── hosts
│   │   └── group_vars/all.yml
│   └── production/
│       ├── hosts
│       └── group_vars/all.yml
├── playbooks/
│   ├── development.yml
│   ├── development-user.yml   # Optional: no sudo when tools already present
│   └── production.yml
└── roles/
    └── <role_name>/
        ├── tasks/main.yml
        ├── defaults/main.yml
        ├── vars/
        │   ├── Darwin.yml
        │   ├── Debian.yml
        │   └── RedHat.yml
        └── meta/main.yml
```

### 2. OS-specific role vars

- In each role, **load OS vars first** in `tasks/main.yml`:
  ```yaml
  - name: Load OS-specific variables
    ansible.builtin.include_vars: "{{ ansible_os_family }}.yml"
    tags: [role_name]
  ```
- Provide one file per family: `vars/Darwin.yml`, `vars/Debian.yml`, `vars/RedHat.yml`.
- Use **private vars** (prefix `_`) for package lists and paths so they don’t leak:
  ```yaml
  # vars/Darwin.yml
  _pkg_manager: homebrew
  _base_packages: [curl, wget, git, make]
  _ca_bundle_path: /etc/ssl/cert.pem
  ```
- Use those in tasks: `"{{ _base_packages }}"`, `when: ansible_os_family == "Debian"`, `when: ansible_system == "Darwin"`.

### 3. Optional features and tags

- In **inventory** `group_vars/all.yml`:
  ```yaml
  install_linters: true
  install_ollama: false
  install_redis: false
  ```
- In **playbook**: include role conditionally and tag it:
  ```yaml
  - role: linters
    when: install_linters | default(false)
    tags: [linters, optional]
  ```
- Use **tags** for partial runs: `--tags golang,linters`, `--skip-tags optional`.

### 4. Playbook conventions

- `hosts: development` (or `production`); `gather_facts: true`; `become: true` when installing system packages.
- One “summary” debug task at the end listing what was installed/enabled.
- Prefer `ansible.builtin.*` and `community.general.*` (e.g. `community.general.homebrew`) and list them in `requirements.yml`.

### 5. ansible.cfg (reusable snippet)

```ini
[defaults]
inventory = ./inventories/development/hosts
roles_path = ./roles
host_key_checking = False
retry_files_enabled = False
gathering = smart
fact_caching = memory
inject_facts_as_vars = True
deprecation_warnings = False

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
pipelining = True
```

### 6. run-dev-setup.sh pattern

- **Location**: `ansible/run-dev-setup.sh` (run from repo root or from `ansible/`).
- **Steps**:
  1. `cd` to ansible dir (or use `REPO_ROOT`/`ANSIBLE_DIR` when script lives under repo root).
  2. **macOS**: set `SSL_CERT_FILE` and `REQUESTS_CA_BUNDLE` to system CA path (`/etc/ssl/cert.pem` or Homebrew OpenSSL) to avoid Python/Ansible SSL errors.
  3. Check `ansible-playbook` exists; print install hint (brew / apt / dnf / pip) and exit 1 if missing.
  4. Install Galaxy deps: `ansible-galaxy collection install -r requirements.yml --force-with-deps` (optional; continue on failure).
  5. Syntax check: `ansible-playbook --syntax-check -i inventories/development playbooks/development.yml`; exit 1 on failure.
  6. List tasks: `ansible-playbook --list-tasks -i ...` (optional; limit lines).
  7. Run playbook: either `development.yml` with `--ask-become-pass`, or (if exarp-go style) detect “user-only” (Go + build tools present) and run `development-user.yml` without sudo.
- Use `set -e`; echo clear steps with ✅/❌.

---

## Scripts

### 1. Shared includes (scripts/include/)

- **Guard** so the file is only sourced once:
  ```bash
  if [[ -n "${__MY_INCLUDE_GUARD:-}" ]]; then return 0 2>/dev/null || :; fi
  __MY_INCLUDE_GUARD=1
  ```
- **Usage**: from a script in `scripts/`:
  ```bash
  SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
  . "${SCRIPT_DIR}/include/ensure_third_party.sh"
  ensure_third_party
  ```
- Keep includes **sourced**, not executed (no top-level side effects unless intended).

### 2. OS detection

- Use `uname -s` for behavior:
  ```bash
  if [[ "$(uname -s)" == "Darwin" ]]; then
    export PARALLEL=$(sysctl -n hw.ncpu 2>/dev/null || echo 4)
  else
    export PARALLEL=$(nproc 2>/dev/null || echo 4)
  fi
  ```
- Use the same pattern for package manager (Homebrew vs apt/dnf), paths, and optional features (e.g. DMG).

### 3. Repo root

- Prefer one of:
  ```bash
  REPO_ROOT="${REPO_ROOT:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"
  # or
  cd "$(dirname "$0")"
  # or from scripts/:
  SCRIPT_DIR="$(cd "$(dirname "${BASH_S0}")" && pwd)"
  REPO_ROOT="$(cd "$SCRIPT_DIR/.." && pwd)"
  ```

### 4. Script categories and maintenance

- **Categories** (from your audit): Build, Test, Coverage/Analysis, Documentation, TODO/Task, Installation, Setup, Service start/stop, Utilities.
- **Naming**: `verb_thing.sh` or `verb_thing.py`; primary script per task (e.g. `build_fast.sh` as primary build).
- **Consolidation**: Prefer one script with flags (e.g. `generate_coverage.sh --cpp` / `--python`) over many one-off scripts.
- **Deprecation**: Move to `scripts/deprecated/` and document replacement in an audit doc (e.g. `SCRIPTS_AUDIT.md`) with a migration table.

### 5. Service start/stop

- Consistent naming: `start_<service>.sh`, `stop_<service>.sh`; optional `service_manager.sh` for a single entry point.
- Reuse the same pattern for env (e.g. project root, venv, ports) so Ansible and scripts stay aligned.

### 6. Calling Ansible from scripts

- Prefer **wrapper script** (e.g. `fetch_third_party.sh`) that checks `ansible-playbook` and runs the right playbook with the right inventory.
- If deps are missing, print a one-line hint: “Install Ansible and run: ./scripts/fetch_third_party.sh”.

---

## Cross-project reuse

| From              | Reuse in new project |
|-------------------|----------------------|
| `mcp/exarp-go/ansible` | Copy `ansible/` layout, `run-dev-setup.sh`, and `ansible.cfg`; trim roles to what you need (common, golang, linters, etc.). |
| `roles/common`    | OS vars + base packages + optional (e.g. gh, git config); duplicate and rename role if project-specific. |
| `scripts/include/` | Copy `set_parallel_level.sh`, `ensure_third_party.sh`-style guard and sourcing pattern. |
| `run-dev-setup.sh` | Same flow (SSL on macOS, galaxy, syntax-check, list-tasks, run playbook); adjust playbook/inventory names. |

---

*Last updated: 2026-03-04*
