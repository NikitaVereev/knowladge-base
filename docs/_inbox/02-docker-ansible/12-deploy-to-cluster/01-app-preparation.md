---
title: 01 Application Preparation
---

---

## Dockerfile Optimization

**Multi-stage build:**
```dockerfile
# Build stage
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Production stage
FROM node:18-alpine

WORKDIR /app

RUN npm install -g dumb-init

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package*.json ./

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD node healthcheck.js

ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/server.js"]
```

---

## Image Versioning

```bash
# Semantic versioning
VERSION=1.2.3
REGISTRY=docker.io/myapp

# Build with tags
docker build -t $REGISTRY:$VERSION \
             -t $REGISTRY:latest \
             -t $REGISTRY:1.2 \
             -f Dockerfile .

# Push all tags
docker push $REGISTRY:$VERSION
docker push $REGISTRY:latest
docker push $REGISTRY:1.2
```

**In Ansible:**
```yaml
---
- name: Build and push image
  hosts: localhost
  gather_facts: no
  
  vars:
    app_name: myapp
    registry: docker.io
    version: "{{ lookup('pipe', 'git describe --tags --always') }}"

  tasks:
    - name: Build image
      docker_image:
        name: "{{ registry }}/{{ app_name }}"
        tag: "{{ version }}"
        source: build
        build:
          path: ./app
          dockerfile: Dockerfile

    - name: Push to registry
      docker_image:
        name: "{{ registry }}/{{ app_name }}"
        tag: "{{ version }}"
        push: yes
```

---

## Registry Management

**Login to registry:**
```yaml
---
- name: Login to Docker Hub
  docker_login:
    registry_url: "{{ registry_url }}"
    username: "{{ registry_user }}"
    password: "{{ registry_password }}"
    reauthorize: yes
```

**Private registry:**
```yaml
---
- name: Create registry secret in Swarm
  hosts: swarm_managers
  
  tasks:
    - name: Create registry credentials secret
      community.docker.docker_secret:
        name: registry_creds
        data: "{{ lookup('template', 'registry-creds.j2') }}"
        state: present
```

---

## Health Check Configuration

```dockerfile
# In Dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

**Health check script:**
```javascript
// healthcheck.js
const http = require('http');

const options = {
  hostname: 'localhost',
  port: 3000,
  path: '/health',
  method: 'GET',
  timeout: 2000
};

const req = http.request(options, (res) => {
  if (res.statusCode == 200) {
    process.exit(0);
  } else {
    process.exit(1);
  }
});

req.on('error', () => process.exit(1));
req.end();
```

---

## Pre-deployment Validation

```yaml
---
- name: Validate image
  hosts: localhost
  gather_facts: no
  
  tasks:
    - name: Check image size
      docker_image_info:
        name: "{{ image_name }}:{{ version }}"
      register: image_info
      failed_when: image_info.images[0].Size > 500000000

    - name: Run tests in container
      docker_container:
        name: test_container
        image: "{{ image_name }}:{{ version }}"
        command: npm test
        detach: no
        auto_remove: yes
      register: test_result
      failed_when: test_result.rc != 0
```

---

**Следующее:** [[02-swarm-deployment|Swarm Deployment]]
