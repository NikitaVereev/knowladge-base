---
title: "6 SSH подключение и файлы"
description: "Как подключаться к ВМ через сторонние клиенты и передавать файлы."
---

## Использование внешнего SSH клиента

Иногда нужно подключиться не через `vagrant ssh`, а через Putty, IDE или Ansible.

1.  Выполните команду в папке проекта:
    ```bash
    vagrant ssh-config
    ```
2.  Вы получите параметры подключения:
    *   **User:** vagrant
    *   **Port:** (обычно 2222)
    *   **IdentityFile:** путь к приватному ключу

Пример подключения через обычный ssh:
```bash
ssh -p 2222 -i .vagrant/machines/default/virtualbox/private_key vagrant@127.0.0.1
```

## Передача файлов (SCP)

Чтобы скопировать файл с хоста внутрь ВМ:

```bash
scp -P 2222 -i path/to/private_key local-file.txt vagrant@127.0.0.1:/home/vagrant/
```

*Примечание: Проще использовать Shared Folders (синхронизированные папки), настроенные в Vagrantfile.*
