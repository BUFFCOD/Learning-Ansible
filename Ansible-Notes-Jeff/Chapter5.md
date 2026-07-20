# Chapter 5 - Ansible Playbooks - Beyond the Basics



learned


You can use the notify option to  nofity multiple handlers in a playbook. This is useful when you want to trigger multiple actions based on a single event. For example, if you have a task that updates a configuration file, you might want to notify both a service restart handler and a logging handler.


 ```
- name: Rebuild application configuration.
command: /opt/app/rebuild.sh
notify:
- restart apache
- restart memcached

handlers:
- name: restart apache
service: name=apache2 state=restarted
notify: restart memcached
- name: restart memcached
service: name=memcached state=restarted

```


 handlers  only run once  at the end of the playbook. if you want to run the handler in the middle of the playbook you can use the meta: flush_handlers directive. This will force Ansible to run all notified handlers immediately, rather than waiting until the end of the playbook. If the play fails, you can use the --force-handlers option to force the handlers to run even if the playbook fails or use the meta module to force the handlers to run at a specific point in the playbook. 



