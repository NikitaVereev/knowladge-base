---
title: "12 Управление секретами с Ansible Vault"
description: "Как шифровать пароли и ключи."
---

Ansible Vault позволяет хранить чувствительные данные в зашифрованном виде прямо в Git репозитории.

## Основные команды

1.  **Создать новый зашифрованный файл:**
    ```bash
    ansible-vault create secrets.yml
    ```
    Вас попросят ввести пароль. Этот пароль будет нужен для дешифровки.

2.  **Редактировать существующий файл:**
    ```bash
    ansible-vault edit secrets.yml
    ```

3.  **Зашифровать обычный файл:**
    ```bash
    ansible-vault encrypt existing_file.yml
    ```

4.  **Расшифровать файл:**
    ```bash
    ansible-vault decrypt existing_file.yml
    ```

## Использование в Playbook

Подключите файл с секретами как обычные переменные.

```yaml
- hosts: all
  vars_files:
    - secrets.yml
  tasks:
    - name: Use secret
      debug:
        msg: "The password is {{ db_password }}"
```

## Запуск Playbook

Вам нужно передать пароль Vault при запуске.

1.  **Интерактивно:**
    ```bash
    ansible-playbook site.yml --ask-vault-pass
    ```
2.  **Из файла (для CI/CD):**
    Создайте файл `.vault_pass` с паролем внутри.
    ```bash
    ansible-playbook site.yml --vault-password-file .vault_pass
    ```
