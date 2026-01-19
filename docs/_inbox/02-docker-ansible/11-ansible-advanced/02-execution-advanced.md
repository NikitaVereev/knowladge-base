---
title: 02 Advanced Execution Control
---

---

## Loops

```yaml
---
# Install packages
- name: Install packages
  apt:
    name: "{{ item }}"
  loop:
    - nginx
    - curl
    - git
    - python3-pip

# Create users
- name: Create users
  user:
    name: "{{ item.name }}"
    shell: "{{ item.shell }}"
  loop:
    - name: alice
      shell: /bin/bash
    - name: bob
      shell: /bin/sh
```

---

## Lookups

```yaml
---
# Read file
- name: Load configuration
  set_fact:
    config: "{{ lookup('file', 'files/config.json') }}"

# Environment variables
- name: Get home directory
  set_fact:
    home: "{{ lookup('env', 'HOME') }}"

# Execute command
- name: Get current date
  set_fact:
    timestamp: "{{ lookup('pipe', 'date +%Y-%m-%d') }}"

# Generate password
- name: Create database password
  set_fact:
    db_password: "{{ lookup('password', '/dev/null', length=32) }}"

# Load variables dynamically
- name: Load vars from file
  include_vars:
    file: "{{ lookup('first_found', dict(files=vars_files, skip=true)) }}"
```

---

## Filters

**String filters:**
```yaml
- name: String operations
  debug:
    msg: |
      Upper: {{ 'hello' | upper }}
      Lower: {{ 'HELLO' | lower }}
      Replace: {{ 'hello world' | replace('world', 'ansible') }}
      Split: {{ 'a,b,c' | split(',') | list }}
      Join: {{ ['a', 'b', 'c'] | join(',') }}
```

**List filters:**
```yaml
- name: List operations
  debug:
    msg: |
      Unique: {{ [1,2,2,3] | unique | list }}
      Sorted: {{ [3,1,2] | sort }}
      Reversed: {{ [1,2,3] | reverse | list }}
      Min: {{ [1,5,3] | min }}
      Max: {{ [1,5,3] | max }}
      Sum: {{ [1,2,3] | sum }}
```

**JSON/YAML filters:**
```yaml
- name: Format data
  debug:
    msg: |
      JSON: {{ data | to_json }}
      Pretty JSON: {{ data | to_nice_json }}
      YAML: {{ data | to_yaml }}
```

---

## Tags

```yaml
---
- name: Complex playbook
  hosts: all
  
  tasks:
    - name: Install packages
      apt:
        name: nginx
      tags: [install, packages]

    - name: Configure nginx
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      tags: [config, nginx]

    - name: Start service
      systemd:
        name: nginx
        state: started
      tags: [service, always]

    - name: Verify installation
      uri:
        url: http://localhost/
        status_code: 200
      tags: [verify]
```

**Execute with tags:**
```bash
# Run only install tasks
ansible-playbook site.yml --tags install

# Run config and verify
ansible-playbook site.yml --tags "config,verify"

# Skip nginx tasks
ansible-playbook site.yml --skip-tags nginx

# Run always tasks
ansible-playbook site.yml --tags always
```

---

## Jinja2 in Variables

```yaml
---
- name: Use Jinja2 in variables
  hosts: all
  
  vars:
    app_name: myapp
    app_version: 1.0.0
    # Concatenation
    app_dir: "/opt/{{ app_name }}"
    # Conditionals
    port: "{{ 443 if use_ssl else 80 }}"
    # Loops
    packages: "{{ ['nginx', 'curl'] | join(',') }}"
    
  tasks:
    - name: Deploy
      debug:
        msg: "Deploying {{ app_name }} v{{ app_version }} to {{ app_dir }}"
```

---

**Следующее:** [[03-secrets-vault|Secrets Management]]
