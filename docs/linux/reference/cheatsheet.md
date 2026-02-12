---
title: "Справочник: Команды Linux"
type: reference
tags: [linux, cheatsheet, commands, reference]
related:
  - "[[linux/reference/filesystem-hierarchy]]"
  - "[[linux/reference/permissions-table]]"
  - "[[linux/reference/systemd-reference]]"
---

# Справочник: Команды Linux

## Навигация и файлы

```bash
pwd                          # текущая директория
ls -lah                      # подробный список с скрытыми
cd /path                     # перейти | cd ~ домой | cd - назад
tree -L 2                    # дерево (2 уровня)

touch file                   # создать / обновить время
mkdir -p a/b/c               # создать с родительскими
cp -r src/ dst/              # копировать рекурсивно
mv old new                   # переместить / переименовать
rm -rf dir/                  # удалить рекурсивно
ln -s target link            # символическая ссылка
```

## Просмотр и поиск

```bash
cat file                     # весь файл
head -20 / tail -20 file     # начало / конец
tail -f file                 # follow (логи)
less file                    # постраничный (q — выход)
wc -l file                   # кол-во строк

grep -rn "pattern" dir/      # поиск текста (рекурсивно, с номерами строк)
grep -i "pattern" file       # без учёта регистра
find / -name "*.conf"        # поиск файлов по имени
find / -size +100M           # большие файлы
locate filename              # быстрый поиск (по индексу)
which command                # где binary
```

## Текстовые утилиты

```bash
sort file | uniq -c          # сортировка + подсчёт уникальных
cut -d: -f1 /etc/passwd      # извлечь колонку
awk '{print $1, $3}' file    # обработка колонок
sed 's/old/new/g' file       # замена текста
sed -i 's/old/new/g' file    # замена в файле
xargs                        # stdin → аргументы
```

## Перенаправление

```bash
cmd > file                   # stdout → файл (перезаписать)
cmd >> file                  # stdout → файл (дописать)
cmd 2> err.log               # stderr → файл
cmd &> all.log               # stdout + stderr
cmd < input                  # stdin ← файл
cmd1 | cmd2                  # pipe (stdout → stdin)
```

## Пользователи и права

```bash
whoami / id                  # кто я / детальная инфо
sudo command                 # от root
su - user                    # стать пользователем

chmod 755 file               # rwxr-xr-x
chmod +x script.sh           # добавить execute
chown user:group file        # сменить владельца
chown -R user:group dir/     # рекурсивно

useradd -m -s /bin/bash user # создать
usermod -aG group user       # добавить в группу (с -a!)
passwd user                  # сменить пароль
```

## Процессы

```bash
ps aux                       # все процессы
ps aux | grep name           # найти процесс
top / htop                   # интерактивный мониторинг
kill PID                     # SIGTERM (мягко)
kill -9 PID                  # SIGKILL (принудительно)
killall name                 # по имени
bg / fg                      # фон / передний план
nohup cmd &                  # продолжить после logout
```

## Systemd

```bash
systemctl start|stop|restart svc
systemctl enable|disable svc
systemctl status svc
systemctl --failed
journalctl -u svc -f         # логи (follow)
journalctl -b -p err         # ошибки с загрузки
```

## Сеть

```bash
ip a                         # интерфейсы и IP
ip route                     # маршруты
ss -tlnp                     # listening TCP ports
ping -c 3 host               # проверка связи
curl -I url                  # HTTP заголовки
dig domain +short            # DNS lookup
ssh user@host                # удалённое подключение
scp file user@host:/path     # копировать по SSH
```

## Диски и память

```bash
df -h                        # свободное место
du -sh dir/                  # размер директории
free -h                      # RAM / swap
lsblk                        # блочные устройства
mount / umount               # монтирование
```

## Пакеты

```bash
# apt (Debian/Ubuntu)        # pacman (Arch)           # dnf (Fedora)
apt update                   # pacman -Sy              # dnf check-update
apt upgrade                  # pacman -Syu             # dnf upgrade
apt install pkg              # pacman -S pkg           # dnf install pkg
apt remove pkg               # pacman -Rs pkg          # dnf remove pkg
apt search pkg               # pacman -Ss pkg          # dnf search pkg
```

## Разное

```bash
date                         # дата/время
uptime                       # аптайм + load average
uname -a                     # информация о системе
cat /etc/os-release          # дистрибутив
history                      # история команд
alias ll='ls -la'            # создать alias
echo $PATH                   # пути поиска команд
```