# 03 — Ansible Playbooks: Variables

This README is a learning companion for this section of the *Dive Into Ansible* course. It walks through each numbered example (`01`–`17`) in this folder, in the order you're meant to learn them, covering every major source Ansible variables can come from: play `vars`, external `vars_files`, interactive `vars_prompt`, `hostvars`, group/host inventory variables, and command-line extra vars.

The goal is to build a complete mental map of Ansible's variable precedence and access patterns — dot notation vs. bracket notation, defaults, and where a given value is actually coming from.

---

## How to use this folder while learning

Each subfolder `01`–`17` is a **self-contained mini-lab** with its own `ansible.cfg`, `hosts` inventory, and a `variables_playbook.yaml` (examples with inventory-based variables also include `group_vars`/`host_vars` directories). There's also a top-level `show_examples.sh` script that walks through every example interactively, pausing to show the playbook source before running it:

```bash
cd 02-ansible-playbooks-introduction/03-ansible-playbooks-variables
./show_examples.sh
```

Or run any example individually:

```bash
cd 01
ansible-playbook variables_playbook.yaml
```

---

## Walking Through the Examples

### 01 — A simple named variable

```yaml
-
  hosts: centos1
  gather_facts: False

  vars:
    example_key: example value

  tasks:
    - name: Test dictionary key value
      debug:
        msg: "{{ example_key }}"
```

The most basic case: a scalar variable defined in `vars:` and referenced with `{{ example_key }}`.

---

### 02 — Named dictionary, dot vs. bracket notation

```yaml
  vars:
    dict:
      dict_key: This is a dictionary value

  tasks:
    - name: Test named dictionary dictionary
      debug:
        msg: "{{ dict }}"

    - name: Test named dictionary dictionary key value with dictionary dot notation
      debug:
        msg: "{{ dict.dict_key }}"

    - name: Test named dictionary dictionary key value with python brackets notation
      debug:
        msg: "{{ dict['dict_key'] }}"
```

Introduces a **nested dictionary variable**, and shows the two equivalent ways to access a nested key: `dict.dict_key` (dot notation) and `dict['dict_key']` (Python-style bracket notation). Both produce identical results here.

---

### 03 — Inline (flow-style) dictionary

```yaml
  vars:
    inline_dict:
      {inline_dict_key: This is an inline dictionary value}

  tasks:
    - name: Test named inline dictionary dictionary
      debug:
        msg: "{{ inline_dict }}"
    - name: Test named inline dictionary dictionary key value with dictionary dot notation
      debug:
        msg: "{{ inline_dict.inline_dict_key }}"
    - name: Test named inline dictionary dictionary key value with brackets notation
      debug:
        msg: "{{ inline_dict['inline_dict_key'] }}"
```

Direct counterpart to `02`, but defined using flow-style `{ }` syntax instead of block indentation (see `01-yaml/13` for the underlying YAML concept). Access patterns are identical.

---

### 04 — Named list, index access

```yaml
  vars:
    named_list:
      - item1
      - item2
      - item3
      - item4

  tasks:
    - name: Test named list
      debug:
        msg: "{{ named_list }}"
    - name: Test named list first item dot notation
      debug:
        msg: "{{ named_list.0 }}"
    - name: Test named list first item brackets notation
      debug:
        msg: "{{ named_list[0] }}"
```

Same dot-vs-bracket idea as `02`, but for **lists**: `named_list.0` and `named_list[0]` both retrieve the first (zero-indexed) item.

---

### 05 — Inline (flow-style) list

```yaml
  vars:
    inline_named_list:
      [ item1, item2, item3, item4 ]

  tasks:
    - name: Test inline named list
      debug:
        msg: "{{ inline_named_list }}"
    - name: Test inline named list first item dot notation
      debug:
        msg: "{{ inline_named_list.0 }}"
    - name: Test inline named list first item brackets notation
      debug:
        msg: "{{ inline_named_list[0] }}"
```

The flow-style counterpart to `04`, using `[ ]` inline syntax.

---

### 06 — External variables via `vars_files`

`external_vars.yaml`:
```yaml
external_example_key: example value

external_dict:
   dict_key: This is a dictionary value

external_inline_dict: 
   {inline_dict_key: This is an inline dictionary value}

external_named_list:
   - item1
   - item2
   - item3
   - item4

external_inline_named_list:
   [ item1, item2, item3, item4 ]
```

`variables_playbook.yaml`:
```yaml
  vars_files:
    - external_vars.yaml

  tasks:
    - name: Test external dictionary key value
      debug:
        msg: "{{ external_example_key }}"
    # ...repeats every pattern from 01–05, but sourced externally
```

Introduces `vars_files:`, which loads variables from a separate YAML file rather than defining them inline. This single external file re-demonstrates every variable shape covered in `01`–`05` (scalar, dictionary, inline dictionary, list, inline list) — a good file to compare side-by-side with the earlier examples.

---

### 07 — Interactive prompt, visible input

```yaml
  vars_prompt:
    - name: username
      private: False

  tasks:
    - name: Test vars_prompt
      debug:
        msg: "{{ username }}"
```

Introduces `vars_prompt:`, which pauses the playbook run and interactively asks for a value at the terminal. `private: False` means the typed input is **echoed to the screen** — appropriate for non-sensitive values like a username.

---

### 08 — Interactive prompt, hidden input

```yaml
  vars_prompt:
    - name: password
      private: True

  tasks:
    - name: Test vars_prompt
      debug:
        msg: "{{ password }}"
```

The direct counterpart to `07`: `private: True` hides the typed input (like a terminal password prompt) — the safer default for sensitive values, though note the value is still printed in plaintext by the `debug` task here for teaching purposes.

---

### 09 — `hostvars` and `ansible_port`

```yaml
  hosts: centos1
  gather_facts: True

  tasks:
    - name: Test hostvars with an ansible fact and collect ansible_port, dot notation
      debug:
        msg: "{{ hostvars[ansible_hostname].ansible_port }}"

    - name: Test hostvars with an ansible fact and collect ansible_port, dict notation
      debug:
        msg: "{{ hostvars[ansible_hostname]['ansible_port'] }}"
```

Introduces `hostvars`, a special Ansible variable holding variables/facts for **every host in the play**, indexed by hostname. Here it's used to look up the current host's own `ansible_port` (an inventory-defined connection variable) — again with both dot and bracket notation. Note `gather_facts: True` is required so `ansible_hostname` is available.

---

### 10 — Same lookup, different host

```yaml
  hosts: centos
```

Identical playbook logic to `09`, but targets the `centos` group instead of `centos1` — a good one to run back-to-back with `09` to confirm the same pattern works across different inventory groups/hosts.

---

### 11 — `hostvars` with a `default()` filter

```yaml
    - name: Test hostvars with an ansible fact and collect ansible_port, dot notation, default if not found
      debug:
        msg: "{{ hostvars[ansible_hostname].ansible_port | default('22') }}"

    - name: Test hostvars with an ansible fact and collect ansible_port, dict notation, default if not found
      debug:
        msg: "{{ hostvars[ansible_hostname]['ansible_port'] | default('22') }}"
```

Adds the `| default('22')` filter to both lookups from `09`/`10`, so the task doesn't fail with an "undefined variable" error if `ansible_port` isn't explicitly set for a host — falling back to the standard SSH port `22` instead. A very common defensive pattern in real playbooks.

---

### 12 — Group variables (`ansible_user`)

```yaml
  hosts: centos
  gather_facts: True

  tasks:
    - name: Test groupvars
      debug:
        msg: "{{ ansible_user }}"
```

Introduces `group_vars` — this example's `group_vars` directory defines `ansible_user` for the `centos` group, and the playbook simply references it directly as `{{ ansible_user }}` (no `hostvars[...]` needed when it's the *current* host).

---

### 13 — Group variables on a different group

```yaml
  hosts: ubuntu
```

Same lookup as `12`, but targets `ubuntu` instead of `centos` — confirming that `ansible_user` resolves differently depending on which group's `group_vars` apply.

---

### 14 — Accessing group vars via `hostvars`

```yaml
  hosts: centos1
  gather_facts: True

  tasks:
    - name: Test groupvars with an ansible fact, show that the variable is also accessible from the hostvars section
      debug:
        msg: "{{ hostvars[ansible_hostname].ansible_user }}"
```

Demonstrates that group/host inventory variables aren't only accessible directly (as in `12`/`13`) — they're *also* visible through `hostvars[hostname]`, reinforcing that `hostvars` is a comprehensive view into every variable Ansible knows about for a host.

---

### 15 — Combining hostvars + groupvars in one playbook

```yaml
  hosts: centos1
  gather_facts: True

  tasks:
    - name: Test hostvars with an ansible fact and collect ansible_port, dot notation
      debug:
        msg: "{{ hostvars[ansible_hostname].ansible_port }}"

    - name: Test groupvars
      debug:
        msg: "{{ ansible_user }}"
```

Combines the `hostvars`/`ansible_port` pattern from `09`/`10` with the direct `group_vars`/`ansible_user` pattern from `12` in a single playbook — a good "everything together" checkpoint before moving to extra vars.

---

### 16 — Extra vars via `-e` (command line)

```yaml
  hosts: centos1

  tasks:
    - name: Test extra vars
      debug:
        msg: "{{ extra_vars_key }}"
```

No `vars:`/`vars_files:`/`vars_prompt:` at all — `extra_vars_key` is expected to come purely from the command line via `-e`, in any of three equivalent formats:

```bash
ansible-playbook variables_playbook.yaml -e 'extra_vars_key="extra vars value"'
ansible-playbook variables_playbook.yaml -e '{"extra_vars_key": "extra vars value"}'
ansible-playbook variables_playbook.yaml -e '{extra_vars_key: extra vars value}'
```

Extra vars (`-e`) have the **highest precedence** of any variable source in Ansible — they override play vars, `vars_files`, group/host vars, everything.

---

### 17 — Extra vars from a file (`@file`)

Identical playbook to `16`, but this time the extra vars are supplied from external files rather than typed inline, using the `@` prefix:

`extra_vars_file.yaml`:
```yaml
extra_vars_key: extra vars value
```

`extra_vars_file.json`:
```json
{
    "extra_vars_key": "extra vars value"
}
```

```bash
ansible-playbook variables_playbook.yaml -e @extra_vars_file.yaml
ansible-playbook variables_playbook.yaml -e @extra_vars_file.json
```

Useful when extra vars are numerous, sensitive, or shared across multiple playbook runs (e.g., a CI pipeline passing a vars file rather than a long inline string).

---

## Summary Table

| # | Concept | Key Feature(s) | Notes |
|---|---------|-----------------|-------|
| 01 | Simple variable | `vars:` | |
| 02 | Named dictionary | dot / bracket notation | |
| 03 | Inline dictionary | `{ }` flow style | Same access as `02` |
| 04 | Named list | dot / bracket index | |
| 05 | Inline list | `[ ]` flow style | Same access as `04` |
| 06 | External vars file | `vars_files:` | Re-demonstrates `01`–`05` externally |
| 07 | Interactive prompt (visible) | `vars_prompt:`, `private: False` | |
| 08 | Interactive prompt (hidden) | `vars_prompt:`, `private: True` | |
| 09 | `hostvars` + `ansible_port` | `hostvars[host].fact` | Requires `gather_facts: True` |
| 10 | `hostvars`, different host | Same as `09` | Targets `centos` group |
| 11 | `hostvars` + `default()` | `\| default('22')` | Avoids undefined-variable errors |
| 12 | Group vars | `group_vars`, `ansible_user` | |
| 13 | Group vars, different group | Same as `12` | Targets `ubuntu` group |
| 14 | Group vars via `hostvars` | `hostvars[host].ansible_user` | Confirms visibility from both angles |
| 15 | Combined hostvars + groupvars | Both patterns together | |
| 16 | Extra vars (`-e`, inline) | ini / JSON / YAML syntax | Highest precedence |
| 17 | Extra vars (`-e @file`) | YAML file / JSON file | Same precedence as `16` |

---

## Quick-Reference: Variable Precedence (Lowest → Highest, as demonstrated here)

| Source | Example(s) | Notes |
|---|---|---|
| Play `vars:` | `01`–`05` | Defined directly in the playbook |
| `vars_files:` | `06` | Loaded from an external file |
| `vars_prompt:` | `07`, `08` | Collected interactively at runtime |
| `group_vars` / `host_vars` | `12`–`15` | Defined per-group or per-host in inventory structure |
| Extra vars (`-e`) | `16`, `17` | Always wins — highest precedence of all |

---

## Suggested Next Steps

1. Run `06` and compare its output line-by-line against `01`–`05` to confirm external vars behave identically to inline vars.
2. Run `16` three times, once with each `-e` syntax (ini/JSON/YAML), to prove they're equivalent.
3. Deliberately omit `-e` when running `16`/`17` to see the "undefined variable" error, then fix it — a good way to internalize why `default()` (from `11`) is often worth adding defensively.
4. Try overriding a `group_vars`-defined value (from `12`) using `-e` on the command line, to confirm extra vars really do take top precedence.
