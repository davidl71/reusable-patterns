# References & research

Relevant official docs and general research for the patterns in this repo. See [reusable-patterns-ansible-scripts.md](reusable-patterns-ansible-scripts.md) for how they map to our patterns.

## Ansible

| Source | What it covers |
|--------|----------------|
| [General tips — Ansible Documentation](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html) | Keep it simple, version control, FQCN, naming tasks, explicit `state`, comments; inventory grouping and separate prod/staging; OS handling via `group_by` or `include_vars`; `--syntax-check` in staging. |
| [Roles — Ansible Documentation](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html) | Role directory structure; `defaults` vs `vars`; OS-specific tasks with `import_tasks` + `when` or `include_vars`; tags; role dependencies. |
| [Porting guides](https://docs.ansible.com/ansible/latest/porting_guides/porting_guides.html) | Changes between versions (e.g. ansible-core 2.19/Ansible 12 templating); migration from bare variables and deprecated directives. |

## Shell scripts

| Source | What it covers |
|--------|----------------|
| [Google Shell Style Guide](https://google.github.io/styleguide/shellguide.html) | Bash-only executables; STDOUT vs STDERR; file/function comments; 2-space indent, 80-char lines; `$(...)`, `[[ ]]`, quoting, arrays; ShellCheck; avoid `eval`; when to switch to a structured language. |
| [ShellCheck](https://www.shellcheck.net/) | Static analysis for common shell bugs and style issues; recommended for all scripts. |

## Quick takeaways

- **Ansible:** Use FQCN, name tasks, set `state` explicitly, separate inventories per environment, load OS-specific vars with `include_vars` (matches our `vars/Darwin.yml` pattern).
- **Scripts:** Use `[[ ]]` and quoted variables; send errors to STDERR; run ShellCheck; keep scripts small or move logic to another language.
