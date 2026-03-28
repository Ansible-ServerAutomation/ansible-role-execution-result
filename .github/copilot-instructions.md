# Ansible Role: execution-result

An Ansible role that tracks and logs task execution results from `block/rescue/always` sections, publishing outcomes to AWX/Tower job Artifacts via `set_stats`.

## Repository Context

- **Current branch**: `development` 
- **Default branch**: `main`
- **Owner**: Ansible-ServerAutomation
- **Purpose**: Reusable role for error tracking and audit logging across automation workflows

## Architecture

### Core Workflow ([tasks/main.yml](../tasks/main.yml))

1. **Validate** required inputs (`execution_result_return_code`, `execution_result_message`)
2. **Normalize** data (determine SUCCESS/FAILURE status, capture timestamp, format fields)
3. **Build** structured result entry with 18 fields (timestamp, host_os, project, organization, job_template, playbook, scm_branch, scm_revision, execution_environment, status, return_code, stdout, stderr, message, exception, warnings, failed_task, failed_task_module)
4. **Accumulate** in Ansible fact (`execution_results` by default)
5. **Publish** to AWX/Tower Artifacts using `ansible.builtin.set_stats`
6. **Log** to file (OS-specific: [tasks/logging_windows.yml](../tasks/logging_windows.yml) for Windows hosts)
7. **Display** formatted debug output
8. **Optionally fail** play if `execution_result_fail_on_error: true` and return_code ≠ 0

### Platform Support

- **Linux**: Uses `ansible.builtin.file` and `ansible.builtin.lineinfile`
- **Windows**: Uses `ansible.windows.win_file` and `ansible.windows.win_lineinfile` ([tasks/logging_windows.yml](../tasks/logging_windows.yml))
- **OS Detection**: `ansible_os_family` conditionals and ternary operators for path defaults

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

**AWX/Tower Metadata** (auto-captured from environment variables):
- `execution_result_project_name`: Auto-populated from `AWX_PROJECT_NAME` or `TOWER_PROJECT_NAME`
- `execution_result_organization`: Auto-populated from `TOWER_ORGANIZATION`
- `execution_result_scm_revision`: Auto-populated from `AWX_PROJECT_REVISION` (Git commit SHA)
- `execution_result_scm_branch`: Auto-populated from `AWX_PROJECT_SCM_BRANCH` (Git branch)
- `execution_result_execution_environment`: Auto-populated from `AWX_EXECUTION_ENVIRONMENT`
- `execution_result_job_template`: Auto-populated from `AWX_JOB_TEMPLATE_NAME` or `TOWER_JOB_TEMPLATE_NAME`
- `execution_result_playbook_name`: Must be set manually if needed (no auto-detection)

These AWX metadata fields are **automatically populated** when the role runs in AWX/Tower. Users do not need to provide them unless they want to override the auto-detected values.

**Critical**: To capture warnings, you **must** `register:` the task and pass `task_result.warnings`. See [examples/WARNINGS_GUIDE.md](../examples/WARNINGS_GUIDE.md).

### Configuration Defaults

Default behavior ([defaults/main.yml](../defaults/main.yml)):
- `execution_result_log_enabled: false` (logging disabled by default)
- `execution_result_accumulate: true` (build fact list across invocations)
- `execution_result_set_stats_enabled: true` (publish to AWX/Tower Artifacts)
- `execution_result_fail_on_error: false` (continue on failure)

### File Structure Patterns

- `defaults/main.yml`: User-facing configuration with documentation comments
- `vars/main.yml`: Internal constants (role name, version)
- `tasks/main.yml`: Primary orchestration workflow
- `tasks/logging_windows.yml`: Platform-specific includes
- `templates/execution_result_entry.j2`: Log file format template

## Development Guidelines

### When Modifying Tasks

1. **Preserve validation**: Keep `ansible.builtin.assert` checks for required variables at top of [tasks/main.yml](../tasks/main.yml)
2. **Maintain OS-agnostic design**: Use `ansible_os_family` conditionals, not hardcoded paths
3. **Follow normalization pattern**: All empty/null fields render as `(none)` for consistent output
4. **Test both platforms**: Changes affecting logging must work on Linux and Windows

### When Adding Variables

1. **Namespace correctly**: `execution_result_*` for public, `_execution_result_*` for internal
2. **Document in defaults**: Add to [defaults/main.yml](../defaults/main.yml) with inline comments
3. **Update README**: Reflect changes in variable tables and examples
4. **Consider AWX/Tower impact**: New fields should appear in Artifacts output

### Templates and Logging

- **Log format**: [templates/execution_result_entry.j2](../templates/execution_result_entry.j2) uses YAML-like structure with `---` delimiters
- **Indentation**: 2-space for multi-line fields
- **Conditional rendering**: Only show warnings list if present
- **Idempotency**: Uses `lineinfile` append-only mode

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
- [ ] Validate log file on both Linux and Windows targets
- [ ] Confirm fact accumulation across multiple role invocations

### Common Gotchas

1. **Missing warnings**: Forgot to `register:` the task being tracked
2. **Wrong OS paths**: Hardcoded paths instead of using `ansible_os_family` ternary
3. **Empty facts**: `execution_result_accumulate: false` disables accumulation and AWX publishing
4. **Log permissions**: Windows targets require proper directory access for logging

## Commands

### Install Dependencies

```bash
ansible-galaxy collection install ansible.windows
```

### Test Locally

```bash
# Run example playbook against inventory
ansible-playbook examples/example_playbook.yml -i inventory.ini

# Test with warnings
ansible-playbook examples/example_with_warnings.yml -i inventory.ini
```

## Key Files

- **[README.md](../README.md)**: User-facing documentation, usage examples, variable reference
- **[tasks/main.yml](../tasks/main.yml)**: Core orchestration logic (validation → normalization → accumulation → logging → display)
- **[tasks/logging_windows.yml](../tasks/logging_windows.yml)**: Windows-specific task includes
- **[defaults/main.yml](../defaults/main.yml)**: All user-overridable configuration (31 lines)
- **[templates/execution_result_entry.j2](../templates/execution_result_entry.j2)**: Log file format template
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
```

AWX/Tower workflows can consume artifacts via `{{ artifacts['execution_results'] }}` in downstream job templates.
