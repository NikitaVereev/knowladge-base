---
title: 05 Updates & Rollback Strategies
---

---

## Rolling Update

```yaml
---
- name: Rolling update
  hosts: swarm_managers
  
  vars:
    service_name: app
    new_version: "1.2.3"
  
  tasks:
    - name: Backup current version
      shell: |
        docker service inspect {{ service_name }} \
          --format '{{ .Spec.TaskTemplate.ContainerSpec.Image }}' \
          > /tmp/{{ service_name }}_backup.txt

    - name: Update service
      shell: |
        docker service update \
          --image {{ registry }}/{{ service_name }}:{{ new_version }} \
          --update-delay 10s \
          --update-parallelism 1 \
          --update-failure-action rollback \
          {{ service_name }}

    - name: Wait for update
      shell: |
        docker service ps {{ service_name }} \
          --filter 'desired-state=running' \
          --no-trunc
      register: service_ps
      until: "'Running' in service_ps.stdout"
      retries: 30
      delay: 5

    - name: Verify health
      uri:
        url: "http://localhost/health"
        status_code: 200
      register: health
      failed_when: health.status != 200
```

---

## Blue-Green Deployment

```yaml
---
- name: Blue-green deployment
  hosts: swarm_managers
  
  vars:
    current_color: blue
    new_color: green
  
  tasks:
    - name: Deploy green version
      community.docker.docker_swarm_service:
        name: "app-{{ new_color }}"
        image: "{{ registry }}/app:{{ new_version }}"
        replicas: 5
        publish:
          - protocol: tcp
            published_port: 8081
            target_port: 3000

    - name: Wait for green startup
      shell: |
        docker service ps app-{{ new_color }} \
          --filter 'desired-state=running'
      register: green_status
      until: "'Running' in green_status.stdout"
      retries: 20
      delay: 5

    - name: Test green
      uri:
        url: "http://localhost:8081/health"
        status_code: 200
      retries: 5
      delay: 3

    - name: Switch traffic to green
      shell: |
        docker service update \
          --publish-add 80:3000 \
          app-{{ new_color }}

    - name: Remove blue
      shell: "docker service rm app-{{ current_color }}"
      ignore_errors: yes
```

---

## Canary Deployment

```yaml
---
- name: Canary deployment
  hosts: swarm_managers
  
  tasks:
    - name: Scale to canary (1 replica)
      shell: "docker service scale app=1"

    - name: Update canary
      shell: |
        docker service update \
          --image {{ registry }}/app:{{ new_version }} \
          app

    - name: Monitor canary
      shell: "docker service ps app --filter 'desired-state=running'"
      register: canary_status
      until: "'Running' in canary_status.stdout"
      retries: 10
      delay: 5

    - name: Check canary health
      uri:
        url: "http://localhost/health"
        status_code: 200
      retries: 5
      delay: 2

    - name: Scale to full
      shell: "docker service scale app={{ full_replicas }}"
```

---

## Automatic Rollback

```yaml
---
- name: Deploy with automatic rollback
  hosts: swarm_managers
  
  block:
    - name: Update service
      shell: |
        docker service update \
          --image {{ registry }}/app:{{ new_version }} \
          app

    - name: Health check validation
      uri:
        url: "http://localhost/health"
        status_code: 200
      retries: 10
      delay: 5

    - name: Run tests
      shell: "bash /opt/tests/smoke-tests.sh"
      register: test_result
      failed_when: test_result.rc != 0

  rescue:
    - name: Rollback on failure
      shell: |
        docker service update \
          --image {{ registry }}/app:{{ current_version }} \
          app
      
    - name: Verify rollback
      uri:
        url: "http://localhost/health"
        status_code: 200
      retries: 5
```

---

**Следующее:** [[06-complete-deployment|Complete Project]]
