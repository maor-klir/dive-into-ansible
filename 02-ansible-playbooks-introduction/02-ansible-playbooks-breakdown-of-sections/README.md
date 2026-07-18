# 02 — Ansible Playbooks: Breakdown of Sections

This README is a learning companion for this section of the *Dive Into Ansible* course. It walks through each numbered example (`01`–`07`) in this folder, in the order you're meant to learn them, plus the `challenge` and `hands-on` folders, explaining how a playbook is built up section by section: `hosts`, `vars`, `tasks`, and `handlers`.

The goal is to watch a single MOTD (message of the day) playbook grow in complexity one section at a time, so you build a clear mental model of what each playbook section is for before combining them all.

---

## How to use this folder while learning

Each subfolder `01`–`07` is a **self-contained mini-lab** with its own `ansible.cfg`, `hosts` inventory, and a `motd_playbook.yaml`. `cd` into any one and run it independently:

```bash
cd 02-ansible-playbooks-introduction/02-ansible-playbooks-breakdown-of-sections/02
ansible-playbook motd_playbook.yaml
```

---

## Walking Through the Examples

### 01 — The empty skeleton

```yaml
---
-

  # Hosts: where our play will run and options it will run with

  # Vars: variables that will apply to the play, on all target systems

  # Tasks: the list of tasks that will be executed within the play, this section
  #       can also be used for pre and post tasks

  # Handlers: the list of handlers that are executed as a notify key from a task

  # Roles: list of roles to be imported into the play

...
```

A playbook with **no actual tasks** — just comments describing each of the five sections a play can contain: `hosts`, `vars`, `tasks`, `handlers`, `roles`. This is the mental map you'll fill in over the next six examples. Note the top-level `-`: a playbook file is a YAML **list of plays**, each play being a dictionary.

---

### 02 — Adding `hosts` and a first `tasks` entry

```yaml
---
-
  hosts: centos
  user: root

  tasks:
    - name: Configure a MOTD (message of the day)
      copy:
        src: centos_motd
        dest: /etc/motd

...
```

Introduces the `hosts:` key (targeting the `centos` inventory group), `user:` (connect as `root`), and a first real task using the `copy` module to push a static `centos_motd` file to `/etc/motd`.

---

### 03 — Turning off fact gathering

```yaml
---
-
  hosts: centos
  user: root
  gather_facts: False

  tasks:
    - name: Configure a MOTD (message of the day)
      copy:
        src: centos_motd
        dest: /etc/motd

...
```

Same task as `02`, but adds `gather_facts: False`. Since this play doesn't reference any Ansible facts, disabling fact gathering makes the run noticeably faster — a good habit for playbooks that don't need `ansible_*` variables.

---

### 04 — Inline `content` instead of a source file

```yaml
---
-
  hosts: centos
  user: root
  gather_facts: False

  tasks:
    - name: Configure a MOTD (message of the day)
      copy:
        content: Welcome to CentOS Linux - Ansible Rocks
        dest: /etc/motd

...
```

Replaces the `src:` (external file) approach from `02`/`03` with `content:`, writing the string directly inline in the playbook — no separate `centos_motd` file needed.

---

### 05 — Introducing `vars`

```yaml
---
-
  hosts: centos
  user: root
  gather_facts: False

  vars:
    motd: "Welcome to CentOS Linux - Ansible Rocks\n"

  tasks:
    - name: Configure a MOTD (message of the day)
      copy:
        content: "{{ motd }}"
        dest: /etc/motd

...
```

The `vars:` section finally gets used: the MOTD string is pulled out into a `motd` variable and referenced via `{{ motd }}` in the task. Note the explicit `\n` — since this is inside double quotes, it renders as an actual newline (see `01-yaml/04` for why that matters).

---

### 06 — Introducing `handlers`

```yaml
---
-
  hosts: centos
  user: root
  gather_facts: False

  vars:
    motd: "Welcome to CentOS Linux - Ansible Rocks\n"

  tasks:
    - name: Configure a MOTD (message of the day)
      copy:
        content: "{{ motd }}"
        dest: /etc/motd
      notify: MOTD changed

  handlers:
    - name: MOTD changed
      debug:
        msg: The MOTD was changed

...
```

Adds `notify: MOTD changed` to the task, which triggers the `MOTD changed` handler **only if the task actually reports a change** (i.e., the file content was different from what's already on disk). This is the core Ansible pattern: handlers run once, at the end of the play, only when notified — and only once even if notified multiple times.

---

### 07 — Multiple hosts, conditional tasks with `when`

```yaml
---
-
  hosts: linux

  vars:
    motd_centos: "Welcome to CentOS Linux - Ansible Rocks\n"
    motd_ubuntu: "Welcome to Ubuntu Linux - Ansible Rocks\n"

  tasks:
    - name: Configure a MOTD (message of the day)
      copy:
        content: "{{ motd_centos }}"
        dest: /etc/motd
      notify: MOTD changed
      when: ansible_distribution == "CentOS"

    - name: Configure a MOTD (message of the day)
      copy:
        content: "{{ motd_ubuntu }}"
        dest: /etc/motd
      notify: MOTD changed
      when: ansible_distribution == "Ubuntu"

  handlers:
    - name: MOTD changed
      debug:
        msg: The MOTD was changed

...
```

Targets the broader `linux` group (both CentOS and Ubuntu hosts) with two candidate tasks, each guarded by a `when:` condition checking the `ansible_distribution` fact. Only the task matching the current host's distribution actually runs — note that `gather_facts` is *not* disabled here, since `when: ansible_distribution == ...` depends on facts being collected.

---

### `challenge` — Ubuntu-specific MOTD script via `update-motd.d`

```yaml
---
-
  hosts: ubuntu
  tasks:
    - name: Copy MOTD file
      copy:
        src: 60-ansible-motd
        dest: /etc/update-motd.d/60-ansible-motd
        mode: preserve
      notify: Debug, if there is a change
  handlers:
    - name: Debug, if there is a change
      debug:
        msg: Change took place

...
```

A more realistic Ubuntu pattern: rather than writing `/etc/motd` directly, Ubuntu generates its MOTD dynamically via executable scripts in `/etc/update-motd.d/`. This copies a custom script (`60-ansible-motd`) into that directory with `mode: preserve` (keep the source file's existing permissions — important since the script needs to be executable):

```sh
#!/bin/sh

echo "At the third stroke, the time sponsored by Ansible will be $(date +%X) ... beep beep beep\n"
```

---

### `hands-on` — Practice folders

Contains `01/` and `02/` subfolders intended for you to build your own MOTD playbooks from scratch, applying everything from `01`–`07` and the `challenge` without a pre-built solution to reference.

---

## Summary Table

| # | Concept | Key Feature(s) | Notes |
|---|---------|-----------------|-------|
| 01 | Empty skeleton | Comments only | Maps out `hosts`/`vars`/`tasks`/`handlers`/`roles` |
| 02 | `hosts` + first task | `hosts:`, `user:`, `copy` (src) | |
| 03 | Disable fact gathering | `gather_facts: False` | Faster runs when facts aren't needed |
| 04 | Inline content | `copy` (content) | No external file needed |
| 05 | Variables | `vars:`, `{{ motd }}` | |
| 06 | Handlers | `notify:`, `handlers:` | Runs once, only if notified |
| 07 | Conditional tasks | `when:`, `ansible_distribution` | Requires fact gathering |
| `challenge` | Ubuntu `update-motd.d` script | `mode: preserve` | Dynamic MOTD instead of static file |
| `hands-on` | Practice | — | Build your own from scratch |

---

## `ansible.cfg` and `hosts` Across This Folder

Examples `01`–`07` and `challenge` all share the same `ansible.cfg` and `hosts` inventory files, keeping the focus purely on the playbook content itself as it evolves example to example.

---

## Suggested Next Steps

1. Re-run `06` and `07` twice in a row — notice the handler only fires (and prints `The MOTD was changed`) the *first* time, since the second run finds no actual change to notify.
2. Compare `04` and `05` side by side to internalize why pulling repeated values into `vars` makes playbooks easier to maintain.
3. Try adding a third `when:` branch to `07` for a distribution not covered (e.g., Debian), and see what happens on a host that doesn't match either condition.
4. Attempt the `hands-on/01` and `hands-on/02` folders unaided before moving on to `03-ansible-playbooks-variables`.
