---
title: "Управление секретами (Vault)"
type: how-to
tags: [ansible, vault, secrets, encryption, passwords, ci-cd]
sources:
  docs: "https://docs.ansible.com/ansible/latest/vault_guide/index.html"
related:
  - "[[ansible/explanation/variables-and-facts]]"
  - "[[ansible/how-to/write-playbooks]]"
  - "[[ansible/reference/best-practices]]"
---

# Управление секретами (Ansible Vault)

> **TL;DR:** `ansible-vault encrypt secrets.yml` — шифрование. Файл хранится в Git зашифрованным.
> `--vault-password-file` — для CI/CD. Шифровать только секреты, не весь файл.

Ansible Vault шифрует конфиденциальные данные (пароли, ключи, токены) для безопасного хранения в Git.

## Основные команды

```bash
# Создать зашифрованный файл
ansible-vault create group_vars/all/vault.yml

# Редактировать (расшифрует → откроет $EDITOR → зашифрует)
ansible-vault edit group_vars/all/vault.yml

# Зашифровать существующий файл
ansible-vault encrypt secrets.yml

# Расшифровать файл
ansible-vault decrypt secrets.yml

# Просмотреть содержимое (не расшифровывая файл на диске)
ansible-vault view secrets.yml

# Сменить пароль
ansible-vault rekey secrets.yml

# Зашифровать одну строку (для вставки в YAML)
ansible-vault encrypt_string 'SuperSecret123' --name 'db_password'
```

## Рекомендуемая структура

```
group_vars/
├── all/
│   ├── vars.yml           # обычные переменные (открытые)
│   └── vault.yml          # зашифрованные секреты
├── production/
│   ├── vars.yml
│   └── vault.yml
└── staging/
    ├── vars.yml
    └── vault.yml
```

**vars.yml** — ссылается на vault-переменные:

```yaml
# group_vars/all/vars.yml
db_host: postgres.example.com
db_user: app
db_password: "{{ vault_db_password }}"      # ← ссылка на vault
api_key: "{{ vault_api_key }}"
```

**vault.yml** — содержит зашифрованные значения:

```yaml
# group_vars/all/vault.yml (зашифрован)
vault_db_password: "SuperSecret123"
vault_api_key: "sk-abc123def456"
```

Конвенция: vault-переменные с префиксом `vault_`. Это сразу видно, откуда значение.

## Использование в Playbook

```yaml
- hosts: all
  tasks:
    - name: Configure database
      ansible.builtin.template:
        src: db.conf.j2
        dest: /etc/app/db.conf
      # db_password автоматически подставится из vault
```

## Запуск с Vault

```bash
# Интерактивно (запросит пароль)
ansible-playbook site.yml --ask-vault-pass

# Из файла (для CI/CD)
echo "MyVaultPassword" > .vault_pass
chmod 600 .vault_pass
ansible-playbook site.yml --vault-password-file .vault_pass

# Через переменную окружения
export ANSIBLE_VAULT_PASSWORD_FILE=.vault_pass
ansible-playbook site.yml

# В ansible.cfg
# [defaults]
# vault_password_file = .vault_pass
```

> **Важно:** `.vault_pass` — в `.gitignore`. Никогда не коммитить пароль от vault.

## Шифрование отдельных строк

Вместо шифрования целого файла — шифруется одна переменная внутри обычного YAML:

```bash
ansible-vault encrypt_string 'SuperSecret123' --name 'db_password'
```

Вывод (вставить в YAML):

```yaml
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  62313365396662343061393464336163...
```

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Шифрование всего файла с переменными | Не видно какие переменные есть без расшифровки | Разделять: `vars.yml` (открытый) + `vault.yml` (шифрованный) |
| `.vault_pass` в Git | Пароль утёк | Добавить в `.gitignore`, использовать `chmod 600` |
| Разные пароли для разных окружений | `--ask-vault-pass` запрашивает один пароль | `--vault-id prod@prompt --vault-id dev@.vault_pass_dev` |
| `ansible-vault edit` без $EDITOR | Открывается `vi`, непонятно как выйти | `export EDITOR=nano` или `export EDITOR="code --wait"` |