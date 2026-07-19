# 02 — Modules

This README is a learning companion for this section of the *Dive Into Ansible* course. Unlike the other sections in this repository, this folder doesn't contain a series of numbered mini-labs — it holds a single, shared `ansible.cfg` and `hosts` inventory, because this part of the course is about running Ansible **modules ad hoc**, directly from the command line via the `ansible` CLI, rather than writing them into a saved `.yaml` playbook.

The goal of this README is to document the inventory this section runs against and give you a runnable reference for the kinds of ad-hoc commands this lesson covers — so you have something to `cd` into and try commands against immediately, even though there's no `test.yaml` or `playbook.yaml` to open first.

---

## The Inventory

`hosts` (INI format):

```ini
[control]
ubuntu-c ansible_connection=local

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

[linux:children]
centos
ubuntu
```

This is the same structure built up across `01-inventories/11`–`13`: a `control` host running locally, `centos`/`ubuntu` groups each with their own connection vars, and a `linux` parent group combining both. If any of `ansible_connection`, `[group:vars]`, or `[group:children]` looks unfamiliar, that's covered step by step in [`../01-inventories/README.md`](../01-inventories/README.md) — this section assumes that inventory knowledge and builds on top of it.

`ansible.cfg`:

```ini
[defaults]
inventory = hosts
host_key_checking = False
```

Nothing new here either — same `host_key_checking = False` convention used throughout `01-inventories` from example `02` onward.

---

## Ad-Hoc Modules vs. Playbooks

Everywhere else in this course, a module (`copy`, `debug`, `file`, `template`, …) is invoked from inside a `tasks:` list in a saved playbook, run with `ansible-playbook`. **Ad hoc** means invoking a single module directly against your inventory, with `ansible` instead of `ansible-playbook`, and no playbook file at all:

```bash
ansible <pattern> -m <module> -a "<module arguments>"
```

- `<pattern>` — which hosts/groups to target (a group name, `all`, a host, or a pattern like `centos*`)
- `-m` — the module to run (defaults to the `command` module if omitted)
- `-a` — the arguments to pass to that module, as a single quoted string

Ad-hoc commands are one-off and don't get saved anywhere by default — they're the tool for "I need to do this one thing on these hosts right now," not for repeatable, version-controlled automation (that's what a playbook is for). They're also how most people first explore a new module's behavior before committing to using it inside a real playbook.

---

## Common Ad-Hoc Commands to Try Against This Inventory

Run these from inside this folder, so `ansible.cfg` picks up the `hosts` file automatically:

```bash
cd 01-ansible-architecture-and-design/02-modules
```

### `ping` — connectivity check

```bash
ansible all -m ping
```

The standard first command against any new inventory — confirms Ansible can reach and authenticate to every host, independent of what any later task might do. Despite the name, it doesn't use ICMP; it verifies Python is available on the remote host and that Ansible can execute a module there at all.

### `command` — run a simple remote command

```bash
ansible linux -m command -a "uptime"
```

Runs `uptime` on every host in the `linux` group (`centos` + `ubuntu`, per this folder's inventory). The `command` module is the default if `-m` is omitted, but does **not** go through a shell — no pipes, redirects, or environment variable expansion. For that, see `shell` below.

### `shell` — run a command through the remote shell

```bash
ansible centos -m shell -a "cat /etc/os-release | grep VERSION"
```

Same idea as `command`, but executed through `/bin/sh` on the remote host, so shell features like pipes (`|`) work. Prefer `command` when you don't need shell features — it avoids a class of quoting/injection issues that `shell` is more exposed to.

### `copy` — push a file or inline content

```bash
ansible ubuntu -m copy -a "content='hello from ansible\n' dest=/tmp/hello.txt"
```

The ad-hoc equivalent of the `copy` module tasks seen throughout `02-ansible-playbooks-introduction/02-ansible-playbooks-breakdown-of-sections` — same module, same `content`/`dest` arguments, just invoked directly instead of from inside a playbook task.

### `file` — manage file/directory state

```bash
ansible linux -m file -a "path=/tmp/example_dir state=directory mode=0755"
```

Creates `/tmp/example_dir` on every `linux` host with the given permissions. `state=absent` instead would remove it — a quick way to clean up anything created during ad-hoc experimentation.

### `setup` — dump all gathered facts

```bash
ansible centos1 -m setup
```

Runs the same fact-gathering module that fires implicitly at the start of every play (see `02-ansible-playbooks-introduction/04-ansible-playbooks-facts`), but ad hoc, printing the full `ansible_*` fact dictionary for `centos1` to your terminal. Useful for looking up the exact name/shape of a fact before referencing it in a real playbook — pipe through `grep` to find what you need, e.g. `ansible centos1 -m setup | grep -i ipv4`.

### `yum` / `apt` — package management

```bash
ansible centos -m yum -a "name=httpd state=present"
ansible ubuntu -m apt -a "name=apache2 state=present update_cache=yes"
```

OS-specific package modules — `yum` for the `centos` group, `apt` for `ubuntu`. This split is exactly why grouping hosts by OS (as `01-inventories/03` onward does) matters in practice: a single ad-hoc command can only sensibly target hosts that share a package manager.

### `service` — manage a running service

```bash
ansible linux -m service -a "name=sshd state=restarted"
```

Restarts a service by name. Combine with `-b` (become/sudo) if the target hosts require privilege escalation for service management and it isn't already set via `ansible_become` in the inventory (as `ubuntu` is here).

---

## Useful Flags

| Flag | Purpose |
|---|---|
| `-m <module>` | Which module to run (default: `command`) |
| `-a "<args>"` | Arguments passed to the module |
| `-b` | Become (sudo) — usually unnecessary here since `ubuntu:vars` already sets `ansible_become` |
| `-i <inventory>` | Override the inventory file (not needed here — `ansible.cfg` already points at `hosts`) |
| `-o` | One-line-per-host output — easier to scan when targeting many hosts at once |
| `-v` / `-vvv` | Increase verbosity — `-vvv` is the first thing to reach for when an ad-hoc command fails unexpectedly |

---

## Suggested Next Steps

1. Run `ansible all -m ping` first, against the inventory above, before trying anything else — confirm connectivity before debugging any module-specific issue.
2. Run `ansible centos1 -m setup` and find `ansible_distribution` and `ansible_default_ipv4.address` in the output — the same facts referenced by name throughout `02-ansible-playbooks-introduction/04-ansible-playbooks-facts`.
3. Compare `command` vs. `shell` directly: run `ansible linux -m command -a "echo hi | wc -l"` (fails/behaves oddly — no shell) against `ansible linux -m shell -a "echo hi | wc -l"` (works as expected).
4. Once comfortable with a module ad hoc, try writing the equivalent task into a playbook (see `02-ansible-playbooks-introduction/02-ansible-playbooks-breakdown-of-sections`) — this is the natural next step once a one-off command turns into something you want to repeat.
