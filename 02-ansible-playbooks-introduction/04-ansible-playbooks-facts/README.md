# 04 — Ansible Playbooks: Facts

This README is a learning companion for this section of the *Dive Into Ansible* course. It walks through each numbered example (`01`–`06`) in this folder, in the order you're meant to learn them, explaining what each one introduces and why it's built that way.

The goal isn't just to show *what* each example does, but *why* it's built that way — including how Ansible's automatically-gathered **facts** differ from **custom facts** that you write yourself, and how the location where those custom facts live evolves across the examples.

---

## How to use this folder while learning

Each subfolder `01`–`06` is a **self-contained mini-lab**: its own `ansible.cfg`, `hosts` inventory, and a single `facts_playbook.yaml` (plus, from `05` onward, the custom fact scripts themselves). That means you can `cd` into any one of them and run it independently:

```bash
cd 02-ansible-playbooks-introduction/04-ansible-playbooks-facts/03
ansible-playbook facts_playbook.yaml
```

This is a great pattern to copy for your own experiments later — one concept, one folder, one playbook, run independently of everything else.

---

## Background: What Are Facts?

**Facts** are variables that are automatically discovered by Ansible about a remote host — things like its IP addresses, OS family, hostname, memory, and disks. They're gathered by the `setup` module, which runs implicitly at the start of every play unless `gather_facts: false` is set.

Facts are referenced like any other variable, e.g. `{{ ansible_default_ipv4.address }}` — but note the `ansible_` prefix, which distinguishes built-in facts from variables you define yourself.

Beyond the built-in facts, Ansible also supports **custom facts** — small scripts or data files that live in `/etc/ansible/facts.d/` (or another directory pointed to by `fact_path`) on the *managed* host. Their output is automatically merged into a special variable, `ansible_local`, the next time facts are gathered. This folder builds up from basic built-in facts to fully custom, host-supplied facts step by step.

---

## Walking Through the Examples

### 01 — A basic built-in fact

```yaml
tasks:
  - name: Show IP Address
    debug:
      msg: "{{ ansible_default_ipv4.address }}"
```
[Source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/04-ansible-playbooks-facts/01/facts_playbook.yaml)

The simplest possible fact usage: printing `ansible_default_ipv4.address`, a nested fact describing the primary IPv4 address of the default route interface. Facts are gathered automatically before the `tasks:` section runs, so no extra setup is required — this "just works" on any host Ansible can connect to.

---

### 02 — Same fact, reused as a baseline

```yaml
tasks:
  - name: Show IP Address
    debug:
      msg: "{{ ansible_default_ipv4.address }}"
```
[Source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/04-ansible-playbooks-facts/02/facts_playbook.yaml)

Functionally identical to `01`, but this folder introduces an (empty, at this stage) `templates/` directory — a hint of what's coming in `05`, where template-like fact scripts will actually be used. Treat this one as a checkpoint before custom facts are introduced in `03`.

---

### 03 — Introducing custom facts via `ansible_local`

```yaml
tasks:
  - name: Show IP Address
    debug:
      msg: "{{ ansible_default_ipv4.address }}"

  - name: Show Custom Fact 1
    debug:
      msg: "{{ ansible_local.getdate1.date }}"

  - name: Show Custom Fact 2
    debug:
      msg: "{{ ansible_local.getdate2.date.date }}"
```
[Source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/04-ansible-playbooks-facts/03/facts_playbook.yaml)

This is where **custom facts** first appear, via the `ansible_local` variable. `ansible_local` is a dictionary keyed by the *filename* of each custom fact source (without its extension) — here `getdate1` and `getdate2` — assuming those fact files already exist on the target host under `/etc/ansible/facts.d/`. Notice the two facts have slightly different shapes: `ansible_local.getdate1.date` is a plain value, while `ansible_local.getdate2.date.date` is nested one level deeper — the exact shape depends entirely on how each fact script formats its output (more on that in `05`).

---

### 04 — Custom facts via `hostvars`

```yaml
tasks:
  - name: Show IP Address
    debug:
      msg: "{{ ansible_default_ipv4.address }}"

  - name: Show Custom Fact 1
    debug:
      msg: "{{ ansible_local.getdate1.date }}"

  - name: Show Custom Fact 2
    debug:
      msg: "{{ ansible_local.getdate2.date.date }}"

  - name: Show Custom Fact 1 in hostvars
    debug:
      msg: "{{ hostvars[ansible_hostname].ansible_local.getdate1.date }}"

  - name: Show Custom Fact 2 in hostvars
    debug:
      msg: "{{hostvars[ansible_hostname].ansible_local.getdate2.date.date }}"
```
[Source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/04-ansible-playbooks-facts/04/facts_playbook.yaml)

Adds two new tasks that access the *exact same* custom facts, but through `hostvars[ansible_hostname]` instead of directly. `hostvars` is a dictionary of all variables (including facts) known for *every* host in the play, indexed by hostname — this is the mechanism you'd use to reference a fact **from a different host** than the one currently executing (e.g., referencing a database server's IP while configuring an app server). Here it's shown against the *current* host purely to demonstrate that `ansible_local.x` and `hostvars[ansible_hostname].ansible_local.x` resolve to the same value when run against yourself.

---

### 05 — Deploying the custom fact scripts, end-to-end

```yaml
tasks:
  - name: Make Facts Dir
    file:
      path: /etc/ansible/facts.d
      recurse: yes
      state: directory

  - name: Copy Fact 1
    copy:
      src: /etc/ansible/facts.d/getdate1.fact
      dest: /etc/ansible/facts.d/getdate1.fact
      mode: 0755

  - name: Copy Fact 2
    copy:
      src: /etc/ansible/facts.d/getdate2.fact
      dest: /etc/ansible/facts.d/getdate2.fact
      mode: 0755

  - name: Refresh Facts
    setup:

  - name: Show IP Address
    debug:
      msg: "{{ ansible_default_ipv4.address }}"

  - name: Show Custom Fact 1
    debug:
      msg: "{{ ansible_local.getdate1.date }}"

  - name: Show Custom Fact 2
    debug:
      msg: "{{ ansible_local.getdate2.date.date }}"

  - name: Show Custom Fact 1 in hostvars
    debug:
      msg: "{{ hostvars[ansible_hostname].ansible_local.getdate1.date }}"

  - name: Show Custom Fact 2 in hostvars
    debug:
      msg: "{{hostvars[ansible_hostname].ansible_local.getdate2.date.date }}"
```
[Playbook source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/04-ansible-playbooks-facts/05/facts_playbook.yaml)

This is the example that finally makes `03` and `04` work from scratch, rather than assuming the fact files already exist. It:

1. Creates `/etc/ansible/facts.d` on the target with the `file` module (`state: directory`, `recurse: yes`).
2. Copies two executable fact scripts into that directory with the `copy` module.
3. Re-runs the `setup` module explicitly (`Refresh Facts`) to force Ansible to re-gather facts — **this step is essential**, since facts are normally only gathered once, at the start of the play, before these files existed.
4. Then repeats the same `debug` tasks from `03` and `04` to show the now-populated custom facts.

The two fact scripts themselves are simple executables that print JSON (or INI-style) output to stdout — Ansible custom facts can be either a static `.fact` file containing JSON/INI, or an **executable script** whose stdout is parsed the same way:

`templates/getdate1.fact` — outputs a single JSON object:
```bash
#!/bin/bash
echo {\"date\" : \"`date`\"}
```
[Source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/04-ansible-playbooks-facts/05/templates/getdate1.fact)

`templates/getdate2.fact` — outputs INI-style key/value pairs under a `[date]` section:
```bash
#!/bin/bash
echo [date]
echo date=`date`
```
[Source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/04-ansible-playbooks-facts/05/templates/getdate2.fact)

This directly explains the shape difference noticed back in `03`: `getdate1.fact` emits flat JSON, so its value is available at `ansible_local.getdate1.date`. `getdate2.fact` emits an INI `[date]` section, so Ansible's custom-facts parser (which supports both JSON and INI) wraps it in an extra `date` key, making it available at `ansible_local.getdate2.date.date`.

**Note:** both `copy` tasks use the *same* path for `src` and `dest` (`/etc/ansible/facts.d/...`) — meaning the fact files are expected to already exist locally on the control node at that absolute path, rather than being pulled from this folder's own `templates/` directory via a relative path. Keep an eye on this when you compare it to `06`, where that changes.

---

### 06 — Custom facts as a non-root user, with an explicit `fact_path`

```yaml
tasks:
  - name: Make Facts Dir
    file:
      path: /home/ansible/facts.d
      recurse: yes
      state: directory
      owner: ansible

  - name: Copy Fact 1
    copy:
      src: facts.d/getdate1.fact
      dest: /home/ansible/facts.d/getdate1.fact
      owner: ansible
      mode: 0755

  - name: Copy Fact 2
    copy:
      src: facts.d/getdate2.fact
      dest: /home/ansible/facts.d/getdate2.fact
      owner: ansible
      mode: 0755

  - name: Reload Facts
    setup:
      fact_path: /home/ansible/facts.d

  - name: Show IP Address
    debug:
      msg: "{{ ansible_default_ipv4.address }}"

  - name: Show Custom Fact 1
    debug:
      msg: "{{ ansible_local.getdate1.date }}"

  - name: Show Custom Fact 2
    debug:
      msg: "{{ ansible_local.getdate2.date.date }}"

  - name: Show Custom Fact 1 in hostvars
    debug:
      msg: "{{ hostvars[ansible_hostname].ansible_local.getdate1.date }}"

  - name: Show Custom Fact 2 in hostvars
    debug:
      msg: "{{hostvars[ansible_hostname].ansible_local.getdate2.date.date }}"
```
[Playbook source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/04-ansible-playbooks-facts/06/facts_playbook.yaml) ·
[getdate1.fact](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/04-ansible-playbooks-facts/06/facts.d/getdate1.fact) ·
[getdate2.fact](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/04-ansible-playbooks-facts/06/facts.d/getdate2.fact)

Refines `05` in two important ways worth noticing as you compare them side by side:

1. **Custom facts don't have to live under `/etc/ansible/facts.d`.** The default facts directory requires root/sudo to write to on most systems. Here, the directory is instead created under `/home/ansible/facts.d`, owned by the `ansible` user (`owner: ansible` on both the `file` and `copy` tasks) — no elevated privileges required for this part of the play.
2. **`setup:` now takes an explicit `fact_path` argument** (`fact_path: /home/ansible/facts.d`), telling Ansible where to look for custom facts *instead of* the default `/etc/ansible/facts.d`. Without this, Ansible would still gather facts from the default location — the same location `05` used — and would miss the new one entirely.

Also notice the `copy` tasks now reference a *relative* `src: facts.d/getdate1.fact`, resolved against this folder's own `facts.d/` subdirectory on the **control node** — a more portable and realistic pattern than `05`'s same-path-on-both-ends approach, since the fact scripts now genuinely travel from this repository to the managed host rather than being assumed to already exist there.

The fact scripts themselves are unchanged from `05` — same JSON vs. INI-style output, same resulting `ansible_local` shapes.

---

## Summary Table

| # | Concept | Key Feature(s) | File(s) | Notes |
|---|---------|-----------------|---------|-------|
| 01 | Built-in fact | `ansible_default_ipv4.address` | `01/facts_playbook.yaml` | Facts gathered automatically at play start |
| 02 | Built-in fact (baseline) | Same as `01` | `02/facts_playbook.yaml` | Introduces (empty) `templates/` dir, foreshadowing `05` |
| 03 | Custom facts via `ansible_local` | `ansible_local.<factname>` | `03/facts_playbook.yaml` | Assumes fact files already exist on target |
| 04 | Custom facts via `hostvars` | `hostvars[host].ansible_local.<factname>` | `04/facts_playbook.yaml` | Same facts, accessed cross-host style |
| 05 | Deploying + refreshing custom facts | `file`, `copy`, `setup` (re-run) | `05/facts_playbook.yaml`, `05/templates/*.fact` | Default `/etc/ansible/facts.d` path; JSON vs INI fact output shapes |
| 06 | Custom facts as non-root user | `owner:`, `fact_path:` on `setup` | `06/facts_playbook.yaml`, `06/facts.d/*.fact` | Custom `/home/ansible/facts.d` path; relative `src` from repo |

---

## Quick-Reference: Custom Fact File Formats

| Format | Example | Resulting `ansible_local` shape |
|---|---|---|
| Flat JSON object | `{"date": "..."}` | `ansible_local.<name>.date` |
| INI with a section | `[date]`<br>`date=...` | `ansible_local.<name>.date.date` |

**Rule of thumb for when you're writing your own custom facts:** the extra nesting for INI-style output comes from the section header (`[date]`) becoming a key itself — flatten your script's output to plain JSON if you want to avoid that extra level.

---

## Quick-Reference: Where Custom Facts Are Gathered From

| Item | Detail |
|---|---|
| Default location | `/etc/ansible/facts.d/` on the **managed** host |
| Override | `fact_path` argument to the `setup` module |
| File types supported | Static `.fact` files (JSON or INI content) **or** executable scripts whose stdout is JSON/INI |
| When are they (re-)read | At the start of the play (implicit `gather_facts`), or whenever `setup:` is called explicitly |
| Variable they populate | `ansible_local`, keyed by filename (without extension) |
| Permissions gotcha | Default path usually needs root/sudo to write to — `06` shows the workaround (custom `fact_path` + non-root `owner`) |

---

## Suggested Next Steps

Once you're comfortable with everything above, a few good ways to reinforce this section before moving on in the course:

1. Write a third custom fact script of your own — try returning a nested JSON object (not just a flat string) and see how the resulting `ansible_local` path changes.
2. Deliberately skip the `Refresh Facts` / `Reload Facts` task in `05` or `06` and re-run the playbook, to see firsthand why re-gathering facts is necessary after copying new fact files.
3. Compare `05` and `06` line-by-line, focusing purely on the `file`/`copy`/`setup` differences, to internalize exactly what changes when moving custom facts out of the default root-owned path.
4. Try referencing a custom fact from a genuinely different host in your inventory using `hostvars`, rather than referencing your own host as `04` does — this is the real-world use case that task is demonstrating.
