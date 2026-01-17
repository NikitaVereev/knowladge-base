---
title: 06 Container Orchestration & Complete Project
---

---

## Docker Swarm with Ansible

```yaml
---
- name: Initialize Docker Swarm
  hosts: docker_managers
  
  tasks:
    - name: Init swarm on first manager
      shell: docker swarm init --advertise-addr {{ ansible_default_ipv4.address }}
      when: inventory_hostname == groups['docker_managers'][0]
      register: swarm_init

    - name: Get worker join token
      shell: docker swarm join-token -q worker
      when: inventory_hostname == groups['docker_managers'][0]
      register: worker_token

    - name: Get manager join token
      shell: docker swarm join-token -q manager
      when: inventory_hostname == groups['docker_managers'][0]
      register: manager_token

- name: Join workers to swarm
  hosts: docker_workers
  
  tasks:
    - name: Join worker
      shell: "docker swarm join --token {{ hostvars[groups['docker_managers'][0]]['worker_token'].stdout }} {{ hostvars[groups['docker_managers'][0]]['ansible_default_ipv4']['address'] }}:2377"

- name: Deploy service to swarm
  hosts: docker_managers[0]
  
  tasks:
    - name: Create overlay network
      shell: docker network create --driver overlay app_network

    - name: Deploy service
      shell: |
        docker service create \
          --name web \
          --replicas 3 \
          --publish 8080:80 \
          --network app_network \
          nginx
```

---

## Kubernetes Deployment

```yaml
---
- name: Deploy to Kubernetes
  hosts: localhost
  gather_facts: no
  
  tasks:
    - name: Create namespace
      kubernetes.core.k8s:
        name: myapp
        api_version: v1
        kind: Namespace
        state: present

    - name: Deploy application
      kubernetes.core.k8s:
        state: present
        namespace: myapp
        definition:
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: myapp
          spec:
            replicas: 3
            selector:
              matchLabels:
                app: myapp
            template:
              metadata:
                labels:
                  app: myapp
              spec:
                containers:
                  - name: app
                    image: myapp:latest
                    ports:
                      - containerPort: 8080

    - name: Expose service
      kubernetes.core.k8s:
        state: present
        namespace: myapp
        definition:
          apiVersion: v1
          kind: Service
          metadata:
            name: myapp
          spec:
            selector:
              app: myapp
            ports:
              - protocol: TCP
                port: 80
                targetPort: 8080
            type: LoadBalancer

    - name: Wait for rollout
      kubernetes.core.k8s_info:
        kind: Deployment
        name: myapp
        namespace: myapp
        wait: yes
        wait_condition:
          type: Progressing
          status: "True"
```

---

## Complete Production Project

**Project structure:**
```
ansible-prod/
├── inventory/
│   ├── production.yml
│   ├── staging.yml
│   └── group_vars/
│       ├── webservers.yml
│       ├── databases.yml
│       └── all.yml
├── roles/
│   ├── common/
│   ├── nginx/
│   ├── postgresql/
│   ├── app/
│   └── monitoring/
├── templates/
├── files/
├── site.yml
├── secrets.yml (vault encrypted)
└── requirements.yml
```

**site.yml:**
```yaml
---
- name: Deploy infrastructure
  hosts: all
  gather_facts: yes

- name: Configure base
  import_playbook: playbooks/base.yml

- name: Configure webservers
  import_playbook: playbooks/webservers.yml
  when: '"webservers" in group_names'

- name: Configure databases
  import_playbook: playbooks/databases.yml
  when: '"databases" in group_names'

- name: Deploy application
  import_playbook: playbooks/application.yml
  when: '"app_servers" in group_names'

- name: Validate deployment
  import_playbook: playbooks/validation.yml
```

---

## Deployment Commands

```bash
# Validate syntax
ansible-lint site.yml

# Check inventory
ansible-inventory -i inventory/production.yml --list

# Dry run
ansible-playbook -i inventory/production.yml site.yml --check

# Deploy with vault
ansible-playbook -i inventory/production.yml site.yml --ask-vault-pass

# Deploy specific tags
ansible-playbook -i inventory/production.yml site.yml --tags deploy

# Check playbook syntax
ansible-playbook -i inventory/production.yml site.yml --syntax-check

# Show variables
ansible-playbook -i inventory/production.yml site.yml -e "ansible_verbosity=3"
```

---

## Monitoring & Validation

```yaml
---
- name: Post-deployment monitoring
  hosts: all
  tasks:
    - name: Service status
      systemd:
        name: "{{ item }}"
        state: started
      loop:
        - nginx
        - app
        - postgresql

    - name: Check connectivity
      wait_for:
        host: "{{ ansible_host }}"
        port: "{{ item }}"
        delay: 10
      loop: [22, 80, 443]

    - name: Collect metrics
      shell: |
        echo "CPU: $(uptime)"
        echo "Disk: $(df -h /)"
        echo "Memory: $(free -h)"
      register: metrics

    - name: Report status
      debug:
        msg: "{{ metrics.stdout }}"
```

---
