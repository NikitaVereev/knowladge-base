---
title: "Анатомия Ansible Playbook"
description: "Структура YAML файла: Play, Tasks, Handlers, Vars. Порядок выполнения задач."
---


Playbook — это сценарий автоматизации, описанный в формате YAML. Он говорит Ansible **что** делать (Tasks) и **где** это делать (Hosts).

## Структура Playbook
Один файл может содержать несколько сценариев (Plays).

```yaml
***
- name: Настройка веб-серверов   # <-- Play 1
  hosts: webservers
  become: yes
  tasks:
    - name: Установить Nginx     # <-- Task
      apt: name=nginx state=present

- name: Настройка БД            # <-- Play 2
  hosts: databases
  tasks:
    - name: Установить MySQL
      apt: name=mysql-server
```

## Основные компоненты

### 1. Play (Сценарий)
Связывает группу хостов (`hosts`) с задачами (`tasks`). Определяет контекст выполнения (пользователь, sudo).

### 2. Tasks (Задачи)
Список действий, выполняемых последовательно сверху вниз. Каждая задача вызывает модуль Ansible.

### 3. Handlers (Обработчики)
Специальные задачи, которые запускаются **только если** их уведомили (`notify`). Они выполняются **один раз** в самом конце Play.
*Пример:* Перезагрузка Nginx после изменения конфига.

### 4. Pre_tasks и Post_tasks
* `pre_tasks`: Выполняются до основных задач (и до ролей).
* `post_tasks`: Выполняются после основных задач.

## Порядок выполнения
1. Сбор фактов (`gather_facts`).
2. `pre_tasks`.
3. Роли (`roles`).
4. Основные `tasks`.
5. `post_tasks`.
6. `handlers` (срабатывают в конце, если были изменения).

## Связанные материалы
- [[devops/ansible/how-to/run-playbook|Запуск и отладка Playbook]]
- [[devops/ansible/reference/common-modules|Популярные модули]]
