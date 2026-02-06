---
title: "15 Управление Docker Swarm через Ansible"
description: "Автоматизация создания кластера и деплоя сервисов."
---

## 1. Инициализация кластера (Manager)

```yaml
- name: Init Swarm Cluster
  hosts: managers
  become: yes
  tasks:
    - name: Initialize Swarm
      community.docker.docker_swarm:
        state: present
        advertise_addr: "{{ ansible_default_ipv4.address }}"
      register: swarm_info

    - name: Save tokens
      set_fact:
        worker_token: "{{ swarm_info.swarm_facts.JoinTokens.Worker }}"
```

## 2. Подключение воркеров

Используем `hostvars` чтобы получить токен, сохраненный на первом хосте.

```yaml
- name: Join Workers
  hosts: workers
  become: yes
  vars:
    manager_ip: "{{ hostvars[groups['managers']]['ansible_default_ipv4']['address'] }}"
    token: "{{ hostvars[groups['managers']]['worker_token'] }}"
  tasks:
    - name: Join Swarm
      community.docker.docker_swarm:
        state: join
        join_token: "{{ token }}"
        remote_addrs: [ "{{ manager_ip }}:2377" ]
```

## 3. Деплой сервиса

```yaml
- name: Deploy Service
  hosts: managers
  become: yes
  tasks:
    - name: Deploy Nginx
      community.docker.docker_swarm_service:
        name: my-web
        image: nginx:alpine
        replicas: 3
        publish:
          - published_port: 8080
            target_port: 80
```
