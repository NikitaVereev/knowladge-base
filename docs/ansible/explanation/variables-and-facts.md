---
title: "Переменные и Facts"
type: explanation
tags: [ansible, variables, facts, precedence, register, hostvars, groupvars, extra-vars]
sources:
  docs: "https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html"
related:
  - "[[ansible/explanation/inventory]]"
  - "[[ansible/explanation/playbook-anatomy]]"
  - "[[ansible/how-to/write-playbooks]]"
  - "[[ansible/reference/jinja2-filters]]"
---

# Переменные и Facts в Ansible

> **TL;DR:** Переменные определяются в 22 местах с чётким приоритетом.
> Extra vars (`-e`) всегда побеждают. Facts — автоматически собранная информация
> о хосте (OS, IP, RAM). Register — результат выполнения задачи.

Ansible использует переменные для управления различиями между системами. Они могут быть определены в десятках мест, поэтому важно понимать порядок их применения.

## Типы переменных

### 1. Пользовательские переменные

Задаются в Playbook, Inventory или отдельных файлах (`group_vars`, `host_vars`).

```yaml
vars:
  app_version: "1.2.0"
  db_port: 5432
  packages:
    - nginx
    - postgresql
    - redis
```

### 2. Facts (Факты)

Информация, которую Ansible собирает о хосте перед выполнением задач (модуль `setup`).

```yaml
# Часто используемые факты
ansible_distribution: "Ubuntu"
ansible_distribution_version: "22.04"
ansible_os_family: "Debian"
ansible_memtotal_mb: 4096
ansible_processor_vcpus: 2
ansible_default_ipv4.address: "10.0.1.5"
ansible_hostname: "web1"
ansible_env.HOME: "/root"
```

Посмотреть все факты хоста:

```bash
ansible web1 -m setup
ansible web1 -m setup -a 'filter=ansible_distribution*'
```

Отключить сбор фактов (ускоряет выполнение на ~2 сек/хост):

```yaml
- hosts: all
  gather_facts: no
```

### 3. Register (Регистры)

Переменные, создаваемые из результата выполнения задачи.

```yaml
- name: Проверить версию Python
  command: python3 --version
  register: python_version
  changed_when: false            # команда не меняет состояние

- name: Показать результат
  debug:
    msg: "Python: {{ python_version.stdout }}"

# Структура register-переменной:
# python_version.stdout        — стандартный вывод
# python_version.stderr        — ошибки
# python_version.rc            — exit code
# python_version.changed       — были ли изменения
# python_version.failed        — была ли ошибка
```

### 4. Специальные переменные

```yaml
# Встроенные переменные (Magic Variables)
inventory_hostname              # имя хоста из inventory
inventory_hostname_short        # короткое имя (до первой точки)
groups['webservers']            # список хостов в группе
hostvars['web1']                # переменные другого хоста
ansible_play_hosts              # хосты текущего Play
play_hosts                      # алиас для ansible_play_hosts
```

## Приоритет переменных (Variable Precedence)

От наименьшего к наибольшему (последний побеждает):

```
 1. command line values (not variables)
 2. role defaults (defaults/main.yml)         ← самый низкий
 3. inventory file group_vars
 4. inventory group_vars/all
 5. inventory group_vars/<group>
 6. inventory file host_vars
 7. inventory host_vars/<host>
 8. playbook group_vars/all
 9. playbook group_vars/<group>
10. playbook host_vars/<host>
11. host facts / cached set_facts
12. play vars
13. play vars_prompt
14. play vars_files
15. role vars (vars/main.yml)
16. block vars
17. task vars
18. include_vars
19. set_facts / registered vars
20. role params (roles: - role: x, var: val)
21. include params
22. extra vars (-e)                           ← ВСЕГДА побеждает
```

**Практическое правило:** Используй `defaults/main.yml` в ролях (легко переопределить) и `vars/main.yml` только для констант (сложно переопределить).

## Подводные камни

| Заблуждение | Реальность |
|------------|-----------|
| «`-e` переменные можно переопределить» | Extra vars (`-e`) имеют наивысший приоритет — их невозможно перебить |
| «Facts бесплатны» | Gather facts занимает ~2 сек на хост. При 100 хостах — 3+ минуты. Отключай если не нужны |
| «`register` доступен везде» | Register-переменная привязана к хосту. В другом Play для другого хоста она недоступна |
| «Переменные в `vars/main.yml` роли легко переопределить» | `vars/main.yml` имеет высокий приоритет (15). Используй `defaults/main.yml` (приоритет 2) для настраиваемых параметров |
