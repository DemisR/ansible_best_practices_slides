# Ansible best practices

---
## Directory Layout

---

```none
inventories/
    group_vars/
        group1                 # here we assign variables to particular groups
        group2 
    host_vars/
        hostname1              # if systems need specific variables, put them here
        hostname2 
    staging
    production

playbooks/
    site.yml                  
    webservers.yml            
    dbservers.yml

roles/
    common/               # this hierarchy represents a "role"
        tasks/            #
            main.yml      #  <-- tasks file can include smaller files if warranted
        handlers/         #
            main.yml      #  <-- handlers file
        templates/        #  <-- files for use with the template resource
            ntp.conf.j2   #  <------- templates end in .j2
        files/            #
            bar.txt       #  <-- files for use with the copy resource
            foo.sh        #  <-- script files for use with the script resource
        vars/             #
            main.yml      #  <-- variables associated with this role
        defaults/         #
            main.yml      #  <-- default lower priority variables for this role
        meta/             #
            main.yml      #  <-- role dependencies

    webtier/              # same kind of structure as "common" was above, done for the webtier role
    monitoring/
    fooapp/
```
@[1-9](Inventories)
@[11-14](Playbooks)
@[16-36](Roles)
---

## Optimize your Ansible content for readability

---
*Name* Your Plays and Tasks
```yaml
- name: "Create {{ springboot_app }} systemd service file"
  template:
    src: 'springboot_service.j2'
    dest: '/lib/systemd/system/{{ item.name }}.service'
  notify: reload systemd service
  with_items: '{{ springboot_app }}'
  become: yes
```
@[1]
---
### Use Native YAML Syntax

```yaml
- name: install telegraf
  apt:
    name: "telegraf-{{ telegraf_version }}" state=present update_cache=yes disable_gpg_check=yes enablerepo=telegraf
  notify: restart telegraf
```
@[3](Here is an example of some tasks using the key=value shorthand)
```yaml
- name: install telegraf
  apt: 
    name: "telegraf-{{ telegraf_version }}"
    state: present
    update_cache: yes
    disable_gpg_check: yes
    enablerepo: telegraf
  notify: restart telegraf
```
@[7-11](Now here is the same tasks using native YAML syntax)
---

## Variables

---
Use prefixes and human meaningful names with variables

```yaml
apache_max_keepalive: 25
apache_port: 80
tomcat_port: 8080
```
Role variables should be **prefixed with the role name**

---

You can compose variables
```yaml
wildfly_mode: standalone
wildfly_home_path: "/usr/local/wildfly"
wildfly_conf_path: "{{ wildfly_home_path }}/{{ wildfly_mode }}/configuration"
```

---

# ...

- Define naming convention for your role, vars , groups...
- Variables should be specified at exactly one place
(or two places if a variable has a reasonable, overridable default value), as close as possible to their usage
- Check variable precendence ( for roles basically use **default** dir ).
- Set a default for every variable

---

Lookup plugins allow Ansible to access data from outside sources

Example for password generation
```yaml
wildfly_management_users:
  - name: admin
    password: '{{ lookup("password", secret + "/wildfly/" + ansible_fqdn + "/management_users/admin length=20 " + "chars=ascii_letters,digits,-_.") }}'

```

---

## Roles

---
Use galaxy command for create roles structure 

```shell
ansible-galaxy init rolename
```


Add **meta** info and use **roles dependencies**
```yaml
galaxy_info:
  author: Demis Rizzotto
  description: Setup and configure Wildfly runtime
  company: Lampiris
  license: BSD

  min_ansible_version: 2.1

  galaxy_tags:
    - wildfly

dependencies:
  - { role: java-jdk, java_jdk_version: oracle-java8-jdk }
  - { role: lampiris-apt-repo }
  - role: debops.secret
```
---

# ...
- use `handlers`  
- use multiple files for tasks in roles
- Prefer templates than files
- Use `{{ ansible_managed }}` to mark auto-generated files as such, so nobody unknowingly edits them manually

---

## Tasks
---
Idempotency done right.Avoid skipping items.

Use modules before run commands
if no choice use `changed_when`
```yaml
- name: Run drush to finish setting it up
  command: "{{ drush_path }}"
  register: drush_result
  changed_when: "'Execute a drush command' not in drush_result.stdout"
  become: no
```
If no module does what you want, you can create your own

---

## Tests

---
Test your roles
Start with `--syntax-check`
You can use `--check` option for test yours playbooks
And `--diff` showing differences in files 

```shell
ansible-playbook foo.yml -i staging --check --diff --limit foo.example.com
```

---
## Tools

---
Ansible galaxy
Some editor have syntax higthlight and linters
Ex : **VScode**: (language-Ansible and YAML Support by Red Hat)

--- 
`ansible-lint` checks playbooks for practices and behaviour that could potentially be improved
Ex:
```None
[ANSIBLE0013] Use shell only when shell functionality is required
/ansible/roles/common/tasks/admins.yml:34
Task/Handler: Check if infrastructuremanagers user exists

[ANSIBLE0011] All tasks should be named
/ansible/roles/common/tasks/exim4.yml:2
Task/Handler: stat __file__=/ansible/roles/common/tasks/exim4.yml __line__=2 path=/etc/postfix/main.cf
```
---
`ansible-doc` is very useful for finding the syntax of a module without having to look it up online
```shell
ansible-doc mysql_user
```
---
Use verstion control (**git**) 
(.gitignore : `*.retry`)
and also for test syntax before merge code


# Inventories
Use directoriers for `host_vars` and `group_vars` 


# Execution
Use tags only for limiting to tasks for speed reasons, as in "only update config files". 

**sudo** (`become`) only where necessary

---

use `when:` in yout task

`with_items` example

registered variables used to store the results of a task in a playbook.


ansible-vault

---
# Tips and tricks

List all facts for one host
Verbose mode

--limit

