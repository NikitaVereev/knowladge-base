# Users, Groups, and Permissions

## Overview

Управление пользователями, группами и правами доступа - это основа безопасности Linux. Правильные права гарантируют что только нужные люди имеют доступ к файлам.

**Что вы узнаете:**
- Файлы пользователей и групп (/etc/passwd, /etc/group)
- Создание и удаление пользователей и групп
- Права доступа (chmod) - числовой и текстовый формат
- Специальные права (setuid, setgid, sticky bit)
- Изменение владельца файла (chown)
- sudo и su - выполнение команд от других пользователей
- Решение типичных проблем с доступом

## Prerequisites

Перед этим разделом нужно:
- Понимание файловой системы ([[01-filesystem-hierarchy|Filesystem Hierarchy]])
- Знание что такое файлы и папки
- Базовое знание команд ls, cd

## User and Group Basics

**Пользователь (User)** — аккаунт, может логиниться в систему и запускать процессы.

**Группа (Group)** — набор пользователей, упрощает управление правами.

**UID (User ID)** — уникальный числовой идентификатор пользователя (0-65535).

**GID (Group ID)** — уникальный числовой идентификатор группы.

**root** — суперпользователь с UID 0, может всё.

### User Database Files

**`/etc/passwd`** — информация о пользователях

Формат:
```
username:x:uid:gid:comment:home:shell
```

Пример:
```
root:x:0:0:root:/root:/bin/bash
user:x:1000:1000:User Name:/home/user:/bin/bash
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
```

**Просмотр:**
```bash
cat /etc/passwd                    # все пользователи
getent passwd username             # информация о конкретном
getent passwd | grep bash          # все с bash shell
```

**`/etc/shadow`** — зашифрованные пароли (только root может читать!)

```bash
sudo cat /etc/shadow               # требует sudo
# Формат: username:hashed_password:last_change:min:max:warning:inactive:expire
```

**`/etc/group`** — информация о группах

Формат:
```
groupname:x:gid:members
```

Пример:
```
sudo:x:27:user1,user2
docker:x:999:user1
```

**Просмотр:**
```bash
cat /etc/group                     # все группы
getent group groupname             # информация о группе
getent group | grep user1          # группы пользователя
```

## User Management Commands

### Просмотр текущего пользователя

```bash
whoami                             # кто я?
id                                 # мой UID, GID и группы
id username                        # информация о пользователе
groups                             # мои группы
groups username                    # группы пользователя
```

### Создание пользователя

```bash
sudo useradd username              # создать пользователя
sudo useradd -m username           # создать + домашняя папка
sudo useradd -m -s /bin/bash username  # с shell
sudo useradd -m -d /custom/home username  # с кастомной домашней папкой
sudo useradd -m -G group1,group2 username # добавить в группы
```

### Модификация пользователя

```bash
sudo usermod -aG group username    # добавить в группу
sudo usermod -d /new/home username # изменить домашнюю папку
sudo usermod -s /bin/zsh username  # изменить shell
sudo usermod -l newname username   # переименовать пользователя
```

### Удаление пользователя

```bash
sudo userdel username              # удалить пользователя (оставляет файлы)
sudo userdel -r username           # удалить + домашнюю папку и почту
```

### Управление паролями

```bash
passwd                             # изменить пароль текущего пользователя
sudo passwd username               # изменить пароль пользователя
sudo passwd -l username            # заблокировать пароль (нельзя логиниться)
sudo passwd -u username            # разблокировать пароль
sudo passwd -e username            # заставить сменить пароль при следующем логине
```

## Group Management Commands

### Просмотр групп

```bash
groups                             # мои группы
groups username                    # группы пользователя
cat /etc/group                     # все группы
getent group groupname             # информация о группе
getent group | grep groupname      # поиск по названию
```

### Создание группы

```bash
sudo groupadd groupname            # создать группу
sudo groupadd -g 1500 groupname    # с конкретным GID
```

### Модификация группы

```bash
sudo groupmod -n newname oldname   # переименовать
sudo groupmod -g 1600 groupname    # изменить GID
```

### Добавление пользователя в группу

```bash
sudo usermod -aG group username    # добавить в группу (сохраняет остальные)
sudo gpasswd -a username group     # альтернативный способ
```

### Удаление пользователя из группы

```bash
sudo gpasswd -d username group     # удалить из группы
```

### Удаление группы

```bash
sudo groupdel groupname            # удалить группу (пользователи остаются)
```

## File Permissions

### Understanding chmod

Каждый файл имеет три набора прав для трёх категорий:

```
-rw-r--r--
 ^   ^   ^
 |   |   +-- others (все остальные)
 |   +------ group (группа владельца)
 +---------- owner (владелец файла)
```

**Права (три буквы):**
- `r` (read) = 4 — читать
- `w` (write) = 2 — писать
- `x` (execute) = 1 — выполнять

### Numeric Format (Octal)

```
Комбинации:
r-- = 4
-w- = 2
--x = 1
rw- = 6 (4+2)
r-x = 5 (4+1)
rwx = 7 (4+2+1)
--- = 0
```

**Примеры:**
```
644 = rw-r--r--  (файл: owner пишет, все читают)
755 = rwxr-xr-x  (программа: owner полный доступ, остальные выполняют)
700 = rwx------  (приватное: только owner)
777 = rwxrwxrwx  (все могут всё - опасно!)
```

### Changing Permissions (chmod)

```bash
chmod 644 file                     # rw-r--r--
chmod 755 file                     # rwxr-xr-x
chmod 600 file                     # rw------- (приватное)
chmod 644 *.txt                    # для всех .txt файлов
chmod -R 755 directory             # рекурсивно для папки

# Текстовый формат
chmod u+x file                     # добавить execute для owner
chmod g+w file                     # добавить write для group
chmod o-r file                     # убрать read для others
chmod a+r file                     # добавить read для всех (a=all)
chmod u=rwx,g=rx,o= file           # явно установить все три
chmod go-rwx file                  # убрать все права для group и others
```

**Структура текстового формата:**
```
who:  u (user/owner), g (group), o (others), a (all)
op:   + (добавить), - (убрать), = (установить)
what: r (read), w (write), x (execute)
```

### Special Permissions

**setuid (4000)** — программа выполняется как владелец файла, не как пользователь

```bash
chmod 4755 file              # setuid + rwxr-xr-x
# Пример: passwd выполняется как root, хотя пользователь его запустил
ls -l /usr/bin/passwd        # -rwsr-xr-x (s вместо x)
```

**setgid (2000)** — программа выполняется как группа файла

```bash
chmod 2755 file              # setgid + rwxr-xr-x
# Файлы, созданные в такой папке, будут принадлежать её группе
```

**sticky bit (1000)** — в такой папке только владелец может удалить файл

```bash
chmod 1777 /tmp              # sticky bit + rwxrwxrwx
# /tmp: каждый может создавать и удалять только свои файлы
ls -ld /tmp                  # drwxrwxrwt (t вместо x)
```

## File Ownership

### Viewing and Changing Owner

```bash
ls -l file                   # показать владельца и группу
stat file                    # детальная информация

# Изменить владельца
sudo chown user file         # новый владелец
sudo chown user:group file   # новый владелец и группа
sudo chown :group file       # только группа
sudo chown user: file        # владелец и его primary группа

# Рекурсивно
sudo chown -R user:group directory  # папку и содержимое
```

### Common Patterns

```bash
# Делаю файл своим
sudo chown $USER file

# Файлы в www папке для web сервера
sudo chown -R www-data:www-data /var/www

# Логи для системы
sudo chown -R syslog:adm /var/log
```

## sudo - Execute as Root

**sudo** (super user do) — выполнить команду как root (или другой пользователь).

```bash
sudo command                       # выполнить как root
sudo -u username command           # выполнить как другой пользователь
sudo -l                            # какие команды можно без пароля
sudo -k                            # забыть пароль (потребуется при следующей)
sudo !!                            # повторить последнюю команду с sudo
sudo apt update && sudo apt upgrade # цепочка команд
```

### sudo Configuration

```bash
sudo visudo                   # безопасное редактирование sudoers
# Это откроет /etc/sudoers в редакторе
```

**Примеры sudoers:**
```
# Пользователь user1 может запускать apt без пароля
user1 ALL=(ALL) NOPASSWD: /usr/bin/apt

# Группа sudo может всё
%sudo ALL=(ALL:ALL) ALL

# Пользователь user2 может перезагружать nginx без пароля
user2 ALL=(ALL) NOPASSWD: /usr/sbin/systemctl restart nginx
```

## su - Switch User

**su** (switch user) — переключиться на другого пользователя.

```bash
su                             # переключиться на root (требует пароля root!)
su - root                      # с загрузкой profile
su - username                  # переключиться на пользователя
su -c "command" username       # выполнить команду как пользователь
exit                           # выйти
```

**Разница:**
- `su` — без `-` не загружает environment
- `su -` — с `-` загружает profile (рекомендуется)

## Permission Matrix

### Для файлов

| Право | Значение |
|-------|----------|
| **r** (read) | читать содержимое файла |
| **w** (write) | писать в файл |
| **x** (execute) | выполнять как программу |

### Для папок

| Право | Значение |
|-------|----------|
| **r** (read) | просмотреть список файлов (ls) |
| **w** (write) | создавать/удалять файлы в папке |
| **x** (execute) | входить в папку (cd) |

## Troubleshooting

### Нет прав на файл

```bash
# Проверьте права
ls -l file
stat file

# Если вы владелец:
chmod u+r file               # добавить себе права на чтение

# Если нужен root:
sudo cat file
```

### Хочу выполнить программу

```bash
# Если нет прав execute:
chmod u+x program            # добавить себе права
./program                    # запустить

# Или через sudo:
sudo program
```

### Не могу создавать файлы в папке

```bash
# Проверить права на папку
ls -ld /path/to/dir          # правый синтаксис для папки

# Если нужна write права:
chmod u+w /path/to/dir       # для себя
chmod g+w /path/to/dir       # для группы
chmod o+w /path/to/dir       # для остальных
```

### Забыл пароль пользователя

```bash
# Как администратор:
sudo passwd username         # установить новый пароль
```

### Хочу что-то сделать без sudo, но требует root

**Способ 1: добавить в группу**
```bash
# Некоторые команды работают через группы
# Например: docker требует быть в группе docker
sudo usermod -aG docker $USER
newgrp docker                # активировать новую группу
docker ps                    # теперь работает
```

**Способ 2: изменить права**
```bash
# Последняя мера - измените права
sudo chmod 755 /path/to/thing
```

**Способ 3: добавить в sudoers**
```bash
sudo visudo
# Добавьте:
# $USER ALL=(ALL) NOPASSWD: /usr/bin/command
```

### Посмотреть историю логинов

```bash
last                          # история логинов
lastlog                       # последний логин каждого пользователя
who                           # кто сейчас логинен
w                             # кто логинен + чем занимается
```

## Cheat Sheet

```bash
# Просмотр
whoami                        # я кто
id                            # мой UID/GID/группы
cat /etc/passwd               # все пользователи
getent group                  # все группы
ls -l file                    # права файла

# Пользователи
sudo useradd -m user          # создать пользователя
sudo userdel -r user          # удалить пользователя
sudo passwd user              # изменить пароль

# Группы
sudo groupadd group           # создать группу
sudo usermod -aG group user   # добавить в группу
sudo groupdel group           # удалить группу

# Права
chmod 755 file                # rwxr-xr-x
chmod 644 file                # rw-r--r--
chmod +x script                # добавить execute
sudo chown user:group file     # изменить владельца

# Управление
sudo command                  # как root
su - user                     # переключиться на пользователя
exit                          # выйти
```

## Key Takeaways

- **UID/GID** — уникальные числовые идентификаторы
- **Три уровня прав** — owner, group, others
- **chmod число** — быстро устанавливать права (755, 644, 600)
- **chmod буквы** — гибко добавлять/убирать права (u+x, g-w)
- **chown** — изменять владельца и группу
- **sudo** — выполнять команды от root'а (требует в sudoers)
- **Сначала подумайте, потом chmod 777** — это опасно!

## Related

Следующий шаг:
- [[03-processes-and-services|Processes and Services]] — управление процессами и сервисами

Предыдущий шаг:
- [[01-filesystem-hierarchy|Filesystem Hierarchy]] — структура файловой системы

Контекст:
- [[docs/_inbox/01-linux/01-core-concepts/README|Core Concepts Index]] — полный индекс этого раздела

## See Also

Встроенная справка:
- `man chmod` — справка по chmod
- `man chown` — справка по chown
- `man sudo` — справка по sudo
- `man sudoers` — формат sudoers файла
- `man useradd` — справка по useradd

Онлайн ресурсы:
- [Linux file permissions](https://wiki.archlinux.org/title/File_permissions_and_attributes) — Arch Wiki
