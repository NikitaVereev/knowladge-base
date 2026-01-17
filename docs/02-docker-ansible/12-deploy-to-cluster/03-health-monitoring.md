---
title: 03 Health Checks & Monitoring
---

---

## Docker Health Checks

```dockerfile
# Basic health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

---

## Health Check Endpoints

```javascript
// Node.js example
app.get('/health', (req, res) => {
  if (db.connected && cache.connected) {
    res.status(200).json({ status: 'healthy' });
  } else {
    res.status(503).json({ status: 'unhealthy' });
  }
});

app.get('/health/live', (req, res) => {
  res.status(200).json({ status: 'alive' });
});

app.get('/health/ready', (req, res) => {
  db.ping()
    .then(() => res.status(200).json({ status: 'ready' }))
    .catch(() => res.status(503).json({ status: 'not ready' }));
});
```

---

## Verify Service Health

```yaml
---
- name: Monitor service health
  hosts: swarm_managers
  
  tasks:
    - name: Get service status
      shell: "docker service ps {{ service_name }} --no-trunc"
      register: service_status

    - name: Check running tasks
      shell: |
        docker service ls --filter name={{ service_name }} \
          --format 'table {{.Replicas}}'
      register: replicas

    - name: Show logs
      shell: "docker service logs {{ service_name }} --tail 50"
      register: service_logs

    - name: Verify health
      uri:
        url: "http://localhost:3000/health"
        status_code: 200
      register: health_check
      failed_when: health_check.status != 200
```

---

## Service Status Commands

```bash
# List all services
docker service ls

# Check specific service
docker service ps myapp

# View service details
docker service inspect myapp

# Follow logs
docker service logs myapp -f

# Check task status
docker service ps myapp --no-trunc --filter desired-state=running
```

---

## Ansible Health Monitoring

```yaml
---
- name: Health check playbook
  hosts: swarm_managers
  
  tasks:
    - name: Check service exists
      shell: "docker service ls --filter name={{ service_name }}"
      register: service_exists
      failed_when: service_exists.stdout == ''

    - name: Check replicas running
      shell: |
        docker service ls --filter name={{ service_name }} \
          --format '{{.Replicas}}'
      register: replicas_count
      failed_when: '0/' in replicas_count.stdout

    - name: Test health endpoint
      uri:
        url: "http://localhost/health"
        status_code: 200
      retries: 5
      delay: 2

    - name: Alert if unhealthy
      debug:
        msg: "WARNING: Service {{ service_name }} is unhealthy!"
      when: health_check.failed
```

---

**Следующее:** [[04-multi-environment|Multi-Environment]]
