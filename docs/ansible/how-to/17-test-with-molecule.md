---
title: "17 Тестирование Ansible ролей с Molecule"
description: "Автоматическое тестирование инфраструктурного кода."
---

Molecule создает временное окружение (обычно Docker контейнер), прогоняет вашу роль, проверяет результат (Unit Test) и уничтожает окружение.

## Установка и Инициализация

```bash
pip install molecule molecule-plugins[docker]
molecule init role my-role --driver-name docker
```

## Цикл тестирования (molecule test)

1.  **Dependency:** Установка зависимостей.
2.  **Lint:** Проверка стиля.
3.  **Cleanup:** Очистка старых тестов.
4.  **Destroy:** Удаление контейнера.
5.  **Create:** Создание чистого Docker контейнера.
6.  **Converge:** Запуск вашей роли (Playbook).
7.  **Idempotence:** Повторный запуск роли (проверка, что изменений нет).
8.  **Verify:** Запуск проверочных тестов (например, "порт 80 открыт").
9.  **Destroy:** Удаление контейнера.

## Пример теста (verify.yml)

```yaml
- name: Verify
  hosts: all
  tasks:
    - name: Check Nginx is running
      command: systemctl status nginx
      register: status
      failed_when: "'active (running)' not in status.stdout"
```
