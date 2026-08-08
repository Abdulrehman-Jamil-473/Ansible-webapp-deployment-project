# Ansible Practice Project — Web App Deployment Automation

A mini real-world project covering ad-hoc commands, playbooks, roles, templates (Jinja2), variables, loops, conditionals, handlers, error handling, and Vault.

## 📋 Table of Contents
- [Scenario](#scenario)
- [Folder Structure](#folder-structure)
- [1. Inventory](#1-inventory)
- [2. Ad-hoc Commands (Warm-up)](#2-ad-hoc-commands-warm-up)
- [3. Role: webserver](#3-role-webserver)
- [4. Role: dbserver](#4-role-dbserver)
- [5. Role: appusers](#5-role-appusers)
- [6. Vault — Secure Password](#6-vault--secure-password)
- [7. Master Playbook](#7-master-playbook)
- [8. Running the Playbook](#8-running-the-playbook)
- [9. Verification Commands](#9-verification-commands)
- [Concept Coverage](#concept-coverage)

---

## Scenario

Deploy a simple web application across two server groups (`webservers`, `dbservers`) using Ansible automation — covering package installation, dynamic config deployment, user management, and secure credential handling.

---

## Folder Structure

```
project/
├── inventory.ini
├── site.yml
├── secrets.yml (vault encrypted)
├── roles/
│   ├── webserver/
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   ├── templates/index.html.j2
│   │   └── defaults/main.yml
│   ├── dbserver/
│   │   ├── tasks/main.yml
│   │   └── handlers/main.yml
│   └── appusers/
│       ├── tasks/main.yml
│       └── defaults/main.yml
```

---

## 1. Inventory

**`inventory.ini`**
```ini
[webservers]
172.31.33.111

[dbservers]
172.31.44.154

[all:vars]
ansible_connection=ssh
ansible_become=true
```

---

## 2. Ad-hoc Commands (Warm-up)

```bash
# Connectivity check
ansible all -m ping -i inventory.ini

# Disk space check
ansible all -m shell -a "df -h" -i inventory.ini
```

---

## 3. Role: webserver

**`roles/webserver/defaults/main.yml`**
```yaml
---
app_env: "staging"
```

**`roles/webserver/templates/index.html.j2`**
```jinja2
<html>
<head><title>Web App</title></head>
<body>
  <h1>Server: {{ ansible_hostname }}</h1>
  <p>Environment: {{ app_env }}</p>
</body>
</html>
```

**`roles/webserver/tasks/main.yml`**
```yaml
---
- name: Install nginx
  yum:
    name: nginx
    state: present

- name: Deploy index.html
  template:
    src: index.html.j2
    dest: /usr/share/nginx/html/index.html
  notify: Reload nginx

- name: Ensure nginx is running
  service:
    name: nginx
    state: started
    enabled: yes
```

**`roles/webserver/handlers/main.yml`**
```yaml
---
- name: Reload nginx
  service:
    name: nginx
    state: reloaded
```

---

## 4. Role: dbserver

**`roles/dbserver/tasks/main.yml`**
```yaml
---
- name: Install mysql-server on CentOS/RedHat only
  yum:
    name: mysql-server
    state: present
  when: ansible_facts['distribution'] in ["CentOS", "RedHat"]
  notify: Start mysql service

- name: Try stopping old legacy service (may not exist)
  service:
    name: old-db-service
    state: stopped
  ignore_errors: yes

- name: Set MySQL root password
  mysql_user:
    name: root
    password: "{{ db_root_password }}"
    login_unix_socket: /var/lib/mysql/mysql.sock
  no_log: true
```

**`roles/dbserver/handlers/main.yml`**
```yaml
---
- name: Start mysql service
  service:
    name: mysqld
    state: started
    enabled: yes
```

> `no_log: true` prevents the password task's output from being printed to console/logs — an extra layer of security.

---

## 5. Role: appusers

**`roles/appusers/defaults/main.yml`**
```yaml
---
user_list:
  - appuser1
  - appuser2
  - appuser3
```

**`roles/appusers/tasks/main.yml`**
```yaml
---
- name: Create application users
  user:
    name: "{{ item }}"
    state: present
  loop: "{{ user_list }}"

- name: Confirm user creation
  debug:
    msg: "User {{ item }} created successfully"
  loop: "{{ user_list }}"
```

---

## 6. Vault — Secure Password

Create the encrypted secrets file:
```bash
ansible-vault create secrets.yml
```
Set a password when prompted, then enter:
```yaml
db_root_password: "MySecureP@ss123"
```
Save and exit.

---

## 7. Master Playbook

**`site.yml`**
```yaml
---
- name: Deploy web application infrastructure
  hosts: webservers:dbservers
  become: yes
  vars_files:
    - secrets.yml
  roles:
    - role: webserver
      when: inventory_hostname in groups['webservers']
    - role: dbserver
      when: inventory_hostname in groups['dbservers']
    - role: appusers
```

---

## 8. Running the Playbook

```bash
ansible-playbook site.yml -i inventory.ini --ask-vault-pass
```

---

## 9. Verification Commands

```bash
# Webserver check
ansible webservers -m command -a "cat /usr/share/nginx/html/index.html" -i inventory.ini

# DB service check
ansible dbservers -m command -a "systemctl status mysqld" -i inventory.ini

# Users check
ansible all -m command -a "id appuser1" -i inventory.ini
```

---

## Concept Coverage

| Topic | Where Used |
|---|---|
| Ad-hoc commands | Warm-up connectivity/disk check |
| Idempotency | `yum`, `service`, `user` modules |
| Playbook | `site.yml` |
| Roles | `webserver`, `dbserver`, `appusers` |
| Templates (Jinja2) | `index.html.j2` (variable interpolation) |
| Handlers | Nginx reload, MySQL start |
| Loops | Creating users |
| Conditionals (`when`) | OS-based install, role selection per host group |
| Error handling | `ignore_errors` |
| Vault | Secure storage of `db_root_password` |
