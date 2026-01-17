---
title: 02 Docker Swarm Deployment
---

---

## Service Definition

```yaml
---
- name: Deploy service to Swarm
  hosts: swarm_managers
  
  tasks:
    - name: Deploy web service
      community.docker.docker_swarm_service:
        name: web
        image: "{{ registry }}/app:{{ version }}"
        mode: replicated
        replicas: 3
        publish:
          - protocol: tcp
            published_port: 80
            target_port: 3000
        env:
          - "NODE_ENV=production"
          - "LOG_LEVEL=info"
        resource_limits:
          cpus: 0.5
          memory: 512M
        resource_reservations:
          cpus: 0.25
          memory: 256M
        placement:
          constraints:
            - node.role == worker
        healthcheck:
          test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
          interval: 30s
          timeout: 3s
          retries: 3
        update_delay: 10s
        update_parallelism: 1
        update_failure_action: rollback
```

---

## Stack Deployment

```yaml
---
- name: Deploy stack
  hosts: swarm_managers
  
  vars:
    stack_name: myapp
    
  tasks:
    - name: Copy compose file
      template:
        src: docker-compose.yml.j2
        dest: /tmp/docker-compose.yml

    - name: Deploy stack
      community.docker.docker_stack:
        state: present
        name: "{{ stack_name }}"
        compose:
          - /tmp/docker-compose.yml

    - name: Wait for services
      shell: |
        docker service ls --filter "label=com.docker.stack.namespace={{ stack_name }}"
      register: services
      until: "'Running' in services.stdout"
      retries: 10
      delay: 5
```

**docker-compose.yml.j2:**
```yaml
version: '3.8'

services:
  web:
    image: "{{ registry }}/web:{{ version }}"
    ports:
      - "80:3000"
    environment:
      - NODE_ENV=production
      - DB_HOST=db
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
      healthcheck:
        test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
        interval: 30s
        timeout: 3s
        retries: 3

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_PASSWORD={{ db_password }}
    volumes:
      - db_data:/var/lib/postgresql/data
    deploy:
      placement:
        constraints: [node.role == manager]

volumes:
  db_data:
    driver: local
```

---

## Service Updates

```yaml
---
- name: Update service
  hosts: swarm_managers
  
  tasks:
    - name: Update service image
      shell: |
        docker service update \
          --image {{ registry }}/app:{{ new_version }} \
          --update-delay 10s \
          --update-parallelism 1 \
          --update-failure-action rollback \
          {{ service_name }}

    - name: Wait for update
      shell: |
        docker service ps {{ service_name }} \
          --no-trunc --filter 'desired-state=running'
      register: service_status
      until: "'Running' in service_status.stdout"
      retries: 30
      delay: 5
```

---

## Secrets Management

```yaml
---
- name: Manage secrets in Swarm
  hosts: swarm_managers
  
  tasks:
    - name: Create database secret
      community.docker.docker_secret:
        name: db_password
        data: "{{ db_password }}"
        state: present

    - name: Deploy service with secret
      shell: |
        docker service create \
          --name app \
          --secret db_password \
          --publish 3000:3000 \
          {{ registry }}/app:{{ version }}
```

---

**Следующее:** [[03-health-monitoring|Health Checks & Monitoring]]
