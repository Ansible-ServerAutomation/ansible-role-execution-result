# Ansible Role: execution-result

An Ansible role that tracks task execution results from `block/rescue/always` sections, publishing outcomes to AWX/Tower job Artifacts via `set_stats`.

## Repository Context

- **Current branch**: `development` 
- **Default branch**: `main`
- **Owner**: Ansible-ServerAutomation
- **Purpose**: Reusable role for error tracking and audit reporting across automation workflows

## Architecture

### Core Workflow ([tasks/main.yml](../tasks/main.yml))

1. **Normalize** data (determine SUCCESS/FAILURE status, capture timestamp, format fields)
2. **Fetch AWX metadata** via AWX API using OAuth Bearer token authentication (auto-enabled when credentials exist)
3. **Fetch project details** for SCM branch and URL when needed
4. **Build** structured result entry with 23 fields (timestamp, project, organization, job_template, playbook, scm_url, scm_branch, scm_revision, execution_environment, status, return_code, stdout, stderr, message, exception, warnings, failed_task, failed_task_module, os_distribution, os_version, os_family, os_system, os_architecture)
5. **Accumulate** in Ansible fact (`execution_results` by default)
6. **Publish** to AWX/Tower Artifacts using `ansible.builtin.set_stats`
7. **Display** formatted debug output
8. **Optionally fail** play if `execution_result_fail_on_error: true` and return_code ≠ 0

### Platform Support

- **All platforms**: Runs on localhost (AWX runner) via `delegate_to: localhost`
- **OS-agnostic**: No platform-specific tasks (removed logging feature)

## Code Conventions

### Variable Namespacing

- **Public variables**: `execution_result_*` (user-overridable, defined in [defaults/main.yml](../defaults/main.yml))
- **Internal variables**: `_execution_result_*` (role-private, set in tasks)
- **Fact name**: Configurable via `execution_result_results_fact` (default: `execution_results`)

### Required Inputs

When calling this role:
- `execution_result_return_code`: Integer (0 = success, non-zero = failure)
- `execution_result_message`: String (human-readable outcome)

Optional but important:
- `execution_result_failed_task`: String (task name for audit trail)
- `execution_result_warnings`: List (from registered task's `.warnings` field)
- `execution_result_os_distribution`: String (OS distribution, e.g., Ubuntu, CentOS)
- `execution_result_os_version`: String (OS version, e.g., 20.04)
- `execution_result_os_family`: String (OS family, e.g., Debian, RedHat)
- `execution_result_os_system`: String (System type, e.g., Linux, Windows)
- `execution_result_os_architecture`: String (Architecture, e.g., x86_64)

**AWX/Tower Metadata** (auto-captured using API-first approach):
- **When running in AWX/Tower**: Automatically fetched from AWX API when `TOWER_JOB_ID` environment variable is detected
- **Priority chain**: API-fetched values → Environment variables → 'N/A'
- **Fields captured**: `project`, `organization`, `scm_url`, `scm_revision`, `scm_branch`, `execution_environment`, `job_template`, `playbook`

**AWX API Configuration** (auto-detected from environment):
- `execution_result_use_awx_api`: Auto-enabled when `TOWER_JOB_ID` exists
- `execution_result_awx_api_url`: Auto-detected from `TOWER_HOST` or `AWX_HOST`
- `execution_result_awx_token`: Auto-detected from `TOWER_OAUTH_TOKEN` or `AWX_OAUTH_TOKEN`
- `execution_result_awx_job_id`: Auto-detected from `TOWER_JOB_ID` or `AWX_JOB_ID`

**Override API configuration** for local testing:
```yaml
execution_result_awx_api_url: "https://awx.example.com"
execution_result_awx_token: "{{ lookup('env', 'MY_AWX_TOKEN') }}"
execution_result_awx_job_id: "12345"
```

**Critical**: To capture warnings, you **must** `register:` the task and pass `task_result.warnings`. See [examples/WARNINGS_GUIDE.md](../examples/WARNINGS_GUIDE.md).

### Configuration Defaults

Default behavior ([defaults/main.yml](../defaults/main.yml)):
- `execution_result_accumulate: true` (build fact list across invocations)
- `execution_result_set_stats_enabled: true` (publish to AWX/Tower Artifacts)
- `execution_result_fail_on_error: false` (continue on failure)
- `execution_result_use_awx_api: auto-enabled` (when TOWER_HOST and TOWER_OAUTH_TOKEN credentials exist)
- API configuration auto-detected from environment variables (`TOWER_HOST`, `TOWER_OAUTH_TOKEN`, `JOB_ID`)
- `execution_result_awx_validate_certs: false` (disable SSL verification for self-signed certs)

### File Structure Patterns

- `defaults/main.yml`: User-facing configuration with documentation comments
- `vars/main.yml`: Internal constants (role name, version)
- `tasks/main.yml`: Primary orchestration workflow (all tasks delegated to localhost)

## Development Guidelines

### When Modifying Tasks

1. **Maintain delegation**: All tasks must run on localhost via `delegate_to: localhost` to access AWX credentials
2. **Follow normalization pattern**: All empty/null fields render as `(none)` for consistent output
3. **API error handling**: Use `ignore_errors: true` with conditionals for optional API calls
4. **Rely on defaults**: Variables have intelligent defaults with fallback chains; no explicit validation needed

### When Adding Variables

1. **Namespace correctly**: `execution_result_*` for public, `_execution_result_*` for internal
2. **Document in defaults**: Add to [defaults/main.yml](../defaults/main.yml) with inline comments
3. **Update README**: Reflect changes in variable tables and examples
4. **Consider AWX/Tower impact**: New fields should appear in Artifacts output

### Example Patterns

[examples/](../examples/) demonstrates:
- **[example_playbook.yml](../examples/example_playbook.yml)**: Standard block/rescue integration with magic variables (`ansible_failed_task.name`, `ansible_failed_result`)
- **[example_with_warnings.yml](../examples/example_with_warnings.yml)**: Proper warnings capture using `register:`

## Testing and Validation

### Manual Testing Checklist

- [ ] Test success path (return_code = 0)
- [ ] Test failure path (return_code ≠ 0)
- [ ] Verify warnings capture with registered tasks
- [ ] Check AWX/Tower Artifacts tab shows `execution_results`
- [ ] Verify API metadata capture (all 8 fields populated)
- [ ] Test with missing job ID (should still work with manual override)
- [ ] Confirm fact accumulation across multiple role invocations

### Common Gotchas

1. **Missing warnings**: Forgot to `register:` the task being tracked
2. **Empty AWX metadata**: Credential not attached to job template
3. **Empty facts**: `execution_result_accumulate: false` disables accumulation and AWX publishing
4. **Wrong job ID env var**: Your AWX version might use `JOB_ID` instead of `TOWER_JOB_ID`
5. **Empty scm_branch or scm_url**: Role automatically fetches from project endpoint

## Commands

### Test Locally

```bash
# Run example playbook against inventory
ansible-playbook examples/example_playbook.yml -i inventory.ini

# Test with warnings
ansible-playbook examples/example_with_warnings.yml -i inventory.ini

# Test with API override (when running outside AWX)
ansible-playbook examples/example_playbook.yml -i inventory.ini -e "execution_result_awx_job_id_override=123"
```

## Key Files

- **[README.md](../README.md)**: User-facing documentation, usage examples, variable reference
- **[tasks/main.yml](../tasks/main.yml)**: Core orchestration logic (normalization → API fetch → accumulation → display)
- **[defaults/main.yml](../defaults/main.yml)**: All user-overridable configuration
- **[examples/WARNINGS_GUIDE.md](../examples/WARNINGS_GUIDE.md)**: Troubleshooting warnings capture

## Integration Context

This role is designed to be invoked from **other roles** in rescue/always blocks:

```yaml
rescue:
  - name: Record execution failure
    ansible.builtin.include_role:
      name: execution_result
    vars:
      execution_result_return_code: "{{ task_result.rc | default(1) }}"
      execution_result_message: "{{ task_result.stderr | default('Unknown error') }}"
      execution_result_failed_task: "{{ ansible_failed_task.name }}"
      execution_result_warnings: "{{ task_result.warnings | default([]) }}"
      # Optional OS details (only included if supplied)
      execution_result_os_distribution: "{{ ansible_distribution | default('') }}"
      execution_result_os_version: "{{ ansible_distribution_version | default('') }}"
      execution_result_os_family: "{{ ansible_os_family | default('') }}"
      execution_result_os_system: "{{ ansible_system | default('') }}"
      execution_result_os_architecture: "{{ ansible_architecture | default('') }}"
      # AWX metadata is auto-captured via API when TOWER_JOB_ID exists
      # No manual configuration needed when running in AWX/Tower
```

AWX/Tower workflows can consume artifacts via `{{ artifacts['execution_results'] }}` in downstream job templates.
