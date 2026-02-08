---
title: "Roles и Collections"
type: explanation
tags: [ansible, roles, collections, galaxy, ansible-galaxy, modularity, reuse]
sources:
  docs: "https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html"
related:
  - "[[ansible/explanation/playbook-anatomy]]"
  - "[[ansible/how-to/create-roles]]"
  - "[[ansible/reference/project-structure]]"
---

# Ansible Roles и Collections

> **TL;DR:** Role — переиспользуемый пакет задач для одной цели (например, «установить PostgreSQL»).
> Collection — формат распространения ролей, модулей и плагинов через Galaxy.
> С Ansible 2.10 «батарейки» вынесены из ядра в коллекции.

По мере роста Playbook хранить всё в одном файле становится неудобно. Ansible предлагает два уровня абстракции: **Roles** и **Collections**.

## Roles (Роли)

Роль — способ автоматической загрузки переменных, задач, обработчиков и шаблонов, основанный на известной файловой структуре. Роль решает одну конкретную задачу.

### Стандартная структура роли

```
roles/nginx/
├── tasks/
│   └── main.yml         # основной список задач
├── handlers/
│   └── main.yml         # обработчики (restart, reload)
├── defaults/
│   └── main.yml         # переменные по умолчанию (НИЗКИЙ приоритет — легко переопределить)
├── vars/
│   └── main.yml         # константы роли (ВЫСОКИЙ приоритет)
├── files/
│   └── nginx.key        # статические файлы (для модуля copy)
├── templates/
│   └── nginx.conf.j2    # Jinja2-шаблоны (для модуля template)
├── meta/
│   └── main.yml         # зависимости от других ролей
└── README.md
```

### Подключение ролей в Playbook

```yaml
# Вариант 1: секция roles (классический)
- hosts: webservers
  roles:
    - nginx                         # просто имя
    - role: app                     # с параметрами
      vars:
        app_port: 8080

# Вариант 2: include_role (динамический, внутри tasks)
- hosts: webservers
  tasks:
    - name: Setup nginx
      include_role:
        name: nginx
      when: install_nginx | default(true)

# Вариант 3: import_role (статический, обрабатывается при парсинге)
- hosts: webservers
  tasks:
    - import_role:
        name: nginx
```

### import vs include

| Свойство | `import_role` / `import_tasks` | `include_role` / `include_tasks` |
|----------|-------------------------------|----------------------------------|
| Обработка | При парсинге (статически) | При выполнении (динамически) |
| `when` | Применяется к каждой задаче внутри | Применяется к самому include |
| `tags` | Наследуются задачами внутри | Не наследуются |
| Loops | Не поддерживает `loop` | Поддерживает `loop` |
| Когда | Фиксированная структура | Условная логика, циклы |

### Зависимости ролей

```yaml
# roles/app/meta/main.yml
dependencies:
  - role: nginx
    vars:
      nginx_port: 8080
  - role: postgres
```

Зависимости устанавливаются автоматически перед задачами роли.

## Collections (Коллекции)

Коллекция — формат упаковки и распространения контента Ansible. Содержит:
- Роли
- Модули (Python-код)
- Плагины (filter, lookup, connection)

С версии 2.10 «батарейки» (модули) вынесены из ядра Ansible в коллекции:

| Коллекция | Содержание |
|-----------|-----------|
| `ansible.builtin` | Встроенные модули (copy, file, shell, template) |
| `community.docker` | Модули для Docker |
| `community.postgresql` | Модули для PostgreSQL |
| `amazon.aws` | Модули для AWS |
| `community.general` | «Всё остальное» |

### Установка и использование

```bash
# Установить коллекцию
ansible-galaxy collection install community.docker

# Установить из requirements.yml
ansible-galaxy install -r requirements.yml
```

```yaml
# requirements.yml
collections:
  - name: community.docker
    version: ">=3.0.0"
  - name: community.postgresql

roles:
  - name: geerlingguy.docker
    version: "7.1.0"
```

```yaml
# Использование FQCN (Fully Qualified Collection Name)
- name: Start container
  community.docker.docker_container:
    name: app
    image: myapp:latest
```

## Подводные камни

| Заблуждение | Реальность |
|------------|-----------|
| «Каждая директория в роли обязательна» | Создавай только нужные (`tasks/` минимум). Пустые директории не нужны |
| «defaults и vars — одно и то же» | `defaults/` — низкий приоритет (для настраиваемых параметров). `vars/` — высокий (для констант) |
| «Galaxy-роли production-ready» | Всегда проверяй код перед использованием. Многие роли устарели или небезопасны |
| «import и include взаимозаменяемы» | import — статический (при парсинге), include — динамический (при выполнении). Разное поведение tags/when |