---
created: 2026-01-04
updated: 2026-01-04
tags: [linux, security, hardening, firewall, ssh, permissions, reference]
type: reference
---

# Защита системы Linux (System Hardening)

## Основная идея

**Linux по умолчанию НЕ безопасен.** Правильная конфигурация - это комплекс мер защиты:
- Минимальные права доступа (principle of least privilege)
- Отключение ненужных сервисов
- Сильные пароли и SSH ключи
- Firewall
- Регулярные обновления
- Логирование и мониторинг

**Три уровня защиты:**
1. **Периметр** - firewall, открытые порты
2. **ОС** - права доступа, пользователи, сервисы
3. **Приложение** - конфиги, секреты, логирование

**Почему это критично:**
- Взломанная система = потеря данных, компромиз сети
- SSH на порту 22 - первая цель (brute force атаки)
- root права при использовании неправильно = уязвимость
- Ненужные сервисы = лишние порты открыты
- Слабые пароли = быстрый взлом

---

## ЧАСТЬ 1: SSH Hardening (критично!)

### Проблема по умолчанию

```bash
# По умолчанию SSH позволяет:
# 1. root логин (очень опасно!)
# 2. Password authentication (brute force атаки)
# 3. Слушает на всех интерфейсах

# Результат: bot'ы брутят root пароль 24/7
# В логах: /var/log/auth.log заполнен неудачными попытками входа
```

### Хардирование SSH (ВСЕ дистры)

**Правильная конфигурация:**
```bash
sudo nano /etc/ssh/sshd_config
```

```ini
# /etc/ssh/sshd_config

# Слушать только на конкретном интерфейсе и порту
Port 2222                          # ❌ Изменить с 22 (не обязательно, но помогает)
ListenAddress 192.168.1.100        # Слушать только на этом IP
# ListenAddress ::1                # Только IPv6 localhost (если нужно)

# ❌ Запретить root логин (КРИТИЧНО!)
PermitRootLogin no

# ❌ Запретить пароли, только ключи (если возможно)
PasswordAuthentication no
PubkeyAuthentication yes

# Запретить пустые пароли
PermitEmptyPasswords no

# Логин только через SSH ключ
AuthenticationMethods publickey
# или если нужны пароли:
AuthenticationMethods password publickey

# Отключить X11 forwarding (если не нужен GUI)
X11Forwarding no

# Отключить port forwarding (если не нужно)
AllowTcpForwarding no
AllowStreamLocalForwarding no

# Максимальное кол-во попыток (чтобы bot'ы не палили в логах)
MaxAuthTries 3
MaxSessions 5

# Timeout для неактивных сессий
ClientAliveInterval 300
ClientAliveCountMax 2

# Отключить старые небезопасные алгоритмы
HostKeyAlgorithms ssh-ed25519,rsa-sha2-512,rsa-sha2-256
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
Ciphers chacha20-poly1305@openssh.com,aes-256-gcm@openssh.com,aes-128-gcm@openssh.com,aes-256-ctr,aes-192-ctr,aes-128-ctr
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-sha2-512,hmac-sha2-256

# Отключить небезопасные опции
IgnoreRhosts yes
HostbasedAuthentication no
RhostsRSAAuthentication no
RSAAuthentication no

# Дополнительная защита
StrictModes yes
```

**Применить конфиг:**
```bash
# Arch:
sudo systemctl restart sshd

# Ubuntu/Debian:
sudo systemctl restart ssh

# Проверить что работает (важно - может заблокировать себя!)
# Открыть новый терминал и попробовать подключиться перед тем как выйти
ssh -p 2222 user@localhost
```

### SSH ключи вместо паролей

**Создать SSH ключ (если нет):**
```bash
ssh-keygen -t ed25519 -C "user@arch" -f ~/.ssh/id_ed25519
# -t ed25519 = современный, безопасный алгоритм
# -C = комментарий (опционально)
# -f = где сохранить

# Вывод запросит пароль (passphrase) - используй сильный пароль!
```

**Правильные права для ключей:**
```bash
# КРИТИЧНО! Ключи должны быть приватными
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
chmod 644 ~/.ssh/authorized_keys

# Проверить
ls -la ~/.ssh/
# drwx------ = 700 (папка)
# -rw------- = 600 (приватный ключ)
# -rw-r--r-- = 644 (публичный ключ)
```

**Скопировать публичный ключ на сервер:**
```bash
# Способ 1 (автоматический)
ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 2222 user@server.com

# Способ 2 (ручной)
cat ~/.ssh/id_ed25519.pub | ssh -p 2222 user@server.com "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# Способ 3 (вручную)
scp -P 2222 ~/.ssh/id_ed25519.pub user@server.com:~/
ssh -p 2222 user@server.com "cat ~/id_ed25519.pub >> ~/.ssh/authorized_keys && rm ~/id_ed25519.pub"
```

**Проверить что работает:**
```bash
ssh -i ~/.ssh/id_ed25519 -p 2222 user@server.com
# Должно войти БЕЗ пароля (используя ключ)
```

---

## ЧАСТЬ 2: Файловая система и права доступа

### Основной принцип: Least Privilege

```bash
# ❌ ПЛОХО:
chmod 777 /home/user/documents
# Все могут читать, писать, исполнять

# ✅ ХОРОШО:
chmod 750 /home/user/documents
# Владелец: rwx
# Группа: r-x
# Другие: ---

# ❌ ПЛОХО:
ls -l /etc/passwd
# -rw-r--r-- (644)
# Все могут читать пароли (хэши)

# ✅ ХОРОШО (по умолчанию на Arch):
ls -l /etc/shadow
# -rw-r----- (640)
# Только владелец (root) и группа (shadow) могут читать
```

### Критичные файлы и права

```bash
# SSH ключи (КРИТИЧНО!)
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_*
chmod 644 ~/.ssh/authorized_keys
chmod 644 ~/.ssh/known_hosts

# Файлы с секретами (конфиги, API ключи)
chmod 600 ~/.config/myapp/secrets.conf
chmod 600 /etc/myapp/database.conf

# Домашняя папка
chmod 750 /home/user  # Владелец может всё, группа может входить, другие ничего

# Временные файлы
chmod 1777 /tmp       # Все могут писать, но удалять только свои (sticky bit)
```

### Аудит файловых прав

```bash
# Найти файлы с опасными правами
find / -perm /002 -type f 2>/dev/null
# Файлы которые могут писать "другие" (very bad!)

find / -perm /020 -type f 2>/dev/null
# Файлы которые может писать "группа" (проверить)

find ~ -perm 777 2>/dev/null
# В домашней папке найти 777 права

# Проверить SSH ключи
find ~/.ssh -type f ! -perm 600 -ls
# Найти файлы в ~/.ssh с неправильными правами
```

---

## ЧАСТЬ 3: Управление пользователями и sudo

### Минимизация root использования

```bash
# ❌ ПЛОХО:
sudo su
# Ты root, все команды = root (можно случайно удалить систему)

# ✅ ХОРОШО:
sudo command
# Только эта команда от root
```

### Sudoers конфигурация (безопасно!)

```bash
# ВСЕГДА используй visudo! (проверяет синтаксис)
sudo visudo

# Основная конфигурация:
%sudo ALL=(ALL:ALL) ALL
# Группа sudo может выполнять любую команду (требует пароль)

# Без пароля (осторожно!):
%docker ALL=(ALL) NOPASSWD: docker
# Группа docker может выполнять "docker" БЕЗ пароля

# Ограничить на конкретные команды:
user ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx
# user может выполнять только "systemctl restart nginx" без пароля
```

### Правильное управление группами

```bash
# Добавить пользователя в группу sudo (для администратора)
sudo usermod -aG sudo username

# Добавить в docker (для работы с контейнерами)
sudo usermod -aG docker username

# ❌ НЕ добавляй в группу root!
# sudo usermod -aG root username    # НЕЛЬЗЯ!

# Проверить группы пользователя
id username
groups username
```

---

## ЧАСТЬ 4: Firewall (UFW/iptables)

### UFW основные правила (ВСЕ дистры)

```bash
# Arch:
sudo pacman -S ufw

# Ubuntu/Debian:
sudo apt install ufw

# Базовая конфигурация (fail-safe):
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw default deny routed

# Разрешить только нужные порты
sudo ufw allow ssh         # 22
sudo ufw allow http        # 80
sudo ufw allow https       # 443

# Если SSH на нестандартном порту:
sudo ufw allow 2222/tcp

# Разрешить только с конкретного IP
sudo ufw allow from 192.168.1.0/24 to any port 22

# Включить firewall
sudo ufw enable

# Проверить правила
sudo ufw status verbose
```

### iptables для продвинутого контроля

```bash
# Arch:
sudo pacman -S iptables

# Базовая политика (drop all)
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# Разрешить localhost
sudo iptables -A INPUT -i lo -j ACCEPT

# Разрешить установленные соединения
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Разрешить входящее SSH
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Защита от port scanning
sudo iptables -N port-scanning
sudo iptables -A port-scanning -p tcp --tcp-flags SYN,ACK,FIN,RST RST -m limit --limit 1/s --limit-burst 2 -j RETURN
sudo iptables -A port-scanning -j DROP

# Сохранить правила
sudo iptables-save > /etc/iptables/iptables.rules
sudo systemctl enable iptables
sudo systemctl start iptables
```

---

## ЧАСТЬ 5: Отключение ненужных сервисов

### Arch Linux способ

```bash
# Посмотреть какие сервисы запущены
systemctl list-units --type=service --state=running

# Посмотреть включённые (будут запускаться при загрузке)
systemctl list-unit-files --type=service --state=enabled

# Отключить сервис (не запускаться при загрузке)
sudo systemctl disable service-name

# Остановить сервис (прямо сейчас)
sudo systemctl stop service-name

# Примеры ненужных сервисов:
sudo systemctl disable bluetooth.service  # Если не нужен Bluetooth
sudo systemctl disable cups.service       # Если не нужен принтер
sudo systemctl disable avahi-daemon.service  # Если не нужен mDNS
sudo systemctl disable ModemManager.service  # Если нет модема
```

### Ubuntu/Debian способ

```bash
# Посмотреть какие сервисы запущены
systemctl list-units --type=service --state=running

# Отключить сервис
sudo systemctl disable service-name
sudo systemctl stop service-name

# Удалить пакет (если совсем не нужен)
sudo apt remove package-name
```

### Проверка открытых портов

```bash
# Какие порты слушают
sudo ss -tlnp

# Или (старый способ):
sudo netstat -tlnp

# Если видишь неизвестный порт - выясни какой сервис
# и отключи его если не нужен
```

---

## ЧАСТЬ 6: Обновления и исправления

### Arch Linux

```bash
# Обновить систему (ОЧЕНЬ ВАЖНО!)
sudo pacman -Syu

# Проверить обновления для AUR
yay -Syu

# Автоматические обновления (опционально)
# Установить: sudo pacman -S unattended-upgrades
# Конфиг: /etc/systemd/system/unattended-upgrades.service
```

### Ubuntu/Debian

```bash
# Обновить списки пакетов
sudo apt update

# Обновить пакеты
sudo apt upgrade

# Обновить с изменением зависимостей
sudo apt full-upgrade

# Автоматические обновления
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

### Проверка обновлений безопасности

```bash
# Arch:
pacman -Q --upgrades

# Ubuntu/Debian:
apt list --upgradable
sudo apt show -a package-name

# Что обновлять в первую очередь:
# 1. Kernel (ядро Linux)
# 2. Пакеты безопасности
# 3. SSH (если используется)
# 4. OpenSSL, GnuTLS (криптография)
```

---

## ЧАСТЬ 7: Логирование и мониторинг

### Просмотр логов (ВСЕ дистры)

```bash
# Systemd журнал (ВСЕ современные дистры)
journalctl -n 50          # 50 последних строк
journalctl -f             # Follow (live, Ctrl+C чтобы выйти)
journalctl -u sshd        # Логи SSH сервера
journalctl -u sshd -f     # SSH логи live
journalctl -p err         # Только ошибки
journalctl -S "-1 hour"   # Последний час
journalctl --since "2026-01-04 00:00" --until "2026-01-04 23:59"

# Сохранить логи на диск
sudo journalctl --vacuum-time=30d  # Хранить 30 дней
```

### Мониторинг SSH попыток входа

```bash
# Посмотреть неудачные попытки SSH
journalctl -u sshd | grep "Failed password"

# Или (старый способ, может быть):
grep "Failed password" /var/log/auth.log | tail -20

# Посмотреть успешные логины
journalctl -u sshd | grep "Accepted"

# Посмотреть кто пытается залогиниться как root
journalctl -u sshd | grep "root"
```

### Автоматизация мониторинга (через systemd timer)

```ini
# /etc/systemd/system/security-check.timer

[Unit]
Description=Daily Security Check

[Timer]
OnCalendar=*-*-* 02:00:00
Unit=security-check.service

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/security-check.service

[Unit]
Description=Check security logs

[Service]
Type=oneshot
ExecStart=/usr/local/bin/check-security.sh

# Скрипт проверяет:
# - Failed SSH attempts
# - Sudo commands
# - Changed permissions
# - Open ports changes
```

---

## ЧАСТЬ 8: Защита данных (шифрование)

### Шифрование в покое (LUKS)

```bash
# Arch:
sudo pacman -S cryptsetup

# Создать зашифрованный диск
sudo cryptsetup luksFormat /dev/sdX
# Вводишь пароль (СИЛЬНЫЙ!)

# Открыть зашифрованный диск
sudo cryptsetup open /dev/sdX encrypted_data

# Использовать как обычный диск
sudo mkfs.ext4 /dev/mapper/encrypted_data
sudo mount /dev/mapper/encrypted_data /mnt/data

# При отключении:
sudo umount /mnt/data
sudo cryptsetup close encrypted_data
```

### Шифрование при передаче

```bash
# SSH - уже зашифрован ✅

# HTTPS вместо HTTP ✅

# VPN для общественной сети ✅

# PGP/GPG для почты (опционально)
```

### Защита приватных файлов

```bash
# Домашняя папка должна быть 750 (не 755!)
chmod 750 /home/user

# Приватные конфиги должны быть 600
chmod 600 ~/.config/myapp/secrets.conf

# Кэш браузера должен быть приватным
chmod 700 ~/.cache
```

---

## ЧАСТЬ 9: Практический сценарий - Hardened Arch Server

### Полная настройка

**Шаг 1: SSH Hardening**
```bash
sudo nano /etc/ssh/sshd_config
# Изменить согласно ЧАСТИ 1
# Port 2222
# PermitRootLogin no
# PasswordAuthentication no

sudo systemctl restart sshd
```

**Шаг 2: Права доступа**
```bash
# SSH ключи
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub

# Домашняя папка
chmod 750 /home/user

# Конфиги с секретами
chmod 600 ~/.config/app/secrets.conf
```

**Шаг 3: Firewall**
```bash
sudo pacman -S ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2222/tcp  # SSH на порту 2222
sudo ufw enable
sudo ufw status verbose
```

**Шаг 4: Отключить ненужные сервисы**
```bash
systemctl list-unit-files --type=service --state=enabled | grep -v systemd-
# Посмотреть что запущено

sudo systemctl disable bluetooth.service
sudo systemctl disable cups.service
sudo systemctl stop bluetooth.service
```

**Шаг 5: Обновления**
```bash
sudo pacman -Syu
yay -Syu

# Проверить что обновилось
pacman -Q --upgrades
```

**Шаг 6: Мониторинг**
```bash
# Проверить логи SSH
journalctl -u sshd -n 50

# Проверить открытые порты
sudo ss -tlnp

# Проверить включённые сервисы
systemctl list-unit-files --type=service --state=enabled
```

---

## ЧАСТЬ 10: Чек-лист безопасности

### Критичное (MUST HAVE)

```bash
□ SSH PermitRootLogin no          (КРИТИЧНО!)
□ SSH PasswordAuthentication no   (использовать ключи)
□ SSH rights chmod 600/.ssh/      (КРИТИЧНО!)
□ Firewall включен (ufw enable)   (КРИТИЧНО!)
□ Sudo правила настроены          (БЕЗ NOPASSWD для важных команд)
□ Обновления установлены          (pacman -Syu или apt upgrade)
□ Ненужные сервисы отключены      (bluetooth, cups, avahi, и т.д.)
```

### Высокоприоритетное (SHOULD HAVE)

```bash
□ SSH ключи вместо паролей
□ Права доступа проверены (chmod)
□ Sudo без пароля только для безопасных команд
□ Резервные копии настроены
□ Логирование включено (journalctl)
□ Port не стандартный (не 22)
```

### Мониторинг (NICE TO HAVE)

```bash
□ Мониторинг SSH попыток входа
□ Проверка открытых портов регулярно
□ Проверка прав доступа регулярно
□ Обновления безопасности проверяются
□ Логирование аудита включено
```

---

## ЧАСТЬ 11: Шпаргалка (быстрая справка)

### SSH

```bash
sudo nano /etc/ssh/sshd_config    # Редактировать конфиг
sudo systemctl restart sshd        # Применить изменения
ssh-keygen -t ed25519              # Создать ключ
ssh-copy-id user@host              # Скопировать ключ на сервер
chmod 700 ~/.ssh                   # Правильные права
chmod 600 ~/.ssh/id_*              # Для ключей
```

### Права доступа

```bash
chmod 750 /home/user               # Домашняя папка
chmod 600 ~/.ssh/id_*              # SSH ключи
chmod 600 config-with-secrets      # Конфиги с секретами
find ~ -perm 777 2>/dev/null       # Найти опасные права
```

### Firewall

```bash
sudo ufw enable                    # Включить
sudo ufw default deny incoming     # Запретить входящее
sudo ufw allow 22/tcp              # Разрешить порт
sudo ufw status verbose            # Статус
```

### Сервисы

```bash
systemctl list-unit-files --type=service --state=enabled
sudo systemctl disable service     # Отключить при загрузке
sudo systemctl stop service        # Остановить сейчас
sudo ss -tlnp                      # Какие порты слушают
```

### Логирование

```bash
journalctl -u sshd -f              # Логи SSH live
journalctl -u sshd | grep Failed   # Неудачные попытки
journalctl -p err                  # Только ошибки
journalctl --vacuum-time=30d       # Хранить 30 дней
```

### Обновления

```bash
# Arch:
sudo pacman -Syu                   # Обновить всё
pacman -Q --upgrades               # Проверить обновления

# Ubuntu/Debian:
sudo apt update && sudo apt upgrade
apt list --upgradable
```

---

## Связанные заметки

### ← Перед этим (предусловие)
- [[linux-file-permissions]] - права доступа (основа безопасности)
- [[linux-users-groups]] - управление пользователями (sudoers)
- [[systemd-basics]] - управление сервисами (отключать ненужные)

### ↔ Параллельно (рядом)
- [[linux-networking]] - firewall это часть networking
- [[systemd-guide-extended]] - мониторинг через systemd таймеры

### 🔒 В этом файле (практическое применение)
- SSH Hardening
- Firewall
- Права доступа
- Отключение сервисов
- Мониторинг логов

### 📚 Главный индекс
- [[00-start-here-index]] - полная навигация по базе знаний

---

## Источники

- `man sshd_config` - конфигурация SSH сервера (ВСЕ)
- `man ssh-keygen` - создание SSH ключей (ВСЕ)
- `man chmod` - права доступа (ВСЕ)
- `man ufw` - простой firewall (ВСЕ)
- `man iptables` - продвинутый firewall (ВСЕ)
- `man sudoers` - конфигурация sudo (ВСЕ)
- `man journalctl` - просмотр логов (ВСЕ)
- Arch Wiki: Security
- Arch Wiki: SSH
- OWASP Top 10 (web security)
- CIS Benchmarks (Linux security)

---

Создано: 2026-01-04