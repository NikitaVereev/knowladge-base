---
created: 2026-01-05
updated: 2026-01-05
type: reference
---

# Пользователи, группы, права доступа

## Основы

**Пользователь** — аккаунт, может логиниться и запускать процессы.

**Группа** — набор пользователей, для управления правами.

**UID (User ID)** — числовой идентификатор пользователя.

**GID (Group ID)** — числовой идентификатор группы.

**root** — суперпользователь, UID 0, может всё.

---

## Файлы пользователей и групп

### /etc/passwd

Информация о пользователях:
```
username:x:uid:gid:comment:home:shell
root:x:0:0:root:/root:/bin/bash
user:x:1000:1000:User Name:/home/user:/bin/bash
```

**Просмотр:**
```bash
cat /etc/passwd              # все пользователи
getent passwd username       # информация о конкретном
getent passwd | grep bash    # все с bash
```

### /etc/shadow

Пароли (только для root):
```bash
sudo cat /etc/shadow         # зашифрованные пароли
```

### /etc/group

Информация о группах:
```
groupname:x:gid:members
sudo:x:27:user1,user2
```

**Просмотр:**
```bash
cat /etc/group               # все группы
getent group groupname       # информация о группе
```

---

## Команды пользователей

### Текущий пользователь

```bash
whoami                       # текущий пользователь
id                           # UID, GID, группы
id username                  # информация о пользователе
groups                       # группы текущего пользователя
groups username              # группы пользователя
```

### Создание/удаление пользователя

```bash
sudo useradd username        # создать пользователя
sudo useradd -m username     # создать + домашняя папка
sudo useradd -s /bin/bash username  # с shell
sudo usermod -aG group username     # добавить в группу
sudo userdel username        # удалить пользователя
sudo userdel -r username     # удалить + домашняя папка
```

### Пароли

```bash
passwd                       # изменить пароль текущего
sudo passwd username         # изменить пароль пользователя
sudo passwd -l username      # заблокировать пароль
sudo passwd -u username      # разблокировать пароль
```

---

## Команды групп

### Просмотр групп

```bash
groups                       # мои группы
groups username              # группы пользователя
getent group                 # все группы
```

### Создание/удаление групп

```bash
sudo groupadd groupname      # создать группу
sudo groupmod -n newname oldname  # переименовать
sudo gpasswd -a user group   # добавить пользователя в группу
sudo gpasswd -d user group   # удалить пользователя из группы
sudo groupdel groupname      # удалить группу
```

---

## Права доступа (chmod)

**Формат:** `rwx` для owner, group, others.

```bash
r (read)     = 4
w (write)    = 2
x (execute)  = 1
```

**Примеры:**
```
rwx------  (700)  только owner может всё
rw-r--r--  (644)  owner может писать, остальные читают
rwxr-xr-x  (755)  owner может всё, остальные читают + выполняют
```

### Команды

```bash
chmod 644 file               # rw-r--r-- (типичное для файлов)
chmod 755 file               # rwxr-xr-x (типичное для программ)
chmod u+x file               # добавить execute для owner
chmod g+w file               # добавить write для группы
chmod o-r file               # убрать read для others
chmod -R 755 dir             # рекурсивно для папки

# Текстовый формат
chmod go-rwx file            # убрать все права для group и others
chmod a+r file               # добавить read для всех
chmod u=rwx,g=rx,o= file     # явно установить права
```

### Специальные права

```bash
setuid (4000):   SUID бит - выполнять как owner файла
setgid (2000):   SGID бит - выполнять как group файла
sticky (1000):   sticky бит - может удалить только owner

# Примеры
chmod 4755 file              # SUID + rwxr-xr-x
chmod 2755 file              # SGID + rwxr-xr-x
chmod 1777 /tmp              # sticky бит для /tmp
```

---

## Владелец файла (chown)

```bash
ls -l file                   # показать владельца
sudo chown user file         # изменить владельца
sudo chown user:group file   # изменить владельца и группу
sudo chown :group file       # изменить только группу
sudo chown -R user:group dir # рекурсивно для папки
```

---

## sudo

**sudo** — выполнить команду как root (или другой пользователь).

```bash
sudo command                 # выполнить как root
sudo -u username command     # выполнить как другой пользователь
sudo -l                      # какие команды можно без пароля
sudo -k                      # забыть пароль (потребуется при следующей команде)
sudo !!                      # повторить последнюю команду с sudo
```

### Конфигурация sudo

```bash
sudo visudo                  # безопасное редактирование sudoers
# Добавить строку:
user1 ALL=(ALL) NOPASSWD: /usr/bin/apt
# Позволяет user1 запускать apt без пароля
```

---

## su

**su** — переключиться на другого пользователя.

```bash
su                           # переключиться на root
su - root                    # с загрузкой profile
su - username                # переключиться на пользователя
exit                         # выйти
```

---

## Таблица прав

| Право | Файл | Папка |
|-------|------|-------|
| **r** (4) | читать содержимое | просмотреть список файлов |
| **w** (2) | писать в файл | создавать/удалять файлы |
| **x** (1) | выполнять программу | входить в папку |

---

## Проблемы и решения

### Нет прав на файл

```bash
ls -l file                   # проверить права
# Если вы владелец:
chmod u+r file               # добавить себе права
# Если нужен root:
sudo cat file
```

### Хочу выполнить без sudo

```bash
# Вариант 1: добавить в группу
sudo usermod -aG group $USER
newgrp group

# Вариант 2: изменить права
sudo chmod 755 /path/to/dir
```

### Забыл пароль пользователя

```bash
sudo passwd username         # установить новый пароль
```

### Пользователь не может создавать файлы в папке

```bash
# Проверить права на папку
ls -ld /path/to/dir

# Если нужна write права для группы
sudo chmod g+w /path/to/dir
```

### Как посмотреть кто логинился

```bash
last                         # история логинов
lastlog                      # последний логин каждого пользователя
who                          # кто сейчас логинен
w                            # кто логинен + чем занимается
```

---

## Шпаргалка

```bash
# Просмотр
whoami                       # я кто
id                           # мой UID/GID/группы
groups                       # мои группы

# Создание
sudo useradd -m username     # новый пользователь
sudo groupadd groupname      # новая группа

# Модификация
sudo usermod -aG group user  # добавить в группу
sudo chown user:group file   # изменить владельца
chmod 755 file               # установить права

# Проверка прав
stat file                    # детали файла
ls -l file                   # права в формате
```

---

## Дальше

[Процессы и сервисы](./03-processes-and-services.md)
