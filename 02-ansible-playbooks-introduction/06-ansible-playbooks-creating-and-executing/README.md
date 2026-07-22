# 06 — Ansible Playbooks: Creating and Executing

This README is a learning companion for this section of the *Dive Into Ansible* course. Unlike the previous folders in this section, this one isn't a set of independent numbered concepts — it's a single hands-on exercise: build a real playbook that installs and configures Nginx, then progressively enhance it. `solution/00`–`09` are ten snapshots of that same playbook, each one small step further along than the last.

The goal isn't just to show *what* each snapshot does, but *why* it's built that way — this is where everything from `01`–`05` (YAML, variables, facts, Jinja2 templating) gets combined into one working example for the first time.

---

## How to use this folder while learning

This folder has a different layout to the ones before it:

- **`template/`** — an empty playbook skeleton: just the commented section headers (`Hosts`, `Vars`, `Tasks`, `Handlers`) and no actual content. This is the starting point you're meant to build from yourself, following along with the course.
- **`solution/00`–`09`** — ten complete, runnable snapshots of the *same* playbook, each one showing the next step of the exercise. `solution/00` is identical to `template/` (the empty starting point); `solution/09` is the finished result.

Each `solution/NN` folder is self-contained — its own `ansible.cfg`, `hosts` inventory, `group_vars`, `host_vars`, `vars`, `files`, and `templates` — so you can `cd` into any one of them and run it in isolation:

```bash
cd 02-ansible-playbooks-introduction/06-ansible-playbooks-creating-and-executing/solution/05
ansible-playbook nginx_playbook.yaml
```

The recommended way to work through this folder: open `template/nginx_playbook.yaml` in an editor, and build it up yourself one piece at a time, checking your work against `solution/01`, then `02`, and so on. Diffing consecutive solution steps is a great way to see exactly what changed and why:

```bash
diff solution/03/nginx_playbook.yaml solution/04/nginx_playbook.yaml
```

---

## Walking Through the Steps

### 00 — The empty skeleton

Identical to `template/nginx_playbook.yaml` — just the section-header comments, no `hosts:`, `tasks:`, or anything else filled in. This is the baseline everything else in this folder builds on.

---

### 01 — First host targeting and first task

```yaml
hosts: linux

tasks:
  - name: Install EPEL
    yum:
      name: epel-release
      update_cache: yes
      state: latest
    when: ansible_distribution == 'CentOS'
```
[Source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/06-ansible-playbooks-creating-and-executing/solution/01/nginx_playbook.yaml)

The play finally targets something: `hosts: linux`, the group defined in this folder's `hosts` inventory (`[linux:children]` containing both `centos` and `ubuntu`). The first task installs the EPEL repository — needed on CentOS before Nginx is available via `yum` — guarded by a `when:` conditional against the `ansible_distribution` fact so it's skipped entirely on Ubuntu.

---

### 02 — Distribution-specific Nginx installation

```yaml
- name: Install Nginx CentOS
  yum:
    name: nginx
    update_cache: yes
    state: latest
  when: ansible_distribution == 'CentOS'

- name: Install Nginx Ubuntu
  apt:
    name: nginx
    update_cache: yes
    state: latest
  when: ansible_distribution == 'Ubuntu'
```
[Source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/06-ansible-playbooks-creating-and-executing/solution/02/nginx_playbook.yaml)

Two separate tasks install Nginx: `yum` for CentOS, `apt` for Ubuntu, each with its own `when:` guard — mirroring the pattern from `01`. Functional, but repetitive; every package you install this way needs two near-identical tasks.

---

### 03 — Collapsing to the `package` module

```yaml
- name: Install Nginx
  package:
    name: nginx
    state: latest
```
[Source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/06-ansible-playbooks-creating-and-executing/solution/03/nginx_playbook.yaml)

Replaces both distribution-specific tasks from `02` with a single task using the `package` module — Ansible's OS-agnostic wrapper that delegates to `yum`, `apt`, or whatever the target's native package manager is, based on gathered facts. No `when:` needed at all. Worth noticing: `Install EPEL` from `01` is *not* rewritten this way, since EPEL itself is CentOS-specific — `package` only helps once the same package name is genuinely installable the same way everywhere.

---

### 04 — Starting the service

```yaml
- name: Restart nginx
  service:
    name: nginx
    state: restarted
```
[Source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/06-ansible-playbooks-creating-and-executing/solution/04/nginx_playbook.yaml)

Installing the package doesn't start it. The `service` module, `state: restarted`, guarantees Nginx is actually running after this task — using `restarted` rather than `started` means the task also picks up config changes on every run, not just the first one.

---

### 05 — Verifying it worked, with a handler

```yaml
tasks:
  - name: Restart nginx
    service:
      name: nginx
      state: restarted
    notify: Check HTTP Service

handlers:
  - name: Check HTTP Service
    uri:
      url: http://{{ ansible_default_ipv4.address }}
      status_code: 200
```
[Source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/06-ansible-playbooks-creating-and-executing/solution/05/nginx_playbook.yaml)

Introduces the `handlers:` section from the playbook skeleton for the first time. `notify: Check HTTP Service` on the restart task queues the matching handler to run — but only once, at the end of the play, and only because the task it's attached to reported a change. The handler itself uses the `uri` module to make an actual HTTP request back to the host (`ansible_default_ipv4.address`, a fact from `04-ansible-playbooks-facts`) and asserts a `200` response, which fails the play if Nginx isn't actually serving traffic. This is the first point in the exercise where "did the task run" and "did it actually work" are checked separately.

---

### 06 — Serving custom content via `template`

```yaml
- name: Template index.html-base.j2 to index.html on target
  template:
    src: index.html-base.j2
    dest: "{{ nginx_root_location }}/index.html"
    mode: 0644
```
[Source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/06-ansible-playbooks-creating-and-executing/solution/06/nginx_playbook.yaml)

Brings in the `template` module from `05-templating-with-jinja2`, rendering `templates/index.html-base.j2` (a static course landing page, no Jinja2 logic yet) to `index.html` in Nginx's web root. Notice `dest` isn't hardcoded — `{{ nginx_root_location }}` is a variable defined per-OS in `group_vars/centos` (`/usr/share/nginx/html`) and `group_vars/ubuntu` (`/var/www/html`), since the two distributions use different default web roots.

---

### 07 — `ansible_managed` in the rendered output

```jinja2
<h1>Dive Into Ansible</h1>
<p>{{ ansible_managed }}</p>
```
[Template source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/06-ansible-playbooks-creating-and-executing/solution/07/templates/index.html-ansible_managed.j2)

Swaps in `index.html-ansible_managed.j2`, which adds one line printing the special `ansible_managed` variable. This isn't a fact or a playbook variable — it's a template-only string built from the `ansible_managed` setting added to `ansible.cfg` in this step:

```ini
ansible_managed = Managed by Ansible - file:{file} - host:{host} - uid:{uid}
```

Ansible substitutes `{file}`, `{host}`, and `{uid}` automatically whenever `ansible_managed` is referenced inside a template rendered by the `template` module — a handy, built-in way to stamp generated files with *where* they came from, useful for anyone who stumbles on the file later and wonders whether it's safe to hand-edit.

---

### 08 — Per-distribution logos with `vars_files` and `if`/`elif`

```yaml
vars_files:
  - vars/logos.yaml
```
```jinja2
<img src="
    {%- if ansible_distribution == 'CentOS' -%}
    {{ centos_logo }}
    {%- elif ansible_distribution == 'Ubuntu' -%}
    {{ ubuntu_logo }}
    {%- endif %}
" class="intro" alt="mainpic">
```
[Playbook source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/06-ansible-playbooks-creating-and-executing/solution/08/nginx_playbook.yaml) ·
[Template source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/06-ansible-playbooks-creating-and-executing/solution/08/templates/index.html-logos.j2)

Two things land together here. First, `vars_files:` loads `vars/logos.yaml` — a file holding two large base64-encoded image strings, `centos_logo` and `ubuntu_logo` — into the play's variables, rather than defining them inline. Second, `index.html-logos.j2` uses the `if`/`elif` Jinja2 syntax from `05-templating-with-jinja2` to pick the right logo based on `ansible_distribution`, so CentOS hosts render the CentOS logo and Ubuntu hosts render the Ubuntu logo from the *same* template file and the *same* task. Notice the trailing `-%}` on each tag — the whitespace-control lesson from `05` applies here too, keeping the rendered `<img src="...">` free of stray blank lines around the base64 string.

---

### 09 — The finished page: an easter egg and a deployed game

```yaml
- name: Template index.html-easter_egg.j2 to index.html on target
  template:
    src: index.html-easter_egg.j2
    dest: "{{ nginx_root_location }}/index.html"
    mode: 0644

- name: Install unzip
  package:
    name: unzip
    state: latest

- name: Unarchive playbook stacker game
  unarchive:
    src: playbook_stacker.zip
    dest: "{{ nginx_root_location }}"
    mode: 0755
```
[Playbook source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/06-ansible-playbooks-creating-and-executing/solution/09/nginx_playbook.yaml) ·
[Template source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/06-ansible-playbooks-creating-and-executing/solution/09/templates/index.html-easter_egg.j2)

The final version of the playbook. `index.html-easter_egg.j2` is the same as `08`'s template, but wraps the per-distribution logo image in a link to `/playbook_stacker` — a small game hidden on the site. Two new tasks make that link work: `unzip` is installed (a dependency for the next task, not for Nginx itself), and the `unarchive` module extracts `files/playbook_stacker.zip` straight into the web root, so the game becomes reachable the moment the play finishes. This is the first time the exercise deploys something other than a rendered template — a good example of `unarchive` handling both the transfer and the extraction of an archive in one task, without a separate "copy the zip, then unzip it" pair of steps.

---

## Summary Table

| # | Concept | Key Feature(s) | Notes |
|---|---------|-----------------|-------|
| 00 | Empty skeleton | — | Identical to `template/`, the exercise's starting point |
| 01 | Host targeting + first task | `hosts:`, `when:`, `yum` | EPEL only needed/installed on CentOS |
| 02 | Distribution-specific installs | `yum` / `apt` + `when:` | Two near-duplicate tasks |
| 03 | Cross-platform install | `package` module | Collapses `02`'s two tasks into one |
| 04 | Starting the service | `service`, `state: restarted` | Package installed doesn't mean running |
| 05 | Verifying with a handler | `notify:`, `handlers:`, `uri` module | First "did it actually work" check |
| 06 | Custom content | `template` module | Renders a static `.j2` page to `nginx_root_location` |
| 07 | File provenance | `ansible_managed` (`ansible.cfg` + template) | Auto-populated file/host/uid string |
| 08 | Per-OS logos | `vars_files`, Jinja2 `if`/`elif` | Same template picks the right logo per distro |
| 09 | Final page + deployed game | `unarchive`, `unzip` | Easter egg link + Playbook Stacker game |

---

## Quick-Reference: Supporting Files (consistent across `solution/00`–`09`)

| File | Purpose |
|---|---|
| `ansible.cfg` | `inventory = hosts`, `host_key_checking = False`; gains `ansible_managed` from `07` onward |
| `hosts` | Defines `control`, `centos` (`centos[1:3]`), `ubuntu` (`ubuntu[1:3]`), and `linux` as the parent group of both |
| `group_vars/centos` | `ansible_user: root`; `nginx_root_location: /usr/share/nginx/html` |
| `group_vars/ubuntu` | `ansible_become: true` + `ansible_become_pass`; `nginx_root_location: /var/www/html` |
| `vars/logos.yaml` | `centos_logo` / `ubuntu_logo` base64 image strings, loaded via `vars_files` from `08` onward |
| `files/playbook_stacker.zip` | The game archive deployed by `unarchive` in `09` |
| `templates/index.html-*.j2` | One template per stage of the exercise — `base`, `ansible_managed`, `logos`, `easter_egg` — kept side by side so you can diff them directly |

**Rule of thumb while comparing steps:** `nginx_root_location` is why the same `template`/`unarchive` tasks work unmodified on both CentOS and Ubuntu — the path itself lives in `group_vars`, not in the task.

---

## Deep Dive: Why `files/` and `templates/` Are "Magic" Directory Names

Look closely at the two tasks that pull content in from this repository onto the target host:

```yaml
- name: Template index.html-easter_egg.j2 to index.html on target
  template:
    src: index.html-easter_egg.j2
    dest: "{{ nginx_root_location }}/index.html"
    mode: 0644

- name: Unarchive playbook stacker game
  unarchive:
    src: playbook_stacker.zip
    dest: "{{ nginx_root_location }}"
    mode: 0755
```

Neither `src:` includes a directory. Yet `index.html-easter_egg.j2` is sitting in `solution/09/templates/`, and `playbook_stacker.zip` is sitting in `solution/09/files/`. Nothing in `ansible.cfg`, `hosts`, or anywhere else in this folder tells Ansible to look there. This section explains where that behavior actually comes from, because it trips people up the first time they reorganize a playbook's directory layout.

### It's not configuration — it's a hardcoded search path

When a task uses a module that reads a file from the **control node** (`copy`, `template`, `script`, `unarchive` with `remote_src: false`, and a handful of others), and the `src:` value is a bare relative path with no directory component, Ansible resolves it using a fixed internal search routine — conceptually similar to `_find_needle()` in Ansible's action-plugin base class. That routine checks a short, hardcoded list of conventional subdirectory names relative to the playbook (or role) directory:

- `files/` — checked by `copy`, `unarchive`, `script`, and similar "transfer a file" modules
- `templates/` — checked specifically by the `template` module
- `vars/` and `tasks/` — checked by other Ansible constructs (`vars_files`, `include_tasks`, etc.), following the same underlying principle

There is **no setting** in `ansible.cfg` that names these directories, and no line in the playbook that says "look in `files/` for archives" or "look in `templates/` for Jinja2 files." The names `files` and `templates` are baked into Ansible core itself — this is exactly why every folder in this course (and in most Ansible projects you'll encounter) uses those exact directory names: not by convention alone, but because Ansible's module resolution logic is hardcoded to check them.

### What happens if you rename the directory

If `solution/09/files/` were renamed to `solution/09/artifacts/` without changing the task, the `unarchive` task would fail — the implicit search never looks inside a directory called `artifacts/`, no matter how logically named it is:

```
Could not find or access 'playbook_stacker.zip'
Searched in:
    <playbook_dir>/files/playbook_stacker.zip
    <playbook_dir>/playbook_stacker.zip
    ...
```

There are exactly two ways to recover from this:

1. **Rename the directory back to `files/`** — the implicit lookup starts working again with no playbook changes at all.
2. **Keep the new name, but make the path explicit in the task**: `src: artifacts/playbook_stacker.zip`. This works for a different reason — a relative path *with* a directory component is resolved directly relative to the playbook's directory, bypassing the "guess the conventional subdirectory" step entirely. You're no longer relying on the magic name; you're just giving a normal relative filesystem path.

The same two options apply identically to `template:` and the `templates/` directory.

### Why `unarchive` needs this even though it also runs on the remote host

It's worth noting that `unarchive` is a little more subtle than `copy`, because unarchiving is inherently a two-machine operation: the archive usually starts on the control node and needs to end up extracted on the *target*. By default, `unarchive` assumes `remote_src: false` — meaning `src` is resolved on the **control node** using exactly the `files/` search path described above, and Ansible transparently copies the archive to the target host before extracting it there in the same task. If the archive were already present on the managed host instead (e.g., downloaded there by a previous task), you'd set `remote_src: true` and give a path that's valid *on the target*, not on the control node — at which point the `files/` search path is irrelevant, since there's no local file to look up at all.

### Quick-Reference: Implicit Search Paths by Module

| Module | Conventional directory searched | Governed by `ansible.cfg`? | Bypass by... |
|---|---|---|---|
| `copy` | `files/` | No — hardcoded in Ansible core | Giving `src:` a directory component, e.g. `src: subdir/file.txt` |
| `unarchive` (`remote_src: false`, the default) | `files/` | No — hardcoded in Ansible core | Same as above, or set `remote_src: true` with a target-side path |
| `script` | `files/` | No — hardcoded in Ansible core | Same as above |
| `template` | `templates/` | No — hardcoded in Ansible core | Same as above |

**Rule of thumb when restructuring any playbook, in this course or your own:** if a task's `src:` is a bare filename with no path, go looking for a `files/` or `templates/` directory sitting next to the playbook before assuming the reference is broken — that's almost always where the file actually lives.

---

## Suggested Next Steps

Once you're comfortable with everything above, a few good ways to reinforce this section before moving on in the course:

1. Build the playbook yourself from `template/nginx_playbook.yaml`, one step at a time, checking your work against `solution/01` through `solution/09` as you go, rather than reading the solutions first.
2. Deliberately remove the `when:` guard from `01`'s EPEL task and run it against an Ubuntu host, to see what failure looks like when a distribution-specific task isn't actually guarded.
3. Add a third distribution's logo (or a placeholder `else` branch) to `08`/`09`'s Jinja2 `if`/`elif`, to practice extending a per-OS template rather than just reading one.
4. Remove the `notify:` from `05` and intentionally break Nginx (e.g. stop the service manually first) to see how the play behaves without the `Check HTTP Service` handler catching the failure.
