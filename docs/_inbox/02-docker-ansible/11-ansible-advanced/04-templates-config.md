---
title: 04 Templates & Configuration Management
---

---

## Jinja2 Templates

```jinja2
{# templates/nginx.conf.j2 #}
# {{ ansible_managed }}

server {
    listen {{ nginx_port }};
    server_name {{ domain }};

    {% if ssl_enabled %}
    ssl_certificate {{ ssl_cert_path }};
    ssl_certificate_key {{ ssl_key_path }};
    ssl_protocols TLSv1.2 TLSv1.3;
    {% endif %}

    location / {
        proxy_pass http://{{ app_host }}:{{ app_port }};
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        
        {% for header_name, header_value in proxy_headers.items() %}
        proxy_set_header {{ header_name }} {{ header_value }};
        {% endfor %}
    }

    {% if gzip_enabled %}
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
    {% endif %}

    {% if cache_enabled %}
    location ~* \.(js|css|png|jpg|jpeg)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
    {% endif %}
}
```

---

## Using Templates in Playbooks

```yaml
---
- name: Deploy configuration
  hosts: webservers
  
  vars:
    nginx_port: 80
    domain: example.com
    ssl_enabled: yes
    ssl_cert_path: /etc/ssl/certs/cert.pem
    ssl_key_path: /etc/ssl/private/key.pem
    app_host: localhost
    app_port: 3000
    gzip_enabled: yes
    cache_enabled: yes
    proxy_headers:
      X-Frame-Options: "SAMEORIGIN"
      X-Content-Type-Options: "nosniff"
      X-XSS-Protection: "1; mode=block"

  tasks:
    - name: Configure Nginx
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: '0644'
        backup: yes
      notify: restart nginx

  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
```

---

## Template Inheritance

**Base template (templates/base.j2):**
```jinja2
---
{% block metadata %}
name: {{ app_name }}
version: {{ app_version }}
{% endblock %}

{% block configuration %}
# Default configuration
{% endblock %}

{% block security %}
# Security settings
{% endblock %}
```

**Child template (templates/app.yml.j2):**
```jinja2
{% extends 'base.j2' %}

{% block configuration %}
server:
  host: {{ server_host }}
  port: {{ server_port }}
  workers: {{ worker_processes }}
{% endblock %}

{% block security %}
auth:
  enabled: {{ auth_enabled }}
  method: {{ auth_method }}
{% endblock %}
```

---

## Filters in Templates

```jinja2
{# String operations #}
{{ hostname | upper }}
{{ hostname | lower }}
{{ 'hello world' | title }}
{{ path | dirname }}

{# List operations #}
{{ items | join(',') }}
{{ items | unique | list }}
{{ items | reverse | list }}

{# JSON/YAML #}
{{ data | to_nice_json }}
{{ data | to_yaml }}

{# Math #}
{{ value | abs }}
{{ 2 | pow(3) }}
```

---

## Including Templates

```jinja2
{# Main template #}
{% include 'header.j2' %}

<main>
  {% for item in items %}
    {% include 'item.j2' %}
  {% endfor %}
</main>

{% include 'footer.j2' %}
```

---

**Следующее:** [[05-deployment-strategies|Deployment Strategies]]
