---
title: "Лучшие практики Ansible"
description: "Линтинг кода, CI/CD, идемпотентность и советы по написанию чистого кода."
---


Чтобы ваши плейбуки были надежными и поддерживаемыми, следуйте этим правилам.

## 1. Качество кода (Linting)
Используйте `ansible-lint` для автоматической проверки стиля.

```bash
pip install ansible-lint
ansible-lint docker_setup.yml
```

**Частые ошибки:**
* Использование `shell` вместо специализированных модулей (например, `git` или `file`).
* Отсутствие `name` у задач.
* Использование команд без `changed_when` (нарушает идемпотентность).

## 2. Идемпотентность
Ваш скрипт должен давать одинаковый результат при повторных запусках, не ломая систему.

**Плохо:**
```yaml
- shell: "echo 'export VAR=1' >> ~/.bashrc"
```
*(При каждом запуске будет добавлять новую строку)*

**Хорошо:**
```yaml
- lineinfile:
    path: ~/.bashrc
    line: "export VAR=1"
    state: present
```

## 3. Обработка изменений
Если команда не меняет состояние системы (например, просто чтение версии), сообщайте об этом Ansible.

```yaml
- name: Check Docker version
  command: docker --version
  register: docker_version
  changed_when: false  # <-- Важно!
```

## 4. CI/CD Интеграция
Пример шага для GitHub Actions:

```yaml
- name: Run Ansible Playbook
  run: |
    ansible-playbook deploy.yml \
      -i inventory.ini \
      --private-key ${{ secrets.SSH_KEY }}
```

## Связанные материалы
- [[devops/ansible/explanation/error-handling|Обработка ошибок]]
