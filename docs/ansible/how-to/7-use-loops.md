---
title: "7 Использование циклов (Loops)"
description: "Как повторять задачи с помощью ключевого слова loop."
---

Вместо того чтобы писать 10 задач для установки 10 пакетов, используйте `loop`.

## Простой список

```yaml
- name: Install packages
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - git
    - curl
    - vim
    - htop
```

## Список словарей (List of Hashes)
Используется, когда для каждого элемента нужно несколько параметров (например, имя пользователя и его группа).

```yaml
- name: Create users
  user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
    state: present
  loop:
    - { name: 'alice', groups: 'admin' }
    - { name: 'bob', groups: 'dev' }
    - { name: 'charlie', groups: 'dev' }
```

## Регистры в циклах
Если вы используете `register` с циклом, переменная будет содержать ключ `results` — список результатов для каждой итерации.

```yaml
- shell: "echo {{ item }}"
  loop: [one, two]
  register: echo_results

- debug:
    msg: "{{ echo_results.results.stdout }}" # Выведет "one"
```
