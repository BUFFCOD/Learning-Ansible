## Learn ##

When use vagrant. It uses public key authentication so if you wish to make a vm  for you to login normally you need to add this your inventory file. 

```ini
[all:vars]
ansible_ssh_private_key_file = /home/your_user/.vagrant.d/insecure_private_key
```

and also include the vagrant user in your inventory file. 

```ini
[all:vars]
ansible_ssh_private_key_file = /home/your_user/.vagrant.d/insecure_private_key
ansible_user = vagrant
```
it will use this private key to login into every vm that you create with vagrant.





In ansible you can use > to make a list of commands run as a single command. 
```
- command: >
cp httpd-vhosts.conf /etc/httpd/conf/httpd-vhosts.conf

```


In ansible every time you use --- it means that you are starting a new playbook. so one file could have multiple playbooks.


ansible-playbook playbook.yml --limit webservers

you can use the ansible --limit option to limit the playbook to a specific group of hosts even if the playbook is targeting all hosts.


ansible-playbook playbook.yml --list-hosts


ansible --list-host will allow you see all the host that this playbook will run on.




There are different ways a setting up sudo in playbook. if you didn't set up the remote user in your config file you can use the become option in your playbook or use the -u option to specify the remote user. 

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

or you can use the -u option to specify the remote user. 

```bash
ansible-playbook playbook.yml -u root
```


Other ansible-playbook options that you can use are:
## Ansible Playbook CLI Options

| Flag | Short | Description |
|------|-------|--------------|
| `--inventory=PATH` | `-i` | Custom inventory file (default: `/etc/ansible/hosts`) |
| `--verbose` | `-v` | Verbose output; use `-vvvv` for max detail |
| `--extra-vars=VARS` | `-e` | Pass variables as `"key=value,key=value"` |
| `--forks=NUM` | `-f` | Number of concurrent forks (>5 speeds up multi-server runs) |
| `--connection=TYPE` | `-c` | Connection type (default `ssh`; use `local` for local/cron runs) |
| `--check` | — | Dry run — shows changes without applying them |





if you receive an error like this: 

```See "systemctl status httpd.service" and "journalctl -xeu httpd.service" for details.

Origin: /home/wlouis/Vagrant/Chapter4/play1-update.yml:25:7

23         - src: httpd-vhost.conf
24           dest: /etc/httpd/conf/httpd-vhosts.conf
25     - name: Make sure Apache is started now and at boot.
         ^ column 7

fatal: [10.19.30.23]: FAILED! => {"changed": false, "msg": "Unable to start service httpd: Job for httpd.service failed because the control process exited with error code.\nSee \"systemctl status httpd.service\" and \"journalctl -xeu httpd.service\" for details.\n"}
```
remeber you can use adhoc commands to check the status of the service and see what is wrong. 

```bash
ansible webservers -b -a "systemctl status httpd.service"
ansible webservers -b -a "journalctl -xeu httpd.service"
```

the -b option is used to run the command as root.
the -a option is used to run the command as an argument.




In ansible we have something called a handler. A handler is a task that is only run when a specific task notifies it that it changed something. Handlers are typically used to restart services after a configuration file has been changed. 

```yaml
- name: Update httpd-vhost.conf
  copy:
    src: httpd-vhost.conf
    dest: /etc/httpd/conf/httpd-vhosts.conf
  notify:
    - restart httpd
```
handlers are defined in the same playbook as the tasks that notify them. 

```yaml
handlers:
  - name: restart httpd
    service:
      name: httpd
      state: restarted

```

simliar to  variables, handlers can be defined in a separate file and included in the playbook.  but  handlers will not run if a task in the playbook fails.  so if you have a task that is supposed to update a configuration file and it fails, the handler will not run and the service will not be restarted. to prevent this from happening you can use --force-handlers option when running the playbook. this will force all handlers to run even if a task fails.


## What `with_*` actually is

`with_items` isn't one keyword — it's two parts glued together:

- **`with_`** = "loop this task using a lookup plugin"
- **`items`** = the name of the lookup plugin to use

Lookup plugins are little programs built into Ansible that fetch or generate lists of data. Ansible ships with a bunch of them, each with a fixed name:

| Keyword | Plugin used | What it loops over |
|---|---|---|
| `with_items` | `items` | a plain list |
| `with_fileglob` | `fileglob` | files matching a pattern (`*.conf`) |
| `with_dict` | `dict` | key/value pairs of a dictionary |
| `with_sequence` | `sequence` | a number range (1, 2, 3...) |
| `with_lines` | `lines` | output lines of a command |

So when Ansible sees `with_items`, it literally goes and finds the plugin named `items`, feeds it your list, and runs the task once per result.

## How this relates to your question

You were treating `items` like a variable name you chose — and asking if you could choose `ja` instead. But `items` was never yours to name. `with_ja` would make Ansible look for a lookup plugin called `ja`, find nothing, and error out.

The two names in that task are both fixed by Ansible, not by you:

1. **`items`** → must match a real plugin
2. **`item`** → the automatic variable holding the current loop value

The only one you can rename is `item`, and only via `loop_control: loop_var: ja` — which tells Ansible "put each value in a variable called `ja` instead of `item`."



loop:
  - apache2
  - mysql
loop_control:
  loop_var: ja

  - name: Create users.
  user:
    name: "{{ item.name }}"
    groups: "{{ item.group }}"
  loop:
    - { name: 'alice', group: 'admin' }
    - { name: 'bob',   group: 'dev' }


In ansible there is a module called lineinfile. This module is used to add a line to a file if it doesn't already exist or to change a line in a file if it does exist. 

```yaml
- name: Adjust OpCache memory setting.
lineinfile:
dest: "/etc/php/7.4/apache2/conf.d/10-opcache.ini"
regexp: "^opcache.memory_consumption"
line: "opcache.memory_consumption = 96"
state: present
notify: restart apache
```

This task will check the file /etc/php/7.4/apache2/conf.d/10-opcache.ini for a line that starts with opcache.memory_consumption. If it finds one, it will replace it with opcache.memory_consumption = 96. If it doesn't find one, it will add the line to the end of the file.


