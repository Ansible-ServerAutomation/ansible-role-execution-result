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
│   └── main.yml          # Core logic: normalize, build result, publish, emit debug
├── templates/
│   └── execution_result_entry.j2  # Log line template
├── vars/
│   └── main.yml          # Internal role constants
└── README.md
```

---

## Input Variables

These variables can be passed by the calling role or task. 

### Simplified Usage with Registered Variable (Recommended)

For easier integration, pass the entire registered variable from your task to the role:

| Variable | Required | Default | Description |
|---|---|---|---|
| `execution_result_registered_var` | no | `{}` | Registered variable from the previous task. The role will automatically extract all fields from this variable. |

**Example:**
```yaml
- name: Run a task
  ansible.builtin.command: /usr/bin/my_script.sh
  register: task_result

- name: Record execution result
  ansible.builtin.include_role:
    name: execution_result
  vars:
    execution_result_registered_var: "{{ task_result }}"
```

The role automatically extracts these fields from the registered variable:
- `rc` → return_code
- `msg` → message
- `stdout` → stdout
- `stderr` → stderr
- `exception` → exception
- `warnings` → warnings
- `task` → failed_task
- `action` → failed_task_module

### Individual Field Variables (Alternative)

You can also pass individual fields explicitly. When not provided, the role uses fallback logic to populate values:
1. First checks `execution_result_registered_var` (if provided)
2. Then checks Ansible magic variables (`ansible_failed_result`, `ansible_failed_task`)
3. Finally defaults to empty string

| Variable | Required | Default | Fallback Source | Description |
|---|---|---|---|---|
| `execution_result_return_code` | no | `""` | `execution_result_registered_var.rc` → `ansible_failed_result.rc` | Return/exit code of the task being tracked |
| `execution_result_message` | no | `""` | `execution_result_registered_var.msg` → `ansible_failed_result.msg` | Human-readable outcome or error message |
| `execution_result_stdout` | no | `""` | `execution_result_registered_var.stdout` → `ansible_failed_result.stdout` | Standard output from the task |
| `execution_result_stderr` | no | `""` | `execution_result_registered_var.stderr` → `ansible_failed_result.stderr` | Standard error from the task |
| `execution_result_exception` | no | `""` | `execution_result_registered_var.exception` → `ansible_failed_result.exception` | Exception traceback if available |
| `execution_result_failed_task` | no | `""` | `execution_result_registered_var.task` → `ansible_failed_task.name` | Name of the task that failed (for audit trail) |
| `execution_result_failed_task_module` | no | `""` | `execution_result_registered_var.action` → `ansible_failed_task.action` | Module name of the failed task |
| `execution_result_warnings` | no | `[]` | `execution_result_registered_var.warnings` | List of warnings returned by the task (from `task_result.warnings`) |
| `execution_result_os_distribution` | no | `""` | `ansible_distribution` | Operating system distribution (e.g., Ubuntu, CentOS) |
| `execution_result_os_version` | no | `""` | `ansible_distribution_version` | Operating system version (e.g., 20.04, 7.9) |
| `execution_result_os_family` | no | `""` | `ansible_os_family` | Operating system family (e.g., Debian, RedHat) |
| `execution_result_os_system` | no | `""` | `ansible_system` | System type (e.g., Linux, Windows) |
| `execution_result_os_architecture` | no | `""` | `ansible_architecture` | System architecture (e.g., x86_64, aarch64) |

**Note:** Individual variables take precedence over `execution_result_registered_var`. If you pass both, the individual field value will be used.

### AWX/Tower Metadata Capture

The role captures **AWX/Tower metadata** (project, organization, job template, SCM details, execution environment) using an **API-first approach when AWX/Tower API configuration is available**:

**When running in AWX/Tower:**
- If AWX/Tower API credentials/configuration are available, the role attempts to fetch metadata from the AWX/Tower REST API using OAuth Bearer token authentication
- `TOWER_JOB_ID` is used to identify the current job context for API lookups
- Falls back to environment variables if API configuration is unavailable or the API fetch fails

**When running locally/CI/CD:**
- Provide API configuration via extra variables or credentials
- Role will attempt API fetch if credentials are available

**Priority Order for AWX Metadata Fields:**
1. **API-fetched values** (primary source when API mode is enabled and configured)
2. **Environment variables** (fallback: `AWX_PROJECT_NAME`, `TOWER_ORGANIZATION`, etc.)
3. **'N/A'** (if neither API nor env vars available)

**API Configuration Variables:**

| Variable | Required | Default | Description |
|---|---|---|---|
| `execution_result_use_awx_api` | no | Auto-enabled when `TOWER_HOST` and `TOWER_OAUTH_TOKEN` credentials exist | Enable fetching metadata from AWX/Tower API |
| `execution_result_awx_api_url` | no | Auto-populated from `TOWER_HOST`, `CONTROLLER_HOST`, or `AWX_HOST` env vars | AWX/Tower API base URL |
| `execution_result_awx_token` | no | Auto-populated from `TOWER_OAUTH_TOKEN`, `CONTROLLER_OAUTH_TOKEN`, or `AWX_OAUTH_TOKEN` env vars | OAuth Bearer token for API authentication |
| `execution_result_awx_job_id` | no | Auto-populated from `TOWER_JOB_ID`, `AWX_JOB_ID`, `CONTROLLER_JOB_ID`, `WORKFLOW_JOB_ID`, or `JOB_ID` env vars | Job ID to fetch metadata for |
| `execution_result_awx_validate_certs` | no | `false` | Validate SSL certificates when connecting to AWX API |

**Example: Override API Configuration (for local testing)**
```yaml
- name: Record execution failure with API metadata
  ansible.builtin.include_role:
    name: execution_result
  vars:
    execution_result_return_code: 1
    execution_result_message: "Task failed"
    # Override API configuration for local testing
    execution_result_awx_api_url: "https://awx.example.com"
    execution_result_awx_token: "{{ lookup('env', 'MY_AWX_TOKEN') }}"
    execution_result_awx_job_id: "12345"
```

---

## Configuration Variables

Override these in your playbook or inventory to control role behaviour:

| Variable | Required | Default | Description |
|---|---|---|---|
| `execution_result_accumulate` | no | `true` | When `true`: builds a list of results across multiple role invocations. When `false`: only publishes the current result (useful to prevent duplicates when role is called multiple times) |
| `execution_result_results_fact` | no | `execution_results` | Name of the fact that holds accumulated results |
| `execution_result_fail_on_error` | no | `false` | Fail the play when `return_code` is non-zero |
| `execution_result_set_stats_enabled` | no | `true` | Publish results to AWX/Tower job Artifacts via `set_stats` |
| `execution_result_set_stats_per_host` | no | `false` | Store stats per-host in addition to the aggregated artifact |

---

## Usage

### Simplified Pattern with Registered Variable (Recommended)

The easiest way to use this role is to pass the entire registered variable from your task:

```yaml
- name: Perform a critical operation
  block:
    - name: Run the primary task
      ansible.builtin.command: /usr/bin/my_script.sh
      register: task_result

  rescue:
    - name: Record execution failure
      ansible.builtin.include_role:
        name: execution_result
      vars:
        execution_result_registered_var: "{{ task_result }}"

  always:
    - name: Record execution success
      ansible.builtin.include_role:
        name: execution_result
      vars:
        execution_result_registered_var: "{{ task_result }}"
```

The role will automatically extract all fields from the registered variable including return code, message, stdout, stderr, exception, and warnings.

### Advanced Pattern with Individual Fields

For more control, you can explicitly specify individual fields:

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
        # Optional OS details (will only appear in results if provided)
        execution_result_os_distribution: "{{ ansible_distribution | default('') }}"
        execution_result_os_version: "{{ ansible_distribution_version | default('') }}"
        execution_result_os_family: "{{ ansible_os_family | default('') }}"
        execution_result_os_system: "{{ ansible_system | default('') }}"
        execution_result_os_architecture: "{{ ansible_architecture | default('') }}"
```

### Using with Ping Results

For ping or connectivity tests:

```yaml
- name: Test connectivity
  block:
    - name: Ping target host
      ansible.builtin.ping:
      register: ping_result

    - name: Capture Execution Details (Success)
      ansible.builtin.include_role:
        name: execution_result
      vars:
        execution_result_registered_var: "{{ ping_result }}"

  rescue:
    - name: Capture Execution Details (Failure)
      ansible.builtin.include_role:
        name: execution_result
      vars:
        execution_result_registered_var: "{{ ping_result }}"
```

### Basic block/rescue pattern (Legacy)

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
        # Optional OS details (will only appear in results if provided)
        execution_result_os_distribution: "{{ ansible_distribution | default('') }}"
        execution_result_os_version: "{{ ansible_distribution_version | default('') }}"
        execution_result_os_family: "{{ ansible_os_family | default('') }}"
        execution_result_os_system: "{{ ansible_system | default('') }}"
        execution_result_os_architecture: "{{ ansible_architecture | default('') }}"
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
  project:                "My Ansible Project"     # AWX/Tower project name
  organization:           "IT Operations"          # AWX/Tower organization
  job_template:           "Deploy Application"     # AWX/Tower job template
  playbook:               "N/A"                    # Playbook filename (if set)
  scm_url:                "https://github.com/..."  # Git repository URL
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
  os_distribution:        "Ubuntu"                 # Operating system distribution
  os_version:             "20.04"                  # OS version
  os_family:              "Debian"                 # OS family
  os_system:              "Linux"                  # System type
  os_architecture:        "x86_64"                 # Architecture
```

### AWX/Tower job Artifacts

When `execution_result_set_stats_enabled: true` (the default), the role publishes results to the **Artifacts** section of the AWX/Tower job:

**With `execution_result_accumulate: true` (default):**
- Builds a list of all results across multiple role invocations in the same playbook run
- Useful for tracking multiple different tasks/operations

**With `execution_result_accumulate: false`:**
- Publishes only the current result (overwrites previous on each invocation)
- Useful when role is called multiple times but you only want the latest result

| Artifact key | Description |
|---|---|
| `execution_results` | List containing result entries. Each entry contains: `timestamp`, `project` (from API), `organization` (from API), `job_template` (from API), `playbook` (from API), `scm_url` (from API), `scm_branch` (from API), `scm_revision` (from API), `execution_environment` (from API), `status`, `return_code`, `stdout`, `stderr`, `message`, `exception`, `warnings`, `failed_task`, `failed_task_module`, `os_distribution`, `os_version`, `os_family`, `os_system`, `os_architecture` |

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
- `gather_facts: true` is recommended so the role can use `ansible_date_time`; if facts are not gathered, the role falls back to a UTC `date` lookup
- For AWX/Tower metadata capture: "Red Hat Ansible Automation Platform" credential attached to job template

---

## License

MIT
