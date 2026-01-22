---
title: "16 Деплой приложений в Kubernetes через Ansible"
description: "Использование коллекции kubernetes.core."
---

Ansible может управлять объектами K8s так же легко, как и файлами конфигурации.

**Требования:**
*   `pip install kubernetes`
*   `ansible-galaxy collection install kubernetes.core`

## Пример Playbook

```yaml
- name: Deploy to K8s
  hosts: localhost
  connection: local
  tasks:
    - name: Create Namespace
      kubernetes.core.k8s:
        name: my-app
        api_version: v1
        kind: Namespace
        state: present

    - name: Deploy Deployment
      kubernetes.core.k8s:
        state: present
        namespace: my-app
        definition:
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: nginx
          spec:
            replicas: 2
            selector:
              matchLabels:
                app: nginx
            template:
              metadata:
                labels:
                  app: nginx
              spec:
                containers:
                  - name: nginx
                    image: nginx:1.14.2
                    ports:
                      - containerPort: 80
```
