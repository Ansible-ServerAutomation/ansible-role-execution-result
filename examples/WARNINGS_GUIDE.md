# How to Capture Warnings - Quick Fix Guide

## Your Issue

Looking at your job output, the Python interpreter warning appeared during the ping task:

```
TASK [ansible-role-ping-test : Ping managed host] ******************************
[WARNING]: Platform linux on host Ubuntu is using the discovered Python
interpreter at /usr/bin/python3.12...
ok: [Ubuntu]
```

But the `execution_results` shows `warnings: []`.

## Root Cause

To capture warnings, you must:
1. **Register** the task result that generates the warning
2. **Pass** that result's `warnings` field to `execution_result_warnings`

## Solution

In your `ansible-role-ping-test` role, modify the ping task to register the result:

### Before (in ansible-role-ping-test/tasks/main.yml):
```yaml
- name: Ping managed host
  ansible.builtin.ping:
```

### After:
```yaml
- name: Ping managed host
  ansible.builtin.ping:
  register: ping_task_result
```

Then when calling the execution_result role, pass the warnings:

### Before:
```yaml
- name: Capture Execution Details (Success)
  ansible.builtin.include_role:
    name: execution_result
  vars:
    execution_result_return_code: 0
    execution_result_message: "Task completed"
```

### After (Simplified Pattern - Recommended):
```yaml
- name: Capture Execution Details (Success)
  ansible.builtin.include_role:
    name: execution_result
  vars:
    # Simplified: Just pass the registered variable
    execution_result_registered_var: "{{ ping_task_result }}"
```

### After (Advanced Pattern):
```yaml
- name: Capture Execution Details (Success)
  ansible.builtin.include_role:
    name: execution_result
  vars:
    execution_result_return_code: "{{ ping_task_result.rc | default(0) }}"
    execution_result_message: "Ping successful"
    execution_result_warnings: "{{ ping_task_result.warnings | default([]) }}"
    execution_result_stdout: "{{ ping_task_result.stdout | default('') }}"
    execution_result_stderr: "{{ ping_task_result.stderr | default('') }}"
```

## What Gets Captured

✅ **Task-level warnings** (these appear during task execution):
- Python interpreter warnings
- Module deprecation warnings
- Package manager warnings

❌ **Play-level warnings** (these appear before the play starts):
- `[DEPRECATION WARNING]: ANSIBLE_COLLECTIONS_PATHS...`
- Ansible configuration warnings

The deprecation warning about `ANSIBLE_COLLECTIONS_PATHS` in your output cannot be captured because it's shown before any tasks run. It's an Ansible configuration issue, not a task result.

## Expected Output After Fix

After making these changes, your next job run should show:

```yaml
execution_results:
  - timestamp: '2026-03-25T21:15:19Z'
    host: Ubuntu
    status: SUCCESS
    return_code: 0
    message: Ping successful
    warnings:
      - "Platform linux on host Ubuntu is using the discovered Python interpreter at /usr/bin/python3.12, but future installation of another Python interpreter could change the meaning of that path. See https://docs.ansible.com/ansible-core/2.18/reference_appendices/interpreter_discovery.html for more information."
    failed_task: ''
```

## Additional Examples

See the new example file: `examples/example_with_warnings.yml` for complete working examples.
