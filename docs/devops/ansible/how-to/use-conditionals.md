---
title: "Условия и логика в Ansible"
description: "Как использовать when, register, failed_when и jinja2 фильтры."
---


Ansible позволяет делать плейбуки адаптивными, выполняя задачи только при определенных условиях.

## Оператор `when`
Аналог `if` в программировании.

```yaml
- name: Install Apache (только для Debian)
  apt: name=apache2 state=present
  when: ansible_os_family == "Debian"

- name: Reboot if required
  reboot:
  when: reboot_required_file.stat.exists and force_reboot | bool
```

## Сохранение результата (`register`)
Часто нужно проверить результат выполнения команды.

```yaml
- name: Check if config exists
  stat: path=/etc/app/config.ini
  register: config_file

- name: Create default config
  copy: src=default.ini dest=/etc/app/config.ini
  when: not config_file.stat.exists
```

## Условия завершения (`failed_when`, `changed_when`)
Можно переопределить стандартное поведение Ansible.

```yaml
- name: Run script ignoring code 2
  command: /opt/script.sh
  register: result
  failed_when: result.rc != 0 and result.rc != 2
  changed_when: "'UPDATED' in result.stdout"
```

## Jinja2 Фильтры
Для преобразования данных внутри условий и шаблонов.

* `default(value)`: Использовать значение по умолчанию.
* `bool`: Преобразовать строку "true"/"yes" в булево.
* `int`: Преобразовать в число.

```yaml
- debug:
    msg: "Port is {{ app_port | default(8080) }}"
```

## Связанные материалы
- [[devops/ansible/explanation/variables-and-facts|Переменные и Facts]]
