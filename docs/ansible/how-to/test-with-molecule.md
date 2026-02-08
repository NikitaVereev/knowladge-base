---
title: "Тестирование ролей (Molecule)"
type: how-to
tags: [ansible, molecule, testing, docker, idempotence, verify, ci-cd, lint]
sources:
  docs: "https://ansible.readthedocs.io/projects/molecule/"
related:
  - "[[ansible/how-to/create-roles]]"
  - "[[ansible/how-to/debug-playbooks]]"
  - "[[ansible/reference/best-practices]]"
---

# Тестирование ролей с Molecule

> **TL;DR:** Molecule создаёт Docker-контейнер → прогоняет роль → проверяет идемпотентность
> → запускает verify-тесты → уничтожает контейнер. `molecule test` — полный цикл.

Molecule автоматизирует тестирование Ansible-ролей: создаёт временное окружение, применяет роль, проверяет результат и удаляет окружение.

## Установка

```bash
pip install molecule molecule-plugins[docker]

# Или через pipx
pipx install molecule
pipx inject molecule molecule-plugins[docker]
```

## Инициализация

```bash
# Создать новую роль с Molecule
molecule init role my_role --driver-name docker

# Добавить Molecule к существующей роли
cd roles/nginx
molecule init scenario --driver-name docker
```

Структура:

```
roles/nginx/
├── molecule/
│   └── default/                # сценарий (может быть несколько)
│       ├── molecule.yml        # конфигурация
│       ├── converge.yml        # playbook для применения роли
│       └── verify.yml          # тесты
├── tasks/
├── defaults/
└── ...
```

## Конфигурация (molecule.yml)

```yaml
---
dependency:
  name: galaxy

driver:
  name: docker

platforms:
  - name: ubuntu-test
    image: geerlingguy/docker-ubuntu2204-ansible
    pre_build_image: true
    privileged: true                  # для systemd
    command: /lib/systemd/systemd
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw

  - name: debian-test
    image: geerlingguy/docker-debian12-ansible
    pre_build_image: true
    privileged: true
    command: /lib/systemd/systemd

provisioner:
  name: ansible
  playbooks:
    converge: converge.yml
    verify: verify.yml

verifier:
  name: ansible
```

## Playbook для тестов

**converge.yml** — применяет роль:

```yaml
---
- name: Converge
  hosts: all
  become: yes
  roles:
    - role: nginx
      vars:
        nginx_port: 8080
```

**verify.yml** — проверяет результат:

```yaml
---
- name: Verify
  hosts: all
  become: yes
  tasks:
    - name: Check Nginx is installed
      ansible.builtin.package_facts:

    - name: Assert Nginx present
      ansible.builtin.assert:
        that: "'nginx' in ansible_facts.packages"

    - name: Check Nginx is running
      ansible.builtin.command: systemctl status nginx
      register: status
      changed_when: false
      failed_when: "'active (running)' not in status.stdout"

    - name: Check port is listening
      ansible.builtin.wait_for:
        port: 8080
        timeout: 5

    - name: Check HTTP response
      ansible.builtin.uri:
        url: http://localhost:8080
        status_code: 200
```

## Команды Molecule

```bash
# Полный цикл тестирования
molecule test

# Отдельные шаги
molecule create        # создать контейнер
molecule converge      # применить роль
molecule idempotence   # повторный запуск (проверка идемпотентности)
molecule verify        # запустить тесты
molecule login         # зайти в контейнер (для отладки)
molecule destroy       # удалить контейнер

# Быстрая итерация при разработке
molecule converge      # применить
molecule login         # проверить вручную
molecule converge      # поправить и перезапустить
molecule verify        # когда готово, прогнать тесты
molecule test          # финальный полный цикл
```

## Цикл `molecule test`

1. **dependency** — установка Galaxy-зависимостей
2. **cleanup** — очистка предыдущих запусков
3. **destroy** — удаление контейнера
4. **create** — создание чистого контейнера
5. **converge** — запуск роли (первый раз)
6. **idempotence** — повторный запуск (не должно быть `changed`)
7. **verify** — проверочные тесты
8. **cleanup** — финальная очистка
9. **destroy** — удаление контейнера

## Molecule в CI/CD (GitHub Actions)

```yaml
# .github/workflows/molecule.yml
name: Molecule Test
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install molecule molecule-plugins[docker] ansible

      - name: Run Molecule
        run: molecule test
        working-directory: roles/nginx
```

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Тесты не проверяют идемпотентность | Роль ломает повторный запуск | `molecule idempotence` — ни одна задача не должна быть `changed` |
| systemd в Docker без privileged | `Failed to connect to bus: No such file or directory` | `privileged: true` + `command: /lib/systemd/systemd` |
| Тестируют только на одном дистрибутиве | Роль ломается на Debian/RHEL | Добавить несколько `platforms` в molecule.yml |
| verify.yml проверяет только «пакет установлен» | Не проверяет конфигурацию и работоспособность | Проверять: сервис running, порт listening, HTTP 200 |