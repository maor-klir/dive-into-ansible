# 01 — Inventories

This README is a learning companion for this section of the *Dive Into Ansible* course. It walks through each numbered example (`01`–`16`) in this folder, in the order you're meant to learn them, plus the `challenge`, `hands-on`, and `templates` folders, explaining how an Ansible inventory grows from a single bare hostname into a fully structured, multi-format, variable-driven description of your infrastructure.

The goal isn't just to show *what* each example does, but *why* it's built that way — including how INI-style inventory shorthand maps onto the richer YAML/JSON forms, and how Ansible resolves a variable when it's set at more than one level (host, group, parent group, and "all") at once.

---

## How to use this folder while learning

Each subfolder `01`–`16` is a **self-contained mini-lab** containing its own `ansible.cfg` and `hosts` inventory file (examples `14`–`16` also add `hosts.yaml` and/or `hosts.json`). `cd` into any one and point `ansible-inventory` at it to see how Ansible actually parses the file:

```bash
cd 01-ansible-architecture-and-design/01-inventories/03
ansible-inventory --list
ansible-inventory --graph
```

`--graph` is the fastest way to see group structure at a glance; `--list` shows the fully resolved hosts-and-vars view, which is what you want once variable precedence enters the picture from `10` onward.

---

## Walking Through the Examples

### 01 — The simplest possible inventory

```ini
[all]
centos1
```

A single host, `centos1`, under the implicit-everywhere `[all]` group. No `ansible.cfg` options beyond pointing at the file itself (`inventory = hosts`). This is the absolute floor of what a valid Ansible inventory looks like.

---

### 02 — Quieting SSH host-key prompts

```ini
[defaults]
inventory = hosts
host_key_checking = False
```

Same one-host inventory as `01`, but adds `host_key_checking = False` to `ansible.cfg`. Without this, the very first connection to any new host pauses for an interactive `yes/no` SSH host-key prompt — disruptive when running playbooks non-interactively (e.g. in CI, or against throwaway lab VMs). This setting carries forward into every remaining example in this folder.

---

### 03 — Splitting hosts into groups

```ini
[centos]
centos1
centos2
centos3

[ubuntu]
ubuntu1
ubuntu2
ubuntu3
```

Introduces **groups**: `[centos]` and `[ubuntu]`, each with three hosts. This is the shape almost every real inventory starts from — hosts grouped by OS, role, environment, or datacenter, so a playbook can target `hosts: centos` instead of listing hostnames individually.

---

### 04 — Per-host connection variable (`ansible_user`)

```ini
[centos]
centos1 ansible_user=root
centos2 ansible_user=root
centos3 ansible_user=root

[ubuntu]
ubuntu1
ubuntu2
ubuntu3
```

Each centos host gets `ansible_user=root` appended inline — a **host variable**, set directly after the hostname on the same line. This tells Ansible to connect as `root` rather than the default/current user, without needing anything in the playbook itself. Notice the repetition across all three centos hosts — a pattern that gets addressed in `10`.

---

### 05 — Privilege escalation variables (`ansible_become*`)

```ini
[ubuntu]
ubuntu1 ansible_become=true ansible_become_pass=password
ubuntu2 ansible_become=true ansible_become_pass=password
ubuntu3 ansible_become=true ansible_become_pass=password
```

The `ubuntu` group's counterpart to `04`'s `ansible_user`: rather than connecting directly as `root`, these hosts connect as a regular user and then escalate privileges via `sudo`, controlled by `ansible_become=true` and `ansible_become_pass`. This mirrors a common real-world split — some hosts allow direct root login, others require `sudo`.

---

### 06 — A non-default SSH port (`ansible_port`)

```ini
[centos]
centos1 ansible_user=root ansible_port=2222
centos2 ansible_user=root
centos3 ansible_user=root
```

Adds `ansible_port=2222` to `centos1` only — the other two hosts still connect on the default port 22. A realistic scenario: one host behind a jump/NAT setup, or simply configured differently from its siblings. Keep an eye on `centos1`'s port value; it reappears as the running example for variable precedence all the way through `13`.

---

### 07 — The same port, via inventory shorthand

```ini
[centos]
centos1:2222 ansible_user=root
centos2 ansible_user=root
centos3 ansible_user=root
```

Functionally identical to `06`, but instead of the explicit `ansible_port=2222` key, it uses the `host:port` shorthand directly on the hostname. Both forms are valid and produce the same connection behavior — compare `ansible-inventory --list` output between `06` and `07` to confirm they resolve identically.

---

### 08 — A dedicated control host (`ansible_connection=local`)

```ini
[control]
ubuntu-c ansible_connection=local

[centos]
centos1 ansible_user=root ansible_port=2222
centos2 ansible_user=root
centos3 ansible_user=root

[ubuntu]
ubuntu1 ansible_become=true ansible_become_pass=password
ubuntu2 ansible_become=true ansible_become_pass=password
ubuntu3 ansible_become=true ansible_become_pass=password
```

Introduces a new `[control]` group containing `ubuntu-c`, with `ansible_connection=local` — telling Ansible to execute tasks against this host **without SSH**, directly on the local machine. This models the common lab/course setup where one host is the Ansible control node itself and is also a manageable target (e.g. for local facts or local file operations).

---

### 09 — Range shorthand for sequential hostnames

```ini
[centos]
centos1 ansible_user=root ansible_port=2222
centos[2:3] ansible_user=root

[ubuntu]
ubuntu[1:3] ansible_become=true ansible_become_pass=password
```

Replaces the repeated `centos2` / `centos3` and `ubuntu1` / `ubuntu2` / `ubuntu3` lines with **range syntax**: `centos[2:3]` expands to `centos2` and `centos3`; `ubuntu[1:3]` expands to `ubuntu1` through `ubuntu3`. Both ends of the range are inclusive. A meaningful line-count reduction once an inventory has dozens of sequentially-named hosts.

---

### 10 — Hoisting repeated vars to `[group:vars]`

```ini
[centos]
centos1 ansible_port=2222
centos[2:3]

[centos:vars]
ansible_user=root

[ubuntu]
ubuntu[1:3]

[ubuntu:vars]
ansible_become=true
ansible_become_pass=password
```

The `ansible_user=root` that was repeated on every centos host line in `09` is now declared **once**, in a `[centos:vars]` section — applying to every host in `[centos]` without repeating it per line. Same treatment for `ansible_become`/`ansible_become_pass` under `[ubuntu:vars]`. Note `centos1` keeps its own `ansible_port=2222` as a host var — group vars and host vars coexist, and (as `12` will show explicitly) the more specific one wins.

---

### 11 — Grouping groups with `[group:children]`

```ini
[linux:children]
centos
ubuntu
```

Adds a new parent group, `linux`, whose members are the *groups* `centos` and `ubuntu` rather than individual hosts — declared via the special `:children` suffix. A playbook can now target `hosts: linux` to reach every centos and ubuntu host in one go, without needing to list both groups (or maintain a third, manually-duplicated group of hostnames).

---

### 12 — Variable precedence: host var beats `[all:vars]`

```ini
[all:vars]
ansible_port=1234
```

Adds an `[all:vars]` section — the broadest possible scope, applying `ansible_port=1234` to *every* host in the inventory. But `centos1` still has its own host-level `ansible_port=2222` from `06`/`10`. Run `ansible-inventory --host centos1` here and you'll see `2222`, not `1234` — **a more specific variable (host) always wins over a less specific one (`all`)**, regardless of which was declared last in the file.

---

### 13 — Variable precedence: child-group var beats parent-group var

```ini
[linux:vars]
ansible_port=1234
```

The same idea as `12`, but the broad `ansible_port=1234` is now set on `[linux:vars]` (the parent group from `11`) instead of `[all:vars]`. `centos1`'s own host var (`2222`) still wins, for the same reason as `12` — but this example isolates the *parent-group vs. child-host* case specifically, separate from the *all vs. host* case in `12`, so you can reason about each precedence layer independently.

---

### 14 — The same inventory, in YAML

`ansible.cfg`:
```ini
[defaults]
inventory = hosts.yaml
host_key_checking = False
```

`hosts.yaml`:
```yaml
---
control:
  hosts:
    ubuntu-c:
      ansible_connection: local
centos:
  hosts:
    centos1:
      ansible_port: 2222
    centos2:
    centos3:
  vars:
    ansible_user: root
ubuntu:
  hosts:
    ubuntu1:
    ubuntu2:
    ubuntu3:
  vars:
    ansible_become: true
    ansible_become_pass: password
linux:
  children:
    centos:
    ubuntu:
...
```

Re-expresses the exact structure built up through `11` (control/centos/ubuntu groups, group vars, `linux:children`) in **YAML inventory format** instead of INI. `ansible.cfg` now points `inventory` at `hosts.yaml` explicitly — the plain `hosts` file is still present in the folder for comparison, but is no longer the active inventory. Note the direct structural mapping: `[centos:vars]` → `centos.vars`, `[linux:children]` → `linux.children`, and a host with no vars (like `centos2`) is simply a key with a null value.

---

### 15 — The same inventory again, in JSON

`ansible.cfg`:
```ini
[defaults]
inventory = hosts.json
host_key_checking = False
```

`hosts.json` (excerpt):
```json
{
    "centos": {
        "hosts": {
            "centos1": { "ansible_port": 2222 },
            "centos2": null,
            "centos3": null
        },
        "vars": { "ansible_user": "root" }
    },
    "linux": {
        "children": { "centos": null, "ubuntu": null }
    }
}
```

The third and final equivalent representation: the same data as `14`, this time as **JSON**. All three files — `hosts` (INI), `hosts.yaml`, `hosts.json` — now sit side by side in this folder, and `ansible.cfg` is repointed at `hosts.json`. A good exercise here is running `ansible-inventory --list` against `13`, `14`, and `15` in turn and confirming the resolved output is identical regardless of source format.

---

### 16 — Comparing formats with `ansible-inventory`

All three inventory files (`hosts`, `hosts.yaml`, `hosts.json`) are present again, but `ansible.cfg` points back at the plain INI `hosts` file — the same active format as `01`–`13`. The JSON in this folder is ordered differently from `15`'s (dictionary-insertion order rather than hand-written order), which is a hint that it was produced by asking Ansible itself to emit it rather than typed by hand:

```bash
cd 01-ansible-architecture-and-design/01-inventories/16
ansible-inventory --list > /tmp/generated.json
ansible-inventory --yaml --list > /tmp/generated.yaml
diff /tmp/generated.json hosts.json
```

This is the natural capstone to `14`/`15`: rather than hand-converting between INI, YAML, and JSON, `ansible-inventory` can generate any of the other formats from whichever one is active — useful when migrating a hand-maintained INI inventory to YAML for a new project, for example.

---

### `challenge` — Regroup hosts by odd/even numbering

`challenge/solution/myinventory`:
```ini
[odd]
centos1 ansible_port=2222 ansible_become=True ansible_become_password=password
centos3 ansible_become=True ansible_become_password=password
ubuntu1 ansible_user=root
ubuntu3 ansible_user=root

[even]
centos2 ansible_become=True ansible_become_password=password
ubuntu2 ansible_user=root

[oddsandevens:children]
odd
even
```

A synthesis exercise: rather than grouping purely by OS (`centos`/`ubuntu`) as in every prior example, hosts are regrouped by whether their trailing number is odd or even — `centos1`/`centos3`/`ubuntu1`/`ubuntu3` under `[odd]`, `centos2`/`ubuntu2` under `[even]` — with the correct per-OS connection vars (`ansible_become*` for centos, `ansible_user=root` for ubuntu) still attached to each host individually, since group membership no longer aligns with OS. Both groups are then joined under a `[oddsandevens:children]` parent, mirroring the `linux:children` pattern from `11`. The `ansible.cfg` here points `inventory` at `myinventory` rather than the conventional `hosts` filename, confirming the file name itself is never significant — only what `ansible.cfg` points `inventory` at.

---

### `hands-on` — Practice copies of `01`–`16`

`hands-on/01` through `hands-on/16` are exact duplicates of the corresponding numbered example above (verified identical file-for-file). They exist as a scratch space to redo each exercise yourself — rebuild the inventory from a blank line up to match what's shown, or deliberately break it — without touching the reference copy you might want to compare back against.

---

### `templates` — Blank starting point

`templates/ansible.cfg` and `templates/hosts` are both empty files — a bare-bones starting point for writing an inventory completely from scratch, without even the `[defaults]` / `inventory = hosts` boilerplate pre-filled in.

---

## Summary Table

| # | Concept | Key Feature(s) | Notes |
|---|---------|-----------------|-------|
| 01 | Simplest inventory | `[all]`, one host | |
| 02 | Quiet SSH prompts | `host_key_checking = False` | Carries through every later example |
| 03 | Groups | `[centos]`, `[ubuntu]` | |
| 04 | Host var: user | `ansible_user=root` | Repeated per host |
| 05 | Host vars: become | `ansible_become`, `ansible_become_pass` | Ubuntu counterpart to `04` |
| 06 | Host var: port | `ansible_port=2222` on `centos1` | Referenced again through `13` |
| 07 | Port shorthand | `centos1:2222` | Same effect as `06` |
| 08 | Control host | `[control]`, `ansible_connection=local` | Local, non-SSH execution |
| 09 | Range shorthand | `centos[2:3]`, `ubuntu[1:3]` | Expands to sequential hostnames |
| 10 | Group vars | `[centos:vars]`, `[ubuntu:vars]` | De-duplicates `04`/`05`/`09` |
| 11 | Group of groups | `[linux:children]` | Combines `centos` + `ubuntu` |
| 12 | Precedence: host vs. all | `[all:vars]` | Host var (`2222`) wins |
| 13 | Precedence: host vs. parent group | `[linux:vars]` | Host var (`2222`) still wins |
| 14 | YAML inventory | `hosts.yaml` | Same structure as `11`, different format |
| 15 | JSON inventory | `hosts.json` | Same structure again, third format |
| 16 | Format conversion | `ansible-inventory --list/--yaml` | Generate one format from another |
| `challenge` | Regroup by odd/even | `[odd]`, `[even]`, `:children` | Group membership decoupled from OS |
| `hands-on` | Practice | — | Exact copies of `01`–`16` |
| `templates` | Blank starter | — | Empty `ansible.cfg` + `hosts` |

---

## Quick-Reference: Inventory Variable Precedence (Lowest → Highest, as demonstrated here)

| Source | Example(s) | Notes |
|---|---|---|
| `[all:vars]` | `12` | Broadest possible scope |
| `[parent_group:vars]` (e.g. `[linux:vars]`) | `13` | More specific than `all`, less specific than the group itself |
| `[group:vars]` (e.g. `[centos:vars]`) | `10` | More specific than a parent group |
| Host variable (inline, e.g. `centos1 ansible_port=2222`) | `04`–`09`, `12`, `13` | **Always wins** — the most specific scope always overrides broader ones |

---

## Quick-Reference: Inventory Formats

| Format | File | First appears | Notes |
|---|---|---|---|
| INI | `hosts` | `01` | Terse, hand-editing-friendly; the default format |
| YAML | `hosts.yaml` | `14` | More explicit nesting; easier to templatize/generate |
| JSON | `hosts.json` | `15` | Machine-generated/consumed; what dynamic inventory scripts typically emit |

All three are equivalent once parsed — `ansible-inventory --list` resolves them to the same in-memory structure regardless of source format.

---

## Suggested Next Steps

1. Run `ansible-inventory --graph` in `03`, `08`, and `11` back to back to watch the group tree grow from flat groups to a nested `linux` parent.
2. In `12` and `13`, run `ansible-inventory --host centos1` and confirm the resolved `ansible_port` is `2222` in both cases — then try commenting out `centos1`'s own `ansible_port` to see the broader scope "win" once nothing more specific overrides it.
3. Hand-convert `01`–`13`'s final INI inventory to YAML yourself before reading `14`, then compare your version against it.
4. Attempt the `challenge` folder unaided — group hosts by a criterion unrelated to OS, while still applying the correct OS-specific connection vars per host — before checking it against `challenge/solution`.
5. Work through `hands-on/01`–`16` from memory, without looking back at `01`–`16`, as a final comprehension check before moving on to `02-modules`.
