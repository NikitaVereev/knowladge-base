---
title: 05 Deployment Strategies
---

---

## Rolling Deployment

```yaml
---
- name: Rolling deployment
  hosts: webservers
  serial: 1  # One server at a time
  
  tasks:
    - name: Stop application
      systemd:
        name: app
        state: stopped

    - name: Pull latest code
      git:
        repo: https://github.com/myapp.git
        dest: /app
        version: main

    - name: Install dependencies
      shell: |
        cd /app
        pip install -r requirements.txt

    - name: Run migrations
      shell: cd /app && python manage.py migrate

    - name: Start application
      systemd:
        name: app
        state: started

    - name: Wait for application
      uri:
        url: http://localhost:8080/health
        status_code: 200
      register: result
      retries: 5
      delay: 3
      until: result.status == 200

    - name: Run smoke tests
      shell: curl -f http://localhost:8080/api/status
      changed_when: false
```

---

## Blue-Green Deployment

```yaml
---
- name: Blue-green deployment
  hosts: webservers
  
  tasks:
    - name: Deploy to green environment
      git:
        repo: https://github.com/myapp.git
        dest: /app-green
        version: main

    - name: Install green dependencies
      shell: |
        cd /app-green
        pip install -r requirements.txt

    - name: Start green application
      systemd:
        name: app-green
        state: started

    - name: Health check green
      uri:
        url: http://localhost:8081/health
        status_code: 200
      retries: 10
      delay: 2

    - name: Switch traffic to green
      shell: |
        rm -f /app
        ln -s /app-green /app
        systemctl reload nginx

    - name: Stop blue application
      systemd:
        name: app-blue
        state: stopped
```

---

## Canary Deployment

```yaml
---
- name: Canary deployment
  hosts: webservers
  serial:
    - 1        # First server (canary)
    - "50%"    # Half of remaining
    - "100%"   # All remaining
  
  tasks:
    - name: Deploy application
      git:
        repo: https://github.com/myapp.git
        dest: /app
        version: "{{ deployment_version }}"

    - name: Install dependencies
      shell: cd /app && pip install -r requirements.txt

    - name: Restart application
      systemd:
        name: app
        state: restarted

    - name: Health check
      uri:
        url: http://localhost:8080/health
        status_code: 200
      retries: 5
      delay: 3
```

---

## GitOps Workflow

```yaml
---
- name: GitOps sync
  hosts: all
  gather_facts: yes
  
  tasks:
    - name: Sync from repository
      git:
        repo: "{{ git_repo }}"
        dest: /etc/ansible/
        version: main
      register: git_sync

    - name: Apply configuration if changed
      shell: |
        cd /etc/ansible
        ansible-playbook deploy.yml
      when: git_sync.changed

    - name: Log deployment
      copy:
        content: |
          Deployment at {{ ansible_date_time.iso8601 }}
          Version: {{ git_sync.after }}
        dest: /var/log/deployment.log
      when: git_sync.changed
```

---

## Deployment Validation

```yaml
---
- name: Post-deployment validation
  hosts: all
  
  tasks:
    - name: Check application status
      uri:
        url: http://localhost:8080/status
        method: GET
        status_code: 200
      register: app_status
      failed_when: app_status.status != 200

    - name: Check database connectivity
      shell: |
        psql -h {{ db_host }} -U {{ db_user }} \
          -d {{ db_name }} -c "SELECT 1"
      register: db_check
      failed_when: db_check.rc != 0

    - name: Verify critical endpoints
      uri:
        url: "http://localhost:8080{{ item }}"
        status_code: 200
      loop:
        - /api/users
        - /api/status
        - /api/health

    - name: Alert on failure
      debug:
        msg: "Deployment validation failed!"
      when: app_status.failed or db_check.failed
```

---

**Следующее:** [[06-orchestration-complete|Container Orchestration]]
