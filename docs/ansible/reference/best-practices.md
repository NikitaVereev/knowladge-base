---
title: "Best Practices"
type: reference
tags: [ansible, best-practices, idempotent, lint, naming, security, no-log, fqcn]
sources:
  docs: "https://docs.ansible.com/ansible/latest/tips_tricks/ansible_tips_tricks.html"
related:
  - "[[ansible/reference/project-structure]]"
  - "[[ansible/how-to/create-roles]]"
  - "[[ansible/how-to/manage-secrets]]"
---

# Ansible Best Practices

> **Справочник:** Правила написания поддерживаемого, идемпотентного и безопасного кода.
> Каждый пункт — конкретный пример «плохо → хорошо».

## 1. Структура и организация

### Используйте роли

```yaml
# ✗ Плохо: site.yml на 500 строк
- hosts: all
  tasks:
    - name: Install nginx ...
    - name: Configure nginx ...
    # ... 200 задач ...

# ✓ Хорошо: site.yml вызывает роли
- hosts: webservers
  roles:
    - common
    - nginx
    - app
```

### Именование переменных с префиксом роли

Ansible имеет плоское пространство имён. Переменная `port` в одной роли перезапишет `port` в другой.

```yaml
# ✗ Плохо
port: 80
user: deploy

# ✓ Хорошо
nginx_port: 80
app_deploy_user: deploy
postgres_port: 5432
```

### FQCN для модулей

```yaml
# ✗ Устаревший стиль
- apt: name=nginx

# ✓ Рекомендуется (с Ansible 2.10+)
- ansible.builtin.apt:
    name: nginx
    state: present
```

### Всегда указывайте `name`

```yaml
# ✗ Плохо — нечитаемые логи
- apt: name=nginx state=present

# ✓ Хорошо
- name: Install Nginx
  ansible.builtin.apt:
    name: nginx
    state: present
```

## 2. Идемпотентность

Playbook должен быть безопасен для повторного запуска.

### Shell/command → модули

```yaml
# ✗ Плохо: НЕ идемпотентно (каждый запуск добавляет строку)
- shell: echo "export FOO=bar" >> ~/.bashrc

# ✓ Хорошо: идемпотентно
- ansible.builtin.lineinfile:
    path: ~/.bashrc
    line: "export FOO=bar"
    state: present

# ✗ Плохо: shell вместо модуля
- shell: git clone https://github.com/org/app.git /opt/app

# ✓ Хорошо: модуль идемпотентен
- ansible.builtin.git:
    repo: https://github.com/org/app.git
    dest: /opt/app
    version: main
```

### Если shell неизбежен — используйте guards

```yaml
# ✓ creates — не выполнять если файл уже есть
- ansible.builtin.command: /opt/app/install.sh
  args:
    creates: /opt/app/.installed

# ✓ changed_when — для read-only команд
- ansible.builtin.command: python3 --version
  register: py_version
  changed_when: false
```

## 3. Безопасность

### Секреты — только через Vault

```yaml
# ✗ Плохо: пароль в открытом виде
vars:
  db_password: "SuperSecret123"

# ✓ Хорошо: ссылка на vault
vars:
  db_password: "{{ vault_db_password }}"
```

### no_log для чувствительных задач

```yaml
- name: Set database password
  ansible.builtin.command: >
    psql -c "ALTER USER app PASSWORD '{{ db_password }}'"
  no_log: true              # пароль не попадёт в лог
```

### Host key checking в production

```ini
# ansible.cfg

# ✗ Удобно для dev, ОПАСНО для prod
host_key_checking = False

# ✓ В production: управлять known_hosts явно
# ssh-keyscan host >> ~/.ssh/known_hosts
```

## 4. Производительность

### Отключить gather_facts если не нужны

```yaml
- hosts: webservers
  gather_facts: no          # экономит ~2 сек на хост
  tasks:
    - name: Restart app
      ansible.builtin.systemd:
        name: myapp
        state: restarted
```

### Pipelining

```ini
# ansible.cfg — ускорение в 3-5 раз
[ssh_connection]
pipelining = True
```

### Forks

```ini
# ansible.cfg — параллельные подключения (по умолчанию 5)
[defaults]
forks = 20
```

### Передавать список в модуль вместо loop

```yaml
# ✗ Медленно: N SSH-операций
- ansible.builtin.apt:
    name: "{{ item }}"
    state: present
  loop:
    - git
    - curl
    - vim

# ✓ Быстро: 1 SSH-операция
- ansible.builtin.apt:
    name:
      - git
      - curl
      - vim
    state: present
```

## 5. Отладка и CI

### Линтер

```bash
pip install ansible-lint
ansible-lint playbooks/ roles/
```

### Тестирование ролей

```bash
pip install molecule molecule-plugins[docker]
molecule test
```

### Проверка перед деплоем

```bash
ansible-playbook site.yml --syntax-check    # синтаксис
ansible-playbook site.yml --check --diff    # dry run + diff
ansible-playbook site.yml --list-tasks      # список задач
```

## Чек-лист

- [ ] Каждая задача имеет `name`
- [ ] Модули вместо shell/command где возможно
- [ ] FQCN для модулей (`ansible.builtin.apt` вместо `apt`)
- [ ] Переменные с префиксом роли (`nginx_port`, не `port`)
- [ ] Секреты в Vault, `no_log: true` для чувствительных задач
- [ ] `changed_when: false` для read-only команд
- [ ] `pipelining = True` в ansible.cfg
- [ ] `ansible-lint` в CI pipeline
- [ ] Molecule-тесты для ролей
- [ ] `--check --diff` перед production-деплоем
