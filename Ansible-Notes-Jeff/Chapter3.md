# Ad-Hoc Commands

## Why Use Ad-Hoc Commands

- Quickly check on a system resource
- Run an emergency command
- Manage users and groups quickly
- Reboot a server

## Vagrant & Provider Notes

On Fedora 44, use **libvirt** as the Vagrant provider.

When defining a VM's provider specs, include:

```ruby
v.qemu_use_session = false
v.uri = "qemu:///system"
```

You can also set `v.linked_clone = true` for faster VM cloning.

**Switching to VirtualBox:** since this machine normally runs libvirt, using VirtualBox instead requires stopping/unloading the KVM modules first:

```bash
sudo systemctl stop libvirtd
sudo modprobe -r kvm_intel
sudo modprobe -r kvm
```

## Inventory Tips

You can combine groups using `:children` to create a group made up of other groups.

```ini
[app]
192.168.60.4
192.168.60.5

# Database server
[db]
192.168.60.6

# Group 'multi' contains all servers.
# 'children' is a special keyword that allows grouping of other groups.
[multi:children]
app
db
```

## Anatomy of an Ad-Hoc Command

```bash
ansible multi -a "hostname" -f 1
```

| Part | Meaning |
|---|---|
| `multi` | the group name being targeted |
| `-a "hostname"` | the argument/command to run |
| `-f 1` | "forks" — how many hosts to run on at once (here, one at a time) |

```bash
ansible multi -b -m yum -a "name=chrony state=present"
```

| Part | Meaning |
|---|---|
| `-b` | run the command as sudo (`become`) |
| `-m yum` | the module to run |

## Limiting Hosts

```bash
# Limit hosts with a simple pattern (asterisk is a wildcard).
ansible app -b -a "service ntpd restart" --limit "*.4"

# Limit hosts with a regular expression (prefix with a tilde).
ansible app -b -a "service ntpd restart" --limit ~".*\.4"
```

`--limit` restricts the command to a specific host (or hosts) within the targeted group. It matches either an exact string or a regular expression (when prefixed with `~`).

## Cross-Platform Package Installation

To install a generic package like `git` across different distros (Debian, RHEL, Fedora, Ubuntu, CentOS, FreeBSD, etc.), use the generic `package` module instead of a distro-specific one (`yum`, `apt`, etc.):

```bash
ansible app -b -m package -a "name=git state=present"
```

## Copying Files

```bash
ansible multi -m copy -a "src=/etc/hosts dest=/tmp/hosts"
```

`src` can be a file or a directory:
- **Trailing slash** (`src=/some/dir/`) → only the *contents* of the directory are copied.
- **No trailing slash** (`src=/some/dir`) → the directory itself (and its contents) is copied to the destination.

## Fetching Files

```bash
ansible multi -b -m fetch -a "src=/etc/hosts dest=/tmp"
```

`fetch` works like `copy`, but in reverse: `src` is on the remote machine, `dest` is on your local machine.

## Managing Files & Directories

Create a directory:

```bash
ansible multi -m file -a "dest=/tmp/test mode=644 state=directory"
```

Create a symlink (`state=link`):

```bash
ansible multi -m file -a "src=/src/file dest=/dest/symlink state=link"
```

## Running Tasks Asynchronously

For ad-hoc commands or playbooks that take a while to finish, you can run them asynchronously:

- `-P <seconds>` — poll interval; Ansible gives you a status update every N seconds
- `-B <seconds>` — the max time (in seconds) to let the task run in the background

While a background task is running, you can check on it elsewhere using the `async_status` module, as long as you have the `ansible_job_id` value to pass in as `jid`:

```bash
ansible multi -b -m async_status -a "jid=169825235950.3572"
```

> Reference: [ansible/ansible#14681](https://github.com/ansible/ansible/issues/14681)

## Managing Cron Jobs

Ansible's `cron` module makes managing cron jobs easy. To run a shell script on all servers every day at 4 a.m.:

```bash
ansible multi -b -m cron -a "name='daily-cron-all-servers' hour=4 job='/path/to/daily-script.sh'"
```