# Developer Activity Auditor

An Ansible playbook that scans developer project directories on CentOS 7 servers, identifies recently modified files within a configurable time window, renders a structured activity report, and fetches it back to the control node.

---

## How to Run the Playbook

### Prerequisites

- Ansible 2.12+ on the control node
- Python 3 on every target host (the playbook asserts this at runtime)
- SSH key-based access configured; the inventory expects the key at `~/.ssh/ansible_id_ed25519`
- The `ansible` user account exists on each target host

### Vault setup (required before first run)

The files `group_vars/all/secrets.yml` and `group_vars/dev_servers/group_secrets.yml` contain sudo credentials and must be encrypted before use.

1. Replace the placeholder values in each file:
   - `group_vars/all/secrets.yml` → set `vault_ansible_become_pass`
   - `group_vars/dev_servers/group_secrets.yml` → set `vault_become_pass`

2. Encrypt them with Ansible Vault:
   ```bash
   ansible-vault encrypt group_vars/all/secrets.yml
   ansible-vault encrypt group_vars/dev_servers/group_secrets.yml
   ```

3. Store your vault password in `~/.vault_pass` (mode `600`) and uncomment the `vault_password_file` line in `ansible.cfg`.

### Running

```bash
# Full audit run
ansible-playbook main.yml

# Pre-flight checks only (OS + Python assertions)
ansible-playbook main.yml --tags preflight

# Re-render and fetch the report without re-running discovery
ansible-playbook main.yml --tags report

# Dry run (check mode)
ansible-playbook main.yml --check

# Prompt for vault password at runtime instead of using a file
ansible-playbook main.yml --ask-vault-pass
```

Reports are fetched to `./reports/<hostname>_dev_activity_<date>.txt` on the control node. The remote copy lives at `/var/log/dev_activity_report.txt`.

---

## Assumptions

- **CentOS 7 only.** The `preflight` tasks assert `ansible_distribution == "CentOS"` and major version 7. Other RHEL-family distros (RHEL 7, Rocky, AlmaLinux) would pass the spirit of the check but are not tested — the assertion would need relaxing for them.
- **Python 3 is available at `/usr/bin/python3`.** The `ansible.cfg` sets `interpreter_python = /usr/bin/python3` explicitly because CentOS 7 ships Python 2 by default. If Python 3 is installed at a different path, update the inventory or `ansible.cfg`.
- **`/opt/dev/projects` follows a flat per-developer layout.** The scanner discovers immediate subdirectories of `projects_base_dir` and treats each one as a developer's workspace. Nested structures are scanned for file modifications but are not separately attributed.
- **`audit_target_users` is a simple basename match.** Users are filtered by matching directory basenames with a regex join. This means a username that is a substring of another (e.g., `bob` vs `bobby`) could produce false matches; the user list should contain sufficiently unique names.
- **The `ansible` user has passwordless sudo for specific commands, or a vault-supplied password is adequate.** `become: true` is used for file system access, report writes, group creation, and logrotate deployment.
- **The `audit` group will be created if missing.** The playbook creates it as a system group; no existing group membership is assumed.
- **Network reachability.** Hosts are reachable at the IPs listed in the inventory over SSH with the specified key. SSH host key checking is enabled (`StrictHostKeyChecking=yes`); target host keys must already be in `~/.ssh/known_hosts`.

---

## What I Would Add or Change for Production

**Security**

- Replace the flat vault password file with a proper secrets manager integration (HashiCorp Vault, AWS Secrets Manager, or 1Password) using a lookup plugin, so credentials are never stored on the control node filesystem.
- Add GPG report encryption end-to-end: the `vault_report_encryption_key` field is already scaffolded in `secrets.yml` but is not wired to any task. A post-report task could encrypt the remote file before fetch and decrypt it on the control node.
- Audit the `ansible` user's sudo rules to follow least privilege — only the specific commands the playbook needs (`find`, `chown`, `cp`, etc.) should be in the sudoers file, not a blanket `NOPASSWD: ALL`.

**Portability**

- Remove the hard `CentOS 7` assertion or replace it with a family-level check (`ansible_os_family == "RedHat"`) with a version floor. This would allow running on RHEL 8/9, Rocky Linux, or AlmaLinux without code changes.
- Parameterise `ansible_python_interpreter` via `group_vars` rather than `ansible.cfg` so multi-distro inventories can specify different paths per group.

**Observability and alerting**

- Add a notification task (email, Slack, PagerDuty) that fires when a developer directory has been inactive beyond a configurable threshold, rather than requiring someone to read the report manually.
- Ship the audit events log (`/var/log/dev_audit_events.log`) to a central log aggregator (ELK, Splunk, CloudWatch Logs) for long-term retention and search.

**Scalability**

- For inventories with more than a handful of hosts, replace the sequential `ansible.builtin.find` loop with an async pattern (`async` / `poll: 0` + `async_status`) so discovery and file collection run in parallel across hosts.
- Consider a dynamic inventory script or plugin (e.g., pulling from a CMDB or cloud API) instead of a static `hosts.yml`, so new servers are picked up automatically.

**Testing and CI**

- Add a Molecule test suite with a Docker driver so the role can be tested in CI without real target hosts. At minimum: a `converge` scenario that runs against a CentOS 7 container with a synthetic project tree, and a `verify` step that asserts the report file exists and contains expected content.
- Lint with `ansible-lint` and `yamllint` in CI to catch style drift early.

**Report format**

- The current report is a plain-text file. For production, rendering to JSON or CSV in addition to the human-readable text format would make it straightforward to ingest into dashboards or ticket systems without screen-scraping.
