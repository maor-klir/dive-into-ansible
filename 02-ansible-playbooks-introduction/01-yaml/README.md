# 01 — YAML Basics

This README is a learning companion for this section of the *Dive Into Ansible* course. It walks through each numbered example (`01`–`19` plus `challenge`) in this folder, in the order you're meant to learn them, explaining what each one demonstrates about YAML syntax and how Ansible (and Python's `yaml` module) parses it.

The goal isn't just to show *what* each example does, but *why* YAML is structured this way — since almost every Ansible playbook, inventory file, and variable file you'll ever write depends on getting these fundamentals right.

---

## How to use this folder while learning

Each subfolder `01`–`19` (plus `challenge`) is a **self-contained mini-lab** containing a single `test.yaml` file and a `show_yaml_python.sh` helper script. The helper script parses `test.yaml` with Python's `yaml` module and pretty-prints the result, so you can see exactly how the YAML gets interpreted:

```bash
cd 02-ansible-playbooks-introduction/01-yaml/01
./show_yaml_python.sh
```

The script itself is a one-liner:

```bash
python3 -c 'import yaml,pprint;pprint.pprint(yaml.load(open("test.yaml").read(), Loader=yaml.FullLoader))'
```

Run this after reading (not before!) each `test.yaml`, so you can predict the parsed structure before confirming it.

---

## Walking Through the Examples

### 01 — Document markers (`---` and `...`)

```yaml
# Every YAML file should start with three dashes
---


# Every YAML file should end with three dots
...
```

The most minimal valid YAML document. `---` marks the start of a document, `...` marks the end. Both are optional in many parsers but are considered good practice — Ansible playbooks conventionally start with `---`.

---

### 02 — Simple key/value pairs

```yaml
---

example_key_1: this is a string
example_key_2: this is another string
 
...
```

Introduces the most basic YAML structure: a mapping (dictionary) of `key: value` pairs, unquoted.

---

### 03 — Quoting strings

```yaml
---

no_quotes: this is a string example
double_quotes: "this is a string example"
single_quotes: 'this is a string example'
 
...
```

Shows that unquoted, single-quoted, and double-quoted strings all parse to the same plain string value here — quoting mostly matters when you need to *escape* characters (see `04`) or avoid ambiguity with YAML's special characters.

---

### 04 — Quoting and escape sequences

```yaml
---

no_quotes: this is a string example\n
double_quotes: "this is a string example\n"
single_quotes: 'this is a string example\n'
 
...
```

The key lesson: `\n` is only interpreted as a newline **inside double quotes**. In the unquoted and single-quoted forms, `\n` is treated as two literal characters (backslash + `n`). Run this one through `show_yaml_python.sh` and compare the three outputs closely.

---

### 05 — Literal block scalar (`|`)

```yaml
---

example_key_1: |
  this is a string
  that goes over
  multiple lines
 
...
```

The pipe `|` preserves line breaks exactly as written — each line in the block becomes its own line in the resulting string, with a single trailing newline. Useful for multi-line config file content, MOTD messages, etc.

---

### 06 — Folded block scalar (`>`)

```yaml
---

example_key_1: >
  this is a string
  that goes over
  multiple lines
 
...
```

The `>` "folds" line breaks into spaces, turning multiple lines into a single line — except for blank lines, which become actual newlines. Contrast this directly with `05` by running both through the helper script.

---

### 07 — Folded block scalar, strip chomping (`>-`)

```yaml
---

example_key_1: >-
  this is a string
  that goes over
  multiple lines
 
...
```

Same folding behavior as `06`, but the `-` "strip" chomping indicator removes the trailing newline entirely. Compare the parsed output against `06` to see the difference.

---

### 08 — Integers

```yaml
---

example_integer: 1
 
...
```

A bare, unquoted number is parsed as an actual integer type, not a string — important when a task expects a specific type.

---

### 09 — Quoted "integers" are strings

```yaml
---

example_integer: "1"
 
...
```

The direct counterpart to `08`: wrapping the same value in quotes forces it to be parsed as a string. Run both through `show_yaml_python.sh` and note the different Python types (`int` vs `str`) in the output.

---

### 10 — Booleans (all the ways YAML spells `true`/`false`)

```yaml
---

is_false_01: false
is_false_02: False
is_false_03: FALSE
is_false_04: no
is_false_05: No
is_false_06: NO
is_false_07: off
is_false_08: Off
is_false_09: OFF
is_false_10: n
is_true_01: true
is_true_02: True
is_true_03: TRUE
is_true_04: yes
is_true_05: Yes
is_true_06: YES
is_true_07: on
is_true_08: On
is_true_09: ON
is_true_10: y

...
```

YAML 1.1 (used by PyYAML/Ansible) recognizes a surprisingly large set of boolean spellings. **Important gotcha called out directly in the file:** `n` and `y` *do* count as booleans here, but this is a common source of bugs — a country code of `"NO"` (Norway) or `"no"` as a literal string can accidentally be parsed as `false` if left unquoted. Always quote such values explicitly when you mean a string.

---

### 11 — A simple list (block style)

```yaml
---

- item 1
- item 2
- item 3
- item 4
- item 5

...
```

A YAML **sequence** (list), block style — each `-` starts a new list item. This is the format Ansible playbooks themselves use at the top level, since a playbook is a list of plays.

---

### 12 — A simple dictionary (block style)

```yaml
---

example_key_1: example_value_1
example_key_2: example_value_2

...
```

A plain block-style mapping with two keys — the direct dictionary counterpart to the list in `11`.

---

### 13 — Inline (flow-style) dictionary

```yaml
---

{example_key_1: example_value_1, example_key_2: example_value_2}

...
```

The same dictionary as `12`, but written in **flow style** using `{ }` — JSON-like inline syntax. YAML is a superset of JSON, so this is valid YAML too.

---

### 14 — Inline (flow-style) list

```yaml
---

[example_list_entry_1, example_list_entry_2]

...
```

The flow-style counterpart to `11`'s block-style list, using `[ ]`.

---

### 15 — Mixing block dictionary and list (invalid at the same level)

```yaml
---

example_key_1: example_value_1
- example_list_entry_1

...
```

**Deliberately mixes** a mapping entry and a sequence entry at the same indentation level. Run this through `show_yaml_python.sh` to see how the parser handles (or rejects) it — a good example to intentionally break so you recognize the error shape later.

---

### 16 — Flow-style dictionary and list together

```yaml
---

{example_key_1: example_value_1}
[example_list_entry_1]

...
```

Similar idea to `15`, but using flow style for both. Compare the parsed result (or error) against `15`.

---

### 17 — Nested dictionaries

```yaml
---

example_key_1:
  sub_example_key1: sub_example_value1

example_key_2:
  sub_example_key2: sub_example_value2

...
```

Indentation creates nested mappings — `example_key_1` and `example_key_2` are each dictionaries containing their own sub-key. This nesting pattern is exactly how `vars:`, `group_vars`, and other structured playbook sections are built.

---

### 18 — Dictionary of lists

```yaml
---

example_1: 
  - item_1
  - item_2
  - item_3

example_2: 
  - item_4
  - item_5
  - item_6

...
```

Each top-level key maps to a nested list rather than a nested dictionary — the list equivalent of `17`'s nesting.

---

### 19 — Nested list of dictionaries of lists

```yaml
---

example_dictionary_1:
  - example_dictionary_2:
    - 1
    - 2
    - 3
  - example_dictionary_3:
    - 4
    - 5
    - 6
  - example_dictionary_4:
    - 7
    - 8
    - 9

...
```

Combines everything learned so far into one deeply nested structure: a dictionary containing a list of dictionaries, each of which contains its own list. Take time to trace through the indentation manually before running the helper script to confirm your mental model.

---

### `challenge` — Practice folder

Contains only `show_yaml_python.sh` with no accompanying `test.yaml` — the intent is for you to write your own `test.yaml` here, combining the concepts from `01`–`19`, and use the helper script to verify your understanding.

---

## Summary Table

| # | Concept | Key YAML Feature(s) | Notes |
|---|---------|----------------------|-------|
| 01 | Document markers | `---`, `...` | |
| 02 | Simple key/value pairs | Block mapping | |
| 03 | Quoting strings | No quotes / `"..."` / `'...'` | All equivalent here |
| 04 | Escape sequences | `\n` | Only expands inside double quotes |
| 05 | Literal block scalar | `\|` | Preserves line breaks |
| 06 | Folded block scalar | `>` | Folds line breaks into spaces |
| 07 | Folded block scalar, stripped | `>-` | Like `06`, but strips trailing newline |
| 08 | Integer | Bare number | Parsed as `int` |
| 09 | Quoted "integer" | `"1"` | Parsed as `str` |
| 10 | Booleans | `true`/`false`, `yes`/`no`, `on`/`off`, `y`/`n` | ⚠️ Quote ambiguous strings like country codes |
| 11 | List (block style) | `- item` | Same shape as a playbook's top level |
| 12 | Dictionary (block style) | `key: value` | |
| 13 | Dictionary (flow style) | `{ key: value }` | |
| 14 | List (flow style) | `[ item1, item2 ]` | |
| 15 | Mixed mapping/sequence (block) | Invalid combination | Intentional error example |
| 16 | Mixed mapping/sequence (flow) | Invalid combination | Intentional error example |
| 17 | Nested dictionaries | Indentation | |
| 18 | Dictionary of lists | Indentation | |
| 19 | Nested list of dicts of lists | Indentation | Combines everything above |
| `challenge` | Practice | — | Write your own `test.yaml` |

---

## Suggested Next Steps

1. Run `show_yaml_python.sh` in every folder `01`–`19` in order, predicting the parsed output before you look.
2. Deliberately break `15` and `16` further (e.g. more mixed levels) to build intuition for YAML's parser errors.
3. Write your own `test.yaml` in `challenge/` combining flow style, block style, quoting, and nesting.
4. Once comfortable, move on to `02-ansible-playbooks-breakdown-of-sections` to see this YAML knowledge applied inside real Ansible playbooks.
