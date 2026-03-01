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
pwd -P                       # реальный путь (без символических ссылок)
ls -lah                      # подробный список с скрытыми
ls -F                        # суффиксы типов: / каталог, * исполняемый, @ ссылка
cd /path                     # перейти | cd ~ домой | cd - назад
tree -L 2                    # дерево (2 уровня)

touch file                   # создать / обновить время
mkdir -p a/b/c               # создать с родительскими
rmdir dir                    # удалить пустой каталог (ошибка если не пуст)
cp file1 file2               # копировать файл
cp -r src/ dst/              # копировать рекурсивно
mv old new                   # переместить / переименовать
rm file                      # удалить файл
rm -rf dir/                  # удалить рекурсивно (осторожно!)
ln -s target link            # символическая ссылка
echo "text"                  # вывод в stdout (полезно для отладки, шаблонов)
```

## Просмотр и поиск

```bash
cat file                     # весь файл
cat file1 file2              # конкатенация нескольких файлов
head -20 / tail -20 file     # начало / конец
tail -f file                 # follow (логи)
tail +5 file                 # с 5-й строки до конца
less file                    # постраничный (q — выход, /word — поиск, n — следующее совпадение)
wc -l file                   # кол-во строк

file document.bin            # определить тип файла по содержимому
diff file1 file2             # различия между файлами
diff -u file1 file2          # unified-формат (для patch и code review)

grep -rn "pattern" dir/      # поиск текста (рекурсивно, с номерами строк)
grep -i "pattern" file       # без учёта регистра
grep -v "pattern" file       # инвертировать: строки НЕ содержащие pattern
grep -E "foo|bar" file       # расширенные regex (egrep): | ? + () без экранирования
find / -name "*.conf"        # поиск файлов по имени
find / -name '*.log' -print  # кавычки обязательны (иначе shell раскроет *)
find / -size +100M           # большие файлы
locate filename              # быстрый поиск (по индексу, обновляется updatedb)
which command                # где binary
```

## Текстовые утилиты

```bash
sort file                    # сортировка (алфавитная)
sort -n file                 # числовая сортировка
sort -r file                 # обратный порядок
sort file | uniq -c          # сортировка + подсчёт уникальных
cut -d: -f1 /etc/passwd      # извлечь колонку
awk '{print $1, $3}' file    # обработка колонок
sed 's/old/new/g' file       # замена текста
sed -i 's/old/new/g' file    # замена в файле
xargs                        # stdin → аргументы
basename /usr/bin/app        # → app (имя файла из пути)
basename file.tar.gz .tar.gz # → file (удалить суффикс)
```

## Перенаправление и потоки

Три стандартных потока: stdin (fd 0) — ввод, stdout (fd 1) — вывод, stderr (fd 2) — ошибки.

```bash
cmd > file                   # stdout → файл (перезаписать)
cmd >> file                  # stdout → файл (дописать)
cmd 2> err.log               # stderr → файл
cmd &> all.log               # stdout + stderr
cmd < input                  # stdin ← файл
cmd1 | cmd2                  # pipe (stdout cmd1 → stdin cmd2)
cmd1 | less                  # постраничный просмотр длинного вывода
```

| Сочетание | Значение |
|---|---|
| Ctrl+D | EOF — завершить ввод (на пустой строке) |
| Ctrl+C | SIGINT — прервать текущую программу |
| Ctrl+Z | SIGTSTP — приостановить (fg — продолжить) |

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