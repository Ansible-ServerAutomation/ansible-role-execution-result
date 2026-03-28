# ansible-role-execution-result

Ansible role that tracks and logs the execution result of tasks in other roles.  
Designed to be invoked from `block/rescue` (or `always`) sections to capture the outcome — return code, a descriptive message, and the name of the task that failed.

Results are automatically published to the **AWX/Tower job Artifacts** section via `ansible.builtin.set_stats`, making them visible in the *Artifacts* tab of every job run without any extra configuration.

---

## Role Structure

```
ansible-role-execution-result/
├── defaults/
│   └── main.yml          # Default variable values (overridable by callers)
├── examples/
│   └── example_playbook.yml  # Sample usage in a block/rescue context
├── handlers/
│   └── main.yml
├── meta/
│   └── main.yml          # Galaxy metadata and dependencies
├── tasks/
│   └── main.yml          # Core logic: validate, record, log, emit debug
├── templates/
│   └── execution_result_entry.j2  # Log line template
├── vars/
│   └── main.yml          # Internal role constants
└── README.md
```

---

## Input Variables

These variables must be passed by the calling role or task:

| Variable | Required | Default | Description |
|---|---|---|---|
| `execution_result_return_code` | yes | `0` | Return/exit code of the task being tracked |
| `execution_result_message` | yes | `""` | Human-readable outcome or error message |
| `execution_result_failed_task` | no | `""` | Name of the task that failed (for audit trail) |
| `execution_result_warnings` | no | `[]` | List of warnings returned by the task (from `task_result.warnings`) |
### AWX/Tower Metadata Variables (Optional)

These variables are **automatically populated** from AWX/Tower environment variables if not explicitly provided:

| Variable | Auto-populated from | Description |
|---|---|---|
| `execution_result_project_name` | `AWX_PROJECT_NAME` or `TOWER_PROJECT_NAME` | AWX/Tower project name |
| `execution_result_organization` | `TOWER_ORGANIZATION` | AWX/Tower organization name |
| `execution_result_scm_revision` | `AWX_PROJECT_REVISION` | Git commit SHA from source control |
| `execution_result_scm_branch` | `AWX_PROJECT_SCM_BRANCH` | Git branch name from source control |
| `execution_result_execution_environment` | `AWX_EXECUTION_ENVIRONMENT` | Execution environment name |
| `execution_result_job_template` | `AWX_JOB_TEMPLATE_NAME` or `TOWER_JOB_TEMPLATE_NAME` | Job template name |
| `execution_result_playbook_name` | n/a | Playbook filename (must be set manually if needed) |
---

## Configuration Variables

Override these in your playbook or inventory to control role behaviour:

| Variable | Default | Description |
|---|---|---|
| `execution_result_log_enabled` | `true` | Write results to a log file on the target host |
| `execution_result_log_file` | Linux: `/var/log/ansible_execution_results.log` / Windows: `C:\ProgramData\Ansible\ansible_execution_results.log` | Path to the log file (auto-detected by `ansible_os_family`) |
| `execution_result_log_dir` | Linux: `/var/log` / Windows: `C:\ProgramData\Ansible` | Directory for the log file (created if absent, auto-detected by `ansible_os_family`) |
| `execution_result_accumulate` | `true` | Accumulate results in an Ansible fact across role calls |
| `execution_result_results_fact` | `execution_results` | Name of the fact that holds accumulated results |
| `execution_result_fail_on_error` | `false` | Fail the play when `return_code` is non-zero |
| `execution_result_set_stats_enabled` | `true` | Publish results to AWX/Tower job Artifacts via `set_stats` |
| `execution_result_set_stats_per_host` | `false` | Store stats per-host in addition to the aggregated artifact |

---

## Usage

### Basic block/rescue pattern

```yaml
- name: Perform a critical operation
  block:
    - name: Run the primary task
      ansible.builtin.command: /usr/bin/my_script.sh
      register: primary_task_result

  rescue:
    - name: Record execution failure
      ansible.builtin.include_role:
        name: execution_result
      vars:
        execution_result_return_code: "{{ primary_task_result.rc | default(1) }}"
        execution_result_message: "{{ primary_task_result.stderr | default('Unknown error') }}"
        execution_result_failed_task: "Run the primary task"
        execution_result_warnings: "{{ primary_task_result.warnings | default([]) }}"
```

**Important:** To capture warnings, you **must** use `register:` on the task you want to track, then pass `task_result.warnings` to `execution_result_warnings`. Warnings are only available in registered task results.

### Accessing accumulated results later

After one or more calls to this role (with `execution_result_accumulate: true`), the fact `execution_results` holds a list of all recorded entries:

```yaml
- name: Print all recorded execution results
  ansible.builtin.debug:
    var: execution_results
```

Each entry in the list has the following fields:

```yaml
- timestamp:              "2026-03-26T10:00:00Z"
  host_os:                "RedHat"                 # OS family (RedHat, Debian, Windows, etc.)
  project:                "My Ansible Project"     # AWX/Tower project name
  organization:           "IT Operations"          # AWX/Tower organization
  job_template:           "Deploy Application"     # AWX/Tower job template
  playbook:               "N/A"                    # Playbook filename (if set)
  scm_branch:             "development"            # Git branch
  scm_revision:           "a1b2c3d4e5f6..."        # Git commit SHA
  execution_environment:  "Default EE"             # Execution environment
  status:                 "FAILURE"                # SUCCESS or FAILURE
  return_code:            1
  message:                "Script exited with code 1"
  failed_task:            "Run the primary task"
  failed_task_module:     "ansible.builtin.command" # FQCN of the module that failed
  warnings:
    - "Platform linux on host Ubuntu is using the discovered Python interpreter..."
```

### AWX/Tower job Artifacts

When `execution_result_set_stats_enabled: true` (the default) and `execution_result_accumulate: true`, the role calls `ansible.builtin.set_stats` after every invocation. This populates the **Artifacts** section of the AWX/Tower job template with:

| Artifact key | Description |
|---|---|
| `execution_results` | Full list of accumulated result entries from all invocations. Each entry contains: `timestamp`, `host_os`, `project`, `organization`, `job_template`, `playbook`, `scm_branch`, `scm_revision`, `execution_environment`, `status`, `return_code`, `stdout`, `stderr`, `message`, `exception`, `warnings`, `failed_task`, `failed_task_module` |

Results from all hosts are aggregated into a single artifact (controlled by `execution_result_set_stats_per_host`).  
The artifacts are visible in the *Artifacts* tab of each job run and can be consumed by downstream workflow job templates via `{{ artifacts['execution_results'] }}`.

---

## Limitations

### Warning Capture

The role captures **task-level warnings** that appear in the `warnings` key of registered task results. Examples include:

- Python interpreter discovery warnings
- Module-specific warnings (deprecated parameters, etc.)
- Warnings returned by modules like `apt`, `yum`, `win_package`, etc.

**NOT captured:**
- Play-level warnings shown before/after play execution (e.g., `[DEPRECATION WARNING]: ANSIBLE_COLLECTIONS_PATHS...`)
- Ansible configuration warnings
- Inventory warnings

These play-level warnings are emitted by Ansible core and are not part of task results. To capture them, you would need a custom callback plugin.

---

## Requirements

- Ansible >= 2.12
- `gather_facts: true` must be enabled (the role uses `ansible_os_family` and `ansible_date_time`)
- For Windows targets: the `ansible.windows` collection must be installed (`ansible-galaxy collection install ansible.windows`)

---

## Log File Format

When `execution_result_log_enabled: true`, each entry appended to the log file looks like:

```
[2026-03-26T10:00:00Z] HOST=webserver01 STATUS=FAILURE RC=1 TASK="Run the primary task" MSG="Script exited with code 1"
```

---

## License

MIT
