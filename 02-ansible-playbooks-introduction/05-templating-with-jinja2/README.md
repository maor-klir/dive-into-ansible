# 05 — Templating with Jinja2

This README is a learning companion for this section of the *Dive Into Ansible* course. It walks through each numbered example (`01`–`11`) in this folder, in the order you're meant to learn them, building up from a single `if` statement to a full external template file.

The goal isn't just to show *what* each example does, but *why* it's built that way — including a couple of details that trip up almost everyone new to Jinja2 in Ansible: whitespace control (the trailing `-`) and the `jinja2_extensions` config setting needed for `break`/`continue`.

---

## How to use this folder while learning

Each subfolder `01`–`11` is a **self-contained mini-lab**: its own `ansible.cfg`, `hosts` inventory, and a single `jinja2_playbook.yaml`. That means you can `cd` into any one of them and run it in isolation:

```bash
cd 02-ansible-playbooks-introduction/05-templating-with-jinja2/03
ansible-playbook jinja2_playbook.yaml
```

This is a great pattern to copy for your own experiments later — one concept, one folder, one playbook, run independently of everything else.

---

## Quick Reference: The Full Combined Template

Folder `11` ties everything together into one Jinja2 template file, rendered with the `template` module instead of `debug`. It's reproduced in full here so you have a single place to see every concept side-by-side once you've worked through the individual examples below.

```jinja2
--== Ansible Jinja2 if statement ==--

{# If the hostname is ubuntu-c, include a message -#}
{% if ansible_hostname == "ubuntu-c" -%}
      This is ubuntu-c
{% endif %}

--== Ansible Jinja2 if elif statement ==--

{% if ansible_hostname == "ubuntu-c" -%}
   This is ubuntu-c
{% elif ansible_hostname == "centos1" -%}
   This is centos1 with it's modified SSH Port
{% endif %}

--== Ansible Jinja2 if elif else statement ==--

{% if ansible_hostname == "ubuntu-c" -%}
   This is ubuntu-c
{% elif ansible_hostname == "centos1" -%}
   This is centos1 with it's modified SSH Port
{% else -%}
   This is good old {{ ansible_hostname }}
{% endif %}

--== Ansible Jinja2 if variable is defined ( where variable is not defined ) ==--

{% if example_variable is defined -%}
   example_variable is defined
{% else -%}
   example_variable is not defined
{% endif %}

--== Ansible Jinja2 if varible is defined ( where variable is defined ) ==--

{% set example_variable = 'defined' -%}
{% if example_variable is defined -%}
   example_variable is defined
{% else -%}
   example_variable is not defined
{% endif %}

--== Ansible Jinja2 for statement ==--

{% for entry in ansible_all_ipv4_addresses -%}
   IP Address entry {{ loop.index }} = {{ entry }}
{% endfor %}

--== Ansible Jinja2 for range

{% for entry in range(1, 11) -%}
   {{ entry }}
{% endfor %}

--== Ansible Jinja2 for range, reversed (simulate while greater 5) ==--

{% for entry in range(10, 0, -1) -%}
   {% if entry == 5 -%}
      {% break %}
   {% endif -%}
   {{ entry }}
{% endfor %}

--== Ansible Jinja2 for range, reversed (continue if odd) ==--

{% for entry in range(10, 0, -1) -%}
   {% if entry is odd -%}
      {% continue %}
   {% endif -%}
   {{ entry }}
{% endfor %}

---=== Ansible Jinja2 filters ===---

--== min [1, 2, 3, 4, 5] ==--

{{ [1, 2, 3, 4, 5] | min }}

--== max [1, 2, 3, 4, 5] ==--

{{ [1, 2, 3, 4, 5] | max }}

--== unique [1, 1, 2, 2, 3, 3, 4, 4, 5, 5] ==--

{{ [1, 1, 2, 2, 3, 3, 4, 4, 5, 5] | unique }}

--== difference [1, 2, 3, 4, 5] vs [2, 3, 4] ==--

{{ [1, 2, 3, 4, 5] | difference([2, 3, 4]) }}

--== random ['rod', 'jane', 'freddy'] ==--

{{ ['rod', 'jane', 'freddy'] | random }}

--== urlsplit hostname ==--

{{ "http://docs.ansible.com/ansible/latest/playbooks_filters.html" | urlsplit('hostname') }}
```
[Template source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/05-templating-with-jinja2/11/template.j2) ·
[Playbook that renders it](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/05-templating-with-jinja2/11/jinja2_playbook.yaml)

Rendered via:

```yaml
- name: Jinja2 template
  template:
    src: template.j2
    dest: "/tmp/{{ ansible_hostname }}_template.out"
    trim_blocks: true
    mode: 0644
```

Notice `trim_blocks: true` — that's directly tied to the whitespace-control lesson below, and is what keeps the rendered output free of stray blank lines after every `{% ... %}` tag.

---

## Lesson: Understanding the Trailing `-` (Whitespace Control)

Before going through the numbered examples, it's worth pausing on something you'll notice immediately in almost every file in this folder: **why do so many tags end with a dash**, like this?

```jinja2
{% if ansible_hostname == "ubuntu-c" -%}
```

instead of the "plain" form:

```jinja2
{% if ansible_hostname == "ubuntu-c" %}
```

### The problem it solves

Jinja2 treats a template as plain text with embedded tags. Every `{% ... %}`, `{{ ... }}`, or `{# ... #}` block is cut out of the output, but the **newline character that follows the tag in your source file is not removed automatically**. So a template written the natural, readable way — with tags on their own lines:

```jinja2
{% if ansible_hostname == "ubuntu-c" %}
This is ubuntu-c
{% endif %}
```

...renders with extra blank lines, because the newline after `%}` on the `if` line, and the newline after `%}` on the `endif` line, both get preserved in the output. For one `if` block this is barely noticeable, but as you start nesting `for` loops and `if` conditions (which you'll do a lot as you progress through this folder), unwanted blank lines multiply fast and produce messy output.

### The fix: whitespace control modifiers

Add a `-` immediately inside a tag delimiter to strip adjacent whitespace (including the newline):

- `{%- ... %}` strips whitespace **before** the tag.
- `{% ... -%}` strips whitespace **after** the tag.
- Both together: `{%- ... -%}`.

Throughout this folder you'll see the trailing form, `-%}`, used on `if`, `elif`, `for`, and even comment tags:

```jinja2
{# If the hostname is ubuntu-c, include a message -#}
{% if ansible_hostname == "ubuntu-c" -%}
      This is ubuntu-c
{% endif %}
```

The `-%}` after the `if` condition strips the line break that would otherwise appear right after the tag, so the rendered text flows directly into `This is ubuntu-c` without an extra blank line in front of it. Notice the `endif` tag is usually left **without** a trailing dash in these examples — a single trailing blank line at the very end of a block is often left in place on purpose, since it doesn't matter once it's the last thing printed for that task.

### How this relates to `trim_blocks` and `lstrip_blocks`

Manually adding `-%}` everywhere works, but Ansible (via Jinja2) also exposes two options that solve the same problem globally, without sprinkling dashes throughout a template:

- **`trim_blocks: true`** — automatically removes the first newline after a block tag (equivalent to always doing `-%}`). Used in `11`'s playbook for the `template` module.
- **`lstrip_blocks: true`** — strips leading whitespace/tabs before a block tag on its own line (equivalent to always doing `{%- `).

**Key thing to remember as you work through this folder:** for `debug`-based inline messages (examples `01`–`10`), you have to add `-%}` (or `-#}` for comments) manually, because `debug` messages don't go through the `trim_blocks`/`lstrip_blocks` engine options — those only apply to files rendered through the `template` module. That's exactly why every inline `debug` example here ends its opening tags with `-%}`, while `template.j2` (example `11`) can additionally lean on `trim_blocks: true` at the playbook level.

---

## Walking Through the Examples

### 01 — `if` statement

```jinja2
{# If the hostname is ubuntu-c, include a message -#}
{% if ansible_hostname == "ubuntu-c" -%}
      This is ubuntu-c
{% endif %}
```
[Source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/05-templating-with-jinja2/01/jinja2_playbook.yaml)

The most basic control structure: a single condition guarding a block of text. Also your first look at a Jinja2 **comment**, `{# ... #}` — comments are stripped entirely from output, useful for documenting *why* a template does something without that explanation leaking into the rendered result. Note the comment itself uses `-#}` to strip the newline that follows it.

---

### 02 — `if` / `elif` statement

```jinja2
{% if ansible_hostname == "ubuntu-c" -%}
   This is ubuntu-c
{% elif ansible_hostname == "centos1" -%}
   This is centos1 with it's modified SSH Port
{% endif %}
```
[Source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/05-templating-with-jinja2/02/jinja2_playbook.yaml)

Adds a second condition with `elif` — just like Python's `elif` — evaluated only if the preceding `if` was false. Handy for host-specific messaging without needing separate `when:` clauses in the playbook itself.

---

### 03 — `if` / `elif` / `else` statement

```jinja2
{% if ansible_hostname == "ubuntu-c" -%}
   This is ubuntu-c
{% elif ansible_hostname == "centos1" -%}
   This is centos1 with it's modified SSH Port
{% else -%}
   This is good old {{ ansible_hostname }}
{% endif %}
```
[Source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/05-templating-with-jinja2/03/jinja2_playbook.yaml)

Adds a fallback `else` branch, and your first **variable substitution**, `{{ ansible_hostname }}` — the double-curly-brace syntax used to print the value of an expression directly into the output. Contrast this with `{% %}`, which is for control structures/statements that don't themselves produce output.

---

### 04 — `is defined` test (undefined case)

```jinja2
{% if example_variable is defined -%}
   example_variable is defined
{% else -%}
   example_variable is not defined
{% endif %}
```
[Source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/05-templating-with-jinja2/04/jinja2_playbook.yaml)

Introduces Jinja2 **tests** (used with `is`), as opposed to filters (used with `|`). `is defined` is one of the most useful tests you'll use in real playbooks, since it lets a template guard against variables that may not be set for every host/group — avoiding `'example_variable' is undefined` errors. Here, `example_variable` was never set, so the `else` branch fires.

---

### 05 — `is defined` test (defined case) + `set`

```jinja2
{% set example_variable = 'defined' -%}
{% if example_variable is defined -%}
   example_variable is defined
{% else -%}
   example_variable is not defined
{% endif %}
```
[Source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/05-templating-with-jinja2/05/jinja2_playbook.yaml)

The direct counterpart to `04` — run these two back to back to really see the contrast. Introduces `{% set %}`, which declares a template-local variable inline (scoped to the template render, not persisted back to Ansible facts/vars). With `example_variable` now set to `'defined'`, the `if` branch is taken this time. Remember: `{% set %}` variables only exist within the Jinja2 rendering context — they won't show up as Ansible facts or be usable outside the template/expression they're defined in.

---

### 06 — `for` loop over a list, with `loop.index`

```jinja2
{% for entry in ansible_interfaces -%}
   Interface entry {{ loop.index }} = {{ entry }}
{% endfor %}
```
[Source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/05-templating-with-jinja2/06/jinja2_playbook.yaml)

Your first `for` loop. Iterates over `ansible_interfaces`, a built-in Ansible fact listing network interface names on the target host. `loop.index` is a Jinja2 loop variable (1-indexed) automatically available inside any `for` block — useful for numbering output without maintaining a manual counter. Other loop variables worth knowing: `loop.first`, `loop.last`, and `loop.index0` (0-indexed).

---

### 07 — `for` loop with `range()`

```jinja2
{% for entry in range(1, 11) -%}
   {{ entry }}
{% endfor %}
```
[Source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/05-templating-with-jinja2/07/jinja2_playbook.yaml)

`range()` is a Python-style built-in available in Jinja2, generating a sequence of integers. `range(1, 11)` produces `1` through `10` (the stop value is exclusive) — handy for generating numbered lists or counters without hardcoding a list.

This directory's `ansible.cfg` doesn't declare any Jinja2 extensions — it doesn't need to, since `range()` is a core Jinja2 built-in:

```ini
[defaults]
inventory = hosts
host_key_checking = False
```

---

### 08 — `for` + `range()` reversed, with `break`

```jinja2
{% for entry in range(10, 0, -1) -%}
   {% if entry == 5 -%}
      {% break %}
   {% endif -%}
   {{ entry }}
{% endfor %}
```
[Source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/05-templating-with-jinja2/08/jinja2_playbook.yaml)

`range(10, 0, -1)` counts down from 10 to 1 (the third argument is the step). Combined with `{% break %}` inside a nested `if`, this simulates a `while` loop that runs "as long as `entry` is greater than 5".

**This is the first example that needs an extra config change — don't skip this if you're following along.** `break`/`continue` are **not** native Jinja2 syntax. Per Jinja's own documentation, they're provided by the `jinja2.ext.loopcontrols` extension, which must be loaded into the environment before those keywords are recognized. Ansible does **not** load any Jinja2 extensions by default — so the extension has to be declared explicitly. This is exactly what changes starting in this directory; compare the two config files:

`07/ansible.cfg` (no loop controls used, no extension declared):
```ini
[defaults]
inventory = hosts
host_key_checking = False
```

`08/ansible.cfg` (introduces `{% break %}`, so the extension is added):
```ini
[defaults]
inventory = hosts
host_key_checking = False
jinja2_extensions = jinja2.ext.loopcontrols
```

Multiple extensions can be declared comma-separated, e.g. `jinja2_extensions = jinja2.ext.loopcontrols,jinja2.ext.do`. The same setting can also be provided via the `ANSIBLE_JINJA2_EXTENSIONS` environment variable.

**If you forget this config and try `{% break %}` anyway**, you'll get a Jinja parser error surfaced directly by Ansible, something like:

```
Encountered unknown tag 'break'. Jinja was looking for the following tags:
'elif' or 'else' or 'endif'. The innermost block that needs to be closed is 'if'.
```

That error is a great one to intentionally trigger once, just so you recognize it later.

**⚠️ A note for your future self as you keep learning Ansible:** `jinja2_extensions` (Ansible-core setting name `DEFAULT_JINJA2_EXTENSIONS`) is **deprecated in current ansible-core**, with removal scheduled for **ansible-core 2.23**. There isn't yet a clean, official one-to-one replacement for `break` specifically. If you keep using this pattern in your own playbooks later, it's worth checking `ansible-config dump` on whatever version you're running to see the current deprecation status, and keeping an eye on whether an alternative has landed by the time you're building anything long-lived. For the `continue` case (example `09`), there already is a better, non-deprecated way — see below.

---

### 09 — `for` loop with `continue` and the `odd` test

```jinja2
{% for entry in range(10, 0, -1) -%}
   {% if entry is odd -%}
      {% continue %}
   {% endif -%}
   {{ entry }}
{% endfor %}
```
[Source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/05-templating-with-jinja2/09/jinja2_playbook.yaml)

Pairs `{% continue %}` with the built-in `odd` test, to print only the even numbers from 10 down to 1. Like `break` above, this depends on `jinja2_extensions = jinja2.ext.loopcontrols` being set (present in `09/ansible.cfg`, carried forward from `08`) — and carries the same deprecation caveat.

The good news: this specific pattern (skip-if-condition) has a clean, currently-preferred alternative that avoids the deprecated extension entirely — Jinja2's **loop filtering** syntax:

```jinja2
{% for entry in range(10, 0, -1) if entry is not odd -%}
   {{ entry }}
{% endfor %}
```

This is functionally equivalent to the `continue`-based version above, without requiring `jinja2_extensions` at all. Once you've understood *why* `continue` works here, try rewriting this example yourself using the filtered `for` syntax as practice — it's a good exercise for internalizing the difference between "skip inside the loop body" and "don't iterate over it in the first place."

---

### 10 — Jinja2 filters

```jinja2
{{ [1, 2, 3, 4, 5] | min }}
{{ [1, 2, 3, 4, 5] | max }}
{{ [1, 1, 2, 2, 3, 3, 4, 4, 5, 5] | unique }}
{{ [1, 2, 3, 4, 5] | difference([2, 3, 4]) }}
{{ ['rod', 'jane', 'freddy'] | random }}
{{ "http://docs.ansible.com/ansible/latest/playbooks_filters.html" | urlsplit('hostname') }}
```
[Source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/05-templating-with-jinja2/10/jinja2_playbook.yaml)

**Filters** (the pipe `|` syntax) transform a value without needing a full statement block — the Jinja2 equivalent of a Unix pipeline. This example is a nice "grab bag" of filters that come up a lot once you start writing real automation:

| Filter | What it's for |
|---|---|
| `min` / `max` | Find the smallest/largest item in a list — e.g. picking the lowest available port from a range. |
| `unique` | De-duplicate a list — e.g. collapsing hostnames/IPs gathered from multiple groups. |
| `difference` | Set difference between two lists — e.g. "which hosts are in group A but not group B". |
| `random` | Pick a random element — e.g. randomly selecting a canary host. |
| `urlsplit` | Pull out a specific component (`hostname`, `scheme`, `path`, `port`, etc.) from a URL string. |

Filters can also be **chained**, e.g. `{{ mylist | unique | sort }}` — try experimenting with chaining a couple of these together once you're comfortable with each one individually. None of the filters here require any Jinja2 extension; filters are core Jinja2 functionality, unlike `break`/`continue`.

---

### 11 — Full `template.j2` file (using the `template` module)

Unlike `01`–`10` (which render messages inline via the `debug` module), this example renders an **external Jinja2 template file** to disk — this is the more "real world" way you'll actually use Jinja2 in Ansible once you move past `debug`-based experiments:

```yaml
- name: Jinja2 template
  template:
    src: template.j2
    dest: "/tmp/{{ ansible_hostname }}_template.out"
    trim_blocks: true
    mode: 0644
```
[Playbook source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/05-templating-with-jinja2/11/jinja2_playbook.yaml) ·
[Template source](https://github.com/maor-klir/dive-into-ansible/blob/master/02-ansible-playbooks-introduction/05-templating-with-jinja2/11/template.j2)

This is the file reproduced in full near the top of this README. It rolls up every concept from `01`–`10` into one file — a good one to keep bookmarked as a reference once you start writing your own templates. Because it still uses `{% break %}` and `{% continue %}` (carried over from `08`/`09`), its `ansible.cfg` also declares `jinja2_extensions = jinja2.ext.loopcontrols`, so the same deprecation note applies here too.

Two differences from the inline examples worth noticing as you compare this to the earlier ones:

- It iterates over `ansible_all_ipv4_addresses` in its `for` loop (all IPv4 addresses on the host), rather than `ansible_interfaces` (interface names) used back in example `06` — a good reminder that Ansible exposes many related-but-different facts, and picking the right one matters for the output you actually want.
- It relies on `trim_blocks: true` at the playbook level (see the whitespace-control lesson above) rather than needing every single tag in the file to end in `-%}`, since this is a real file being rendered through the `template` module, not a `debug` message.

**Try this once you've run it:** run the playbook, then `cat` the generated file at `/tmp/<your_hostname>_template.out` and compare it line-by-line against `template.j2`. Seeing exactly which blank lines got trimmed (and why) is the best way to cement the whitespace-control lesson from earlier.

---

## Summary Table

| # | Concept | Key Jinja2 Feature(s) | File(s) | Notes |
|---|---------|------------------------|---------|-------|
| 01 | `if` statement + comment | `{% if %}`, `{# #}`, trailing `-%}` | `01/jinja2_playbook.yaml` | |
| 02 | `if` / `elif` | `{% elif %}` | `02/jinja2_playbook.yaml` | |
| 03 | `if` / `elif` / `else` + variable substitution | `{% else %}`, `{{ var }}` | `03/jinja2_playbook.yaml` | |
| 04 | `is defined` (undefined case) | `is defined` test | `04/jinja2_playbook.yaml` | |
| 05 | `set` + `is defined` (defined case) | `{% set %}` | `05/jinja2_playbook.yaml` | |
| 06 | `for` loop + `loop.index` | `{% for %}`, `loop.index` | `06/jinja2_playbook.yaml` | |
| 07 | `for` + `range()` | `range()` | `07/jinja2_playbook.yaml` | No extension needed |
| 08 | `for` + `range()` reversed + `break` | `{% break %}` | `08/jinja2_playbook.yaml` | Requires `jinja2_extensions = jinja2.ext.loopcontrols`; ⚠️ deprecated, removal in ansible-core 2.23, no direct replacement yet |
| 09 | `for` + `range()` reversed + `continue` + `odd` test | `{% continue %}`, `is odd` | `09/jinja2_playbook.yaml` | Same requirement/deprecation as `08`; prefer `{% for x in y if cond %}` instead |
| 10 | Filters | `min`, `max`, `unique`, `difference`, `random`, `urlsplit` | `10/jinja2_playbook.yaml` | No extension needed |
| 11 | External template file combining all concepts | `template` module, `trim_blocks` | `11/jinja2_playbook.yaml`, `11/template.j2` | Also requires `loopcontrols` due to inherited `break`/`continue` usage |

---

## Quick-Reference: Whitespace Control

| Syntax | Effect |
|---|---|
| `{% tag %}` | No whitespace stripped — newline after the tag is preserved in output. |
| `{% tag -%}` | Strips whitespace/newline **immediately after** the tag. |
| `{%- tag %}` | Strips whitespace **immediately before** the tag. |
| `{%- tag -%}` | Strips whitespace on **both** sides of the tag. |
| `{# comment -#}` | Same rules apply to comment tags. |
| `trim_blocks: true` (template module option) | Globally applies the effect of `-%}` to every block tag in the file. |
| `lstrip_blocks: true` (template module option) | Globally strips leading whitespace before a block tag on its own line. |

**Rule of thumb for when you're debugging your own templates:** if your rendered `debug` output or generated file has more blank lines than expected, check for a missing trailing `-` on your `{% %}` tags first — it's the most common source of unwanted whitespace when learning Jinja2 in Ansible.

---

## Quick-Reference: Jinja2 Extensions in Ansible

| Item | Detail |
|---|---|
| Config key | `jinja2_extensions` (ini, under `[defaults]`) / `ANSIBLE_JINJA2_EXTENSIONS` (env var) |
| Ansible-core setting name | `DEFAULT_JINJA2_EXTENSIONS` |
| Default value | Empty list — **no extensions loaded by default** |
| Example used in this folder | `jinja2_extensions = jinja2.ext.loopcontrols` (enables `{% break %}` / `{% continue %}`) |
| Multiple extensions | Comma-separated, e.g. `jinja2.ext.loopcontrols,jinja2.ext.do` |
| Error if extension missing | Jinja `TemplateSyntaxError`, surfaced by Ansible as-is, e.g. `Encountered unknown tag 'break'...` |
| Status | ⚠️ Deprecated, scheduled for removal in ansible-core 2.23 |
| Recommended direction | Move toward Ansible-supported Jinja plugins (tests/filters/lookups); for loop filtering use `{% for x in items if cond %}` instead of `continue`; no direct replacement yet for `break` |
| Good habit while learning | Note down which of your own playbooks use this setting, so you remember to revisit them if/when you upgrade ansible-core down the line |

---

## `ansible.cfg` Changes Across This Folder

A quick map of what changes between the subfolders, useful when you're jumping between examples:

| Directory | `ansible.cfg` contents |
|---|---|
| `01`–`07` | `inventory = hosts` / `host_key_checking = False` (no Jinja2 extensions) |
| `08`–`11` | Same as above, **plus** `jinja2_extensions = jinja2.ext.loopcontrols` (required for `break`/`continue`) |

This confirms directly from the repository that the extension is introduced exactly at the point `{% break %}` first appears (`08`), and stays in place through `09`–`11` since those continue to use `break`/`continue`.

---

## Suggested Next Steps

Once you're comfortable with everything above, a few good ways to reinforce this section before moving on in the course:

1. Rewrite example `09` using loop filtering (`{% for x in y if cond %}`) instead of `continue`, and confirm you get identical output.
2. Deliberately remove `jinja2_extensions` from `08/ansible.cfg` and re-run the playbook, just to see the `TemplateSyntaxError` for yourself.
3. Take the combined `template.j2` from example `11` and add one more section of your own — e.g., a filter you haven't tried yet — to practice writing Jinja2 from scratch rather than just reading it.
4. Compare the rendered `/tmp/<hostname>_template.out` file against `template.j2` line-by-line to fully internalize whitespace control.
