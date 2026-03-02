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
chsh -s /bin/zsh             # сменить оболочку по умолчанию

env                          # все переменные окружения
export VAR="value"           # создать переменную окружения
echo $PATH                   # каталоги поиска команд
```

## Процессы

```bash
ps aux                       # все процессы
ps aux | grep name           # найти процесс
ps aux --sort=-%mem | head   # топ по памяти
ps aux --sort=-%cpu | head   # топ по CPU
ps m -p PID                  # потоки процесса
pstree                       # дерево процессов
pgrep -l nginx               # PID по имени
pidof nginx                  # PID по имени (только)
top / htop                   # интерактивный мониторинг
kill PID                     # SIGTERM (мягко)
kill -9 PID                  # SIGKILL (принудительно)
kill -HUP PID                # перечитать конфиг
killall name                 # по имени
pkill -f "python app"        # по части командной строки
bg / fg                      # фон / передний план
nohup cmd &                  # продолжить после logout

# Мониторинг ресурсов
uptime                       # load average
vmstat 1 5                   # CPU/mem/swap/io (1 сек, 5 раз)
iostat -xz 1                 # дисковый I/O
sudo iotop                   # I/O по процессам
pidstat -p PID 1             # CPU процесса (1 сек)
pidstat -r -p PID 1          # память процесса
pidstat -d -p PID 1          # disk I/O процесса

# Открытые файлы и порты
lsof -p PID                  # файлы процесса
lsof +D /path/               # кто использует каталог
lsof -i :80                  # кто слушает порт
fuser /path/to/file          # PID использующий файл

# Трассировка
strace -p PID                # системные вызовы
strace -e trace=open,read cmd   # только open/read
strace -c cmd                # статистика вызовов
ltrace cmd                   # вызовы библиотек
```

## Systemd

```bash
# Управление сервисами
systemctl start|stop|restart svc
systemctl enable|disable svc      # автозапуск
systemctl enable --now svc        # автозапуск + запустить сейчас
systemctl status svc
systemctl reload svc              # перечитать конфиг без перезапуска
systemctl daemon-reload           # перечитать unit-файлы (после редактирования)
systemctl --failed                # сломанные юниты

# Зависимости и информация
systemctl list-dependencies svc   # от чего зависит
systemctl cat svc                 # показать unit-файл
systemctl show svc                # все свойства
systemctl list-units --type=service   # все сервисы
systemctl list-sockets            # все сокеты

# Targets
systemctl get-default             # текущий target
sudo systemctl set-default multi-user.target
sudo systemctl isolate rescue.target   # переключиться сейчас

# Таймеры
systemctl list-timers             # все таймеры

# Временный запуск
sudo systemd-run --unit=test sleep 60   # создать transient unit

# Анализ загрузки
systemd-analyze                   # общее время загрузки
systemd-analyze blame             # самые медленные юниты
systemd-analyze critical-chain    # критический путь

# Логи (journalctl)
journalctl -u svc -f              # follow (реальное время)
journalctl -u svc -n 50           # последние 50 строк
journalctl -b -p err              # ошибки с загрузки
journalctl -b -1                  # логи предыдущей загрузки
journalctl --list-boots            # список загрузок
journalctl -S "1 hour ago"        # за последний час
journalctl -k                     # логи ядра (dmesg)
journalctl -g "pattern"           # поиск regex
journalctl --disk-usage           # сколько занимают логи
sudo journalctl --vacuum-size=500M   # очистить до 500MB

# Выключение/перезагрузка
shutdown -h now                   # выключить
shutdown -r now                   # перезагрузить
shutdown -h +10 "Maintenance"     # через 10 мин с сообщением
shutdown -c                       # отменить
```

## Сеть

```bash
# Интерфейсы и IP
ip a                         # все интерфейсы и адреса
ip -4 a                      # только IPv4
ip -6 a                      # только IPv6
ip -br a                     # краткий формат
ip link show                 # состояние интерфейсов (без IP)
ip link set eth0 up          # поднять интерфейс
ip link set eth0 down        # опустить

# Назначение IP (временно)
sudo ip addr add 192.168.1.100/24 dev eth0
sudo ip addr del 192.168.1.100/24 dev eth0

# Маршруты
ip route                     # таблица маршрутизации
ip -6 route                  # IPv6 маршруты
ip route get 8.8.8.8         # через какой интерфейс/gateway
sudo ip route add default via 192.168.1.1 dev eth0
sudo ip route del default
sudo ip route add 10.0.0.0/24 via 192.168.1.1

# Диагностика
ping -c 3 host               # ICMP (3 пакета)
ping -4 host                 # принудительно IPv4
ping -6 host                 # принудительно IPv6
traceroute host              # маршрут до хоста
tracepath host               # аналог без root

# DNS
host example.com             # DNS lookup (A + AAAA)
host 8.8.8.8                 # reverse DNS (IP → имя)
dig example.com +short       # детальный DNS lookup
resolvectl status            # DNS-серверы по интерфейсам
cat /etc/resolv.conf         # DNS-конфигурация

# Порты и соединения
ss -tn                       # активные TCP соединения
ss -tlnp                     # listening TCP с процессами
ss -un                       # активные UDP
ss -ulnp                     # listening UDP с процессами

# ARP / соседи
ip -4 neigh                  # ARP-кеш (IPv4)
ip -6 neigh                  # NDP-кеш (IPv6)

# Wi-Fi
iw dev wlan0 scan | grep SSID   # сканировать сети
iw dev wlan0 link               # текущее подключение
nmcli dev wifi list              # Wi-Fi через NetworkManager

# Firewall
sudo iptables -L -n          # правила (IPv4)
sudo iptables -L -n -t nat   # NAT-правила
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -P INPUT DROP  # политика по умолчанию

# Маршрутизация (forwarding)
sysctl net.ipv4.ip_forward        # статус
sudo sysctl -w net.ipv4.ip_forward=1   # включить (временно)

# DHCP
sudo dhclient eth0           # запросить IP по DHCP

# HTTP и передача файлов
curl -I url                  # HTTP заголовки
curl -o file url             # скачать
ssh user@host                # удалённое подключение
scp file user@host:/path     # копировать по SSH
scp user@host:/path file     # скачать по SSH
ethtool eth0                 # параметры Ethernet-карты
```

## Диски и память

```bash
df -h                        # свободное место
df -i                        # использование inode
du -sh dir/                  # размер директории
du -sh * | sort -rh | head   # топ по размеру
free -h                      # RAM / swap
lsblk                        # блочные устройства (дерево)
lsblk -f                     # с FS и UUID
sudo blkid                   # UUID всех разделов
mount | column -t            # что куда смонтировано
sudo mount /dev/sda2 /mnt    # монтирование
sudo umount /mnt             # размонтирование
sudo fdisk -l /dev/sda       # таблица разделов
swapon --show                # swap-разделы

# LVM
sudo pvs / vgs / lvs         # обзор PV / VG / LV
sudo lvextend -L +5G /dev/vg/lv   # расширить LV
sudo resize2fs /dev/vg/lv         # расширить ext4
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

## Загрузка и ядро

```bash
uname -r                     # версия ядра
uname -a                     # полная информация
cat /proc/cmdline            # параметры ядра
lsmod                        # загруженные модули
sudo modprobe module         # загрузить модуль
dmesg | tail -30             # сообщения ядра

# GRUB
sudo update-grub             # перегенерировать grub.cfg
sudo grub-install /dev/sda   # установить GRUB (MBR)
efibootmgr -v                # записи UEFI загрузки
```

## Время

```bash
date                         # текущая дата/время
timedatectl                  # полный статус (timezone, NTP, RTC)
timedatectl set-timezone Europe/Moscow
timedatectl set-ntp true     # включить NTP-синхронизацию
hwclock --show               # аппаратные часы (RTC)
```

## Планировщики

```bash
crontab -e                   # редактировать crontab
crontab -l                   # показать текущий
# m h dom mon dow command
# */5 * * * * /opt/check.sh  # каждые 5 мин
systemctl list-timers        # systemd таймеры
```

## Разное

```bash
uptime                       # аптайм + load average
cat /etc/os-release          # дистрибутив
history                      # история команд
alias ll='ls -la'            # создать alias
stat file                    # inode, размер, время, права
namei -l /full/path          # права каждого компонента пути
```