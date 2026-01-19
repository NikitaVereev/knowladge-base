---
title: 06 Complete Deployment Project
---

---

## Full Stack Architecture

```
Production Swarm Cluster:
├── Manager Nodes (3)
│   ├── Docker Engine
│   └── Swarm Manager
├── Worker Nodes (5)
│   ├── Docker Engine
│   └── App Containers
└── Storage
    ├── PostgreSQL volumes
    ├── Redis data
    └── Backups

Services:
├── nginx (reverse proxy)
├── app (Node.js)
├── database (PostgreSQL)
└── cache (Redis)
```

---

## Complete Deployment Playbook

```yaml
---
- name: Complete application deployment
  hosts: swarm_managers
  
  vars:
    app_name: myapp
    version: "{{ lookup('pipe', 'git describe --tags --always') }}"
    registry: docker.io
    environment: production

  pre_tasks:
    - name: Backup database
      shell: |
        docker exec {{ app_name }}_db pg_dump -U postgres \
          > /backup/db_$(date +%Y%m%d_%H%M%S).sql

  tasks:
    - name: Deploy services from stack
      community.docker.docker_stack:
        state: present
        name: "{{ app_name }}"
        compose:
          - /tmp/docker-compose.yml

    - name: Wait for services
      shell: |
        docker service ls --filter "label=com.docker.stack.namespace={{ app_name }}"
      register: services
      until: "'Running' in services.stdout"
      retries: 10
      delay: 5

  post_tasks:
    - name: Validate deployment
      import_tasks: validation.yml

    - name: Show deployment status
      shell: "docker service ls"
      register: status
```

---

## Docker Compose Stack

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - nginx_config:/etc/nginx/conf.d:ro
    deploy:
      replicas: 2
      placement:
        constraints: [node.role == manager]
      healthcheck:
        test: ["CMD", "wget", "-q", "--spider", "http://localhost/health"]
        interval: 30s
        timeout: 3s
        retries: 3

  app:
    image: "{{ registry }}/{{ app_name }}:{{ version }}"
    environment:
      - NODE_ENV=production
      - DB_HOST=db
      - REDIS_HOST=cache
      - LOG_LEVEL=info
    deploy:
      replicas: 5
      resource_limits:
        cpus: '1'
        memory: 1G
      resource_reservations:
        cpus: '0.5'
        memory: 512M
      healthcheck:
        test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
        interval: 30s
        timeout: 3s
        retries: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - db_data:/var/lib/postgresql/data
    deploy:
      replicas: 1
      placement:
        constraints: [node.role == manager]
      healthcheck:
        test: ["CMD-SHELL", "pg_isready -U postgres"]
        interval: 10s
        timeout: 5s
        retries: 5

  cache:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    deploy:
      replicas: 1
      placement:
        constraints: [node.role == manager]
      healthcheck:
        test: ["CMD", "redis-cli", "ping"]
        interval: 10s
        timeout: 3s
        retries: 3

volumes:
  db_data:
    driver: local
  redis_data:
    driver: local
  nginx_config:
    driver: local

secrets:
  db_password:
    external: true
```

---

## Deployment Validation

```yaml
---
- name: Validate deployment
  hosts: swarm_managers
  
  tasks:
    - name: Check all services running
      shell: |
        docker service ls --filter label=com.docker.stack.namespace={{ app_name }} \
          --format '{{.Replicas}}'
      register: services
      failed_when: "'0/' in services.stdout"

    - name: Check health endpoints
      uri:
        url: "http://localhost{{ item }}"
        status_code: 200
      loop:
        - /health
        - /api/status

    - name: Monitor service logs
      shell: |
        docker service logs {{ app_name }}_app --tail 50
      register: logs

    - name: Verify database
      shell: |
        docker exec {{ app_name }}_db psql -U postgres -c "SELECT 1"
      register: db_check

    - name: Show deployment summary
      debug:
        msg: |
          Services: {{ services.stdout }}
          Logs: {{ logs.stdout_lines }}
```

---

## Deployment Commands

```bash
# Deploy complete stack
ansible-playbook deploy-complete.yml -i inventory/production

# Monitor deployment
ansible-playbook monitor.yml -i inventory/production

# Check service status
ansible-playbook status.yml -i inventory/production

# Rollback if needed
ansible-playbook rollback.yml -i inventory/production --extra-vars "version=1.2.1"

# View logs
ansible-playbook logs.yml -i inventory/production
```

---
