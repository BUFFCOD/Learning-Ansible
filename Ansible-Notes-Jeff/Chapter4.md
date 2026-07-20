# Ansible Playbook Notes — Chapter 4

## Vagrant + Ansible Authentication

Vagrant VMs use public key authentication. To connect to them, add the insecure private key and the `vagrant` user to your inventory:

```ini
[all:vars]
ansible_ssh_private_key_file = /home/your_user/.vagrant.d/insecure_private_key
ansible_user = vagrant
```

This key works for every VM created with Vagrant.

---

## YAML Syntax in Playbooks

### Folded scalar (`>`)

Use `>` to write a long command across multiple lines; YAML folds it into a single line.

```yaml
- command: >
    cp httpd-vhosts.conf /etc/httpd/conf/httpd-vhosts.conf
```

### Document separator (`---`)

Each `---` starts a new play. A single file can contain multiple plays.

---

## Playbook Execution

### Common CLI options

| Flag | Short | Description |
|------|-------|-------------|
| `--inventory=PATH` | `-i` | Custom inventory file (default: `/etc/ansible/hosts`) |
| `--verbose` | `-v` | Verbose output; `-vvvv` for max detail |
| `--extra-vars=VARS` | `-e` | Pass variables as `"key=value,key=value"` |
| `--forks=NUM` | `-f` | Concurrent forks (>5 speeds up multi-server runs) |
| `--connection=TYPE` | `-c` | Connection type (default `ssh`; use `local` for cron/local runs) |
| `--check` | — | Dry run — shows changes without applying them |
| `--limit GROUP` | — | Restrict run to a subset of hosts, even if the play targets `all` |
| `--list-hosts` | — | Show which hosts the playbook would run against |
| `--force-handlers` | — | Run handlers even if a task later fails |

**Examples:**

```bash
ansible-playbook playbook.yml --limit webservers
ansible-playbook playbook.yml --list-hosts
```

---

## Privilege Escalation

Two approaches when the remote user isn't set in `ansible.cfg`:

**In the playbook:**

```yaml
- hosts: all
  become: yes
  become_user: root
  tasks:
    - name: Install httpd
      yum:
        name: httpd
        state: present
```

**On the command line:**

```bash
ansible-playbook playbook.yml -u root
```

---

## Debugging Failed Services with Ad-Hoc Commands

When a service fails to start:

```
fatal: [10.19.30.23]: FAILED! => {"changed": false,
"msg": "Unable to start service httpd: Job for httpd.service failed
because the control process exited with error code."}
```

Don't guess — inspect the host directly:

```bash
ansible webservers -b -a "systemctl status httpd.service"
ansible webservers -b -a "journalctl -xeu httpd.service"
```

- `-b` — become root
- `-a` — arguments passed to the module (default: `command`)

---

## Handlers

A handler is a task that runs **only when notified** by another task that reported `changed`. Typical use: restart a service after its config file changes.

**Task that notifies:**

```yaml
- name: Update httpd-vhost.conf
  copy:
    src: httpd-vhost.conf
    dest: /etc/httpd/conf/httpd-vhosts.conf
  notify: restart httpd
```

**Handler definition:**

```yaml
handlers:
  - name: restart httpd
    service:
      name: httpd
      state: restarted
```

### Key behaviors

- Handlers run **once**, at the end of the play, even if notified many times.
- Handlers can live in a separate file and be included, like variables.
- **Gotcha:** if any task fails, remaining handlers are skipped — the service never restarts. Use `--force-handlers` to run them anyway.

---

## Loops: `with_*` vs `loop`

### What `with_items` actually is

`with_items` is two parts glued together:

- **`with_`** — "loop this task using a lookup plugin"
- **`items`** — the *name of the lookup plugin*

Lookup plugins are built-in programs that fetch or generate lists. The plugin name is fixed by Ansible, not chosen by you.

| Keyword | Plugin | Loops over |
|---|---|---|
| `with_items` | `items` | a plain list |
| `with_fileglob` | `fileglob` | files matching a pattern (`*.conf`) |
| `with_dict` | `dict` | key/value pairs of a dictionary |
| `with_sequence` | `sequence` | a number range (1, 2, 3…) |
| `with_lines` | `lines` | output lines of a command |

`with_ja` would make Ansible look for a lookup plugin named `ja`, find nothing, and error out.

### The two names in a loop task

1. **`items`** → must match a real plugin (not yours to rename)
2. **`item`** → the automatic variable holding the current value (*can* be renamed)

Rename `item` with `loop_control`:

```yaml
loop:
  - apache2
  - mysql
loop_control:
  loop_var: ja
```

### Looping over dictionaries

```yaml
- name: Create users.
  user:
    name: "{{ item.name }}"
    groups: "{{ item.group }}"
  loop:
    - { name: 'alice', group: 'admin' }
    - { name: 'bob',   group: 'dev' }
```

---

## The `lineinfile` Module

Adds a line to a file if it's missing, or replaces a matching line if present.

```yaml
- name: Adjust OpCache memory setting.
  lineinfile:
    dest: "/etc/php/7.4/apache2/conf.d/10-opcache.ini"
    regexp: "^opcache.memory_consumption"
    line: "opcache.memory_consumption = 96"
    state: present
  notify: restart apache
```

- `regexp` matches an existing line → replaced with `line`
- No match → `line` is appended to the end of the file