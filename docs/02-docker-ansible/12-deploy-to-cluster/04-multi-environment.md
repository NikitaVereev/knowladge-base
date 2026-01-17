---
title: 04 Multi-Environment Deployment
---

---

## Environment Structure

```
ansible-deploy/
├── inventory/
│   ├── development
│   ├── staging
│   └── production
├── group_vars/
│   ├── development.yml
│   ├── staging.yml
│   └── production.yml
├── host_vars/
│   ├── dev-swarm-01.yml
│   └── prod-swarm-01.yml
├── roles/
│   └── deploy/
├── deploy-dev.yml
├── deploy-staging.yml
└── deploy-prod.yml
```

---

## Environment Variables

**group_vars/development.yml:**
```yaml
---
environment: development
registry: localhost:5000
app_version: latest
replicas: 1
resource_limits_cpu: "0.25"
resource_limits_memory: 256M
log_level: debug
db_backup_enabled: no
```

**group_vars/production.yml:**
```yaml
---
environment: production
registry: docker.io
app_version: "{{ git_version }}"
replicas: 5
resource_limits_cpu: 1
resource_limits_memory: 1G
log_level: info
db_backup_enabled: yes
```

---

## Deployment Playbooks

**deploy-dev.yml:**
```yaml
---
- name: Deploy to development
  hosts: dev_swarm
  
  vars:
    env_name: development
    
  tasks:
    - name: Pull latest code
      git:
        repo: "{{ repo_url }}"
        dest: /tmp/app
        version: develop

    - name: Build image
      docker_image:
        name: "{{ registry }}/app"
        tag: latest
        source: build
        build:
          path: /tmp/app

    - name: Deploy service
      community.docker.docker_swarm_service:
        name: app
        image: "{{ registry }}/app:latest"
        replicas: "{{ replicas }}"
```

**deploy-prod.yml:**
```yaml
---
- name: Deploy to production
  hosts: prod_swarm
  
  vars:
    env_name: production
    
  pre_tasks:
    - name: Backup database
      shell: |
        docker exec {{ service_name }}_db \
          pg_dump -U postgres > /backup/db_$(date +%Y%m%d).sql

  tasks:
    - name: Deploy service
      community.docker.docker_swarm_service:
        name: app
        image: "{{ registry }}/app:{{ app_version }}"
        replicas: "{{ replicas }}"
        resource_limits:
          cpus: "{{ resource_limits_cpu }}"
          memory: "{{ resource_limits_memory }}"

  post_tasks:
    - name: Verify deployment
      uri:
        url: "http://localhost/health"
        status_code: 200
```

---

## Conditional Deployment

```yaml
---
- name: Deploy service
  hosts: "{{ target_env }}_swarm"
  
  tasks:
    - name: Deploy with replicas
      community.docker.docker_swarm_service:
        name: app
        image: "{{ registry }}/app:{{ app_version }}"
        replicas: "{{ replicas }}"
        resource_limits:
          cpus: "{{ resource_limits_cpu }}"
          memory: "{{ resource_limits_memory }}"

    - name: Setup backups
      include_tasks: backups.yml
      when: db_backup_enabled | bool
```

---

## Secrets per Environment

```yaml
---
- name: Setup secrets
  hosts: "{{ target_env }}_swarm"
  
  vars_files:
    - "secrets/{{ target_env }}.yml"
  
  tasks:
    - name: Create database secret
      community.docker.docker_secret:
        name: db_password
        data: "{{ db_password }}"
        state: present
```

---

**Следующее:** [[05-updates-rollback|Updates & Rollback]]
