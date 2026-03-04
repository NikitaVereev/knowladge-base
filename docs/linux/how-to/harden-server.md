---
title: "Защита сервера (Hardening)"
type: how-to
tags: [linux, security, hardening, ssh, firewall, updates, fail2ban, audit]
sources:
  book: "Внутреннее устройство Linux — Брайан Уорд, Глава 10.7"
related:
  - "[[linux/how-to/configure-firewall]]"
  - "[[linux/how-to/manage-users]]"
  - "[[linux/how-to/network-diagnostics]]"
  - "[[linux/how-to/recipes/ssh-hardening]]"
  - "[[linux/how-to/recipes/initial-server-setup]]"
---

# Защита сервера (Hardening)

> **TL;DR:** SSH по ключам + отключить root login + firewall + fail2ban + автообновления.
> Принцип минимальных привилегий. Обновляйте регулярно.

## Типы угроз

Понимание угроз помогает расставить приоритеты в защите:

- **Полная компрометация (full compromise)** — получение root-доступа. Злоумышленник эксплуатирует уязвимость в сервисе (переполнение буфера, незакрытый дефолтный доступ) или получает shell обычного пользователя и повышает привилегии через ошибки в setuid-программах. ASLR (рандомизация адресного пространства) смягчает buffer overflow, но не устраняет риск полностью
- **Отказ в обслуживании (DoS)** — лавина запросов или эксплуатация бага, вызывающего сбой. Труднее предотвратить, но легче обнаружить и отреагировать
- **Перехват паролей в открытом виде** — пароли, переданные без шифрования, легко перехватываются. Отсюда правило: **никогда не используйте FTP, telnet, rlogin** — только SSH и зашифрованные протоколы

> **Устаревшие опасные службы:** `ftpd` (уязвимости + пароли в открытом виде — замена: SSH/rsync), `telnetd` / `rlogind` / `rexecd` (сессии без шифрования — замена: SSH). Если нужна служба без встроенного шифрования — оберните её в **Stunnel** (www.stunnel.org).

## Принципы

- **Минимум сервисов** — злоумышленники не проникнут в то, что не запущено
- **Firewall на всё внутреннее** — даже если сервис нужен локально (rpcbind, D-Bus), он не должен быть виден извне
- **Обновляйте то, что торчит наружу** — SSH, веб-серверы, почта: следите за security-рассылками
- **LTS для серверов** — security-команды фокусируются на стабильных релизах, а не на Fedora Rawhide или Debian Unstable
- **Минимум учёток** — локальная учётная запись → повышение привилегий проще, чем удалённый взлом. Не давайте shell тем, кому он не нужен
- **Нет сомнительным бинарникам** — проверяйте источники пакетов

## Чеклист

### 1. Обновления

```bash
# Обновить всё
sudo apt update && sudo apt upgrade    # Ubuntu
sudo pacman -Syu                       # Arch

# Автоматические security-обновления (Ubuntu)
sudo apt install unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
```

### 2. SSH

```bash
# /etc/ssh/sshd_config
PermitRootLogin no                     # запретить root SSH
PasswordAuthentication no              # только ключи
PubkeyAuthentication yes
MaxAuthTries 3
LoginGraceTime 30
AllowUsers deploy admin                # только конкретные пользователи
Port 2222                              # нестандартный порт (опционально)
```

```bash
sudo systemctl restart sshd
```

Подробнее: [[linux/how-to/recipes/ssh-hardening]].

### 3. Firewall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp                 # или 2222/tcp если сменили
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

### 4. fail2ban (защита от brute-force)

```bash
sudo apt install fail2ban

# /etc/fail2ban/jail.local
cat << 'EOF' | sudo tee /etc/fail2ban/jail.local
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3

[sshd]
enabled = true
port = 22
EOF

sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd       # статистика
```

### 5. Минимум сервисов

```bash
# Что запущено?
systemctl list-units --type=service --state=running

# Отключить ненужное
sudo systemctl disable --now cups      # принтеры (если не нужны)
sudo systemctl disable --now avahi-daemon  # mDNS
sudo systemctl disable --now bluetooth
```

### 6. Пользователи

```bash
# Создать деплой-пользователя
sudo useradd -m -s /bin/bash -G sudo deploy
sudo passwd deploy

# Заблокировать root пароль (sudo вместо su)
sudo passwd -l root

# Проверить пустые пароли
sudo awk -F: '($2 == "") {print $1}' /etc/shadow
```

### 7. Права на файлы

```bash
# Критичные файлы
sudo chmod 600 /etc/shadow
sudo chmod 644 /etc/passwd
sudo chmod 700 /root
sudo chmod 700 /home/*/.ssh
sudo chmod 600 /home/*/.ssh/authorized_keys
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| Заблокировали себя SSH | Консоль VPS / recovery → исправить sshd_config |
| fail2ban заблокировал мой IP | `sudo fail2ban-client set sshd unbanip YOUR_IP` |
| `PasswordAuthentication no` до копирования ключа | Сначала `ssh-copy-id`, потом отключать пароли |
| Забыли `ufw allow ssh` перед `enable` | Консоль VPS → `ufw disable` |

## TLS: шифрование на транспортном уровне

Firewall и SSH защищают доступ, но если сервис передаёт данные в открытом виде — их можно перехватить на любом промежуточном узле. TLS (Transport Layer Security, ранее SSL) решает эту проблему: шифрует канал между клиентом и сервером с помощью криптографии с открытым ключом и сертификатов.

На практике TLS — это то, что превращает HTTP в HTTPS, SMTP в SMTPS, и так далее. Без TLS пароли, токены, данные сессий летят по сети как открытый текст.

**Когда TLS нужен вам:**

- Веб-сервер отдаёт что-то кроме статики → Let's Encrypt + certbot (бесплатно, автоматически)
- Приложение слушает TCP-порт, но не имеет встроенного TLS → **Stunnel** (www.stunnel.org) — обёртка, которая принимает TLS-соединения и проксирует их на незашифрованный порт. Хорошо работает со службами, активируемыми через socket-юниты systemd
- База данных доступна по сети → включить TLS в конфиге (PostgreSQL: `ssl = on`, MySQL: `require_secure_transport`)

```bash
# Проверить TLS-сертификат удалённого сервера
openssl s_client -connect example.com:443 -brief

# Проверить дату истечения
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
```

> **Принцип:** если сервис доступен по сети и передаёт что-либо чувствительное — он должен работать через TLS. Исключение: Unix domain sockets ([[linux/explanation/sockets]]) — данные не покидают машину.

## MAC: SELinux и AppArmor

Mandatory Access Control (MAC) — дополнительный уровень безопасности поверх стандартных rwx-прав. Даже если процесс запущен от root, MAC-политика может запретить ему доступ к определённым файлам или портам.

| | SELinux | AppArmor |
|---|---|---|
| Дистрибутивы | RHEL, Fedora, CentOS | Ubuntu, Debian, SUSE |
| Подход | Метки на файлах и процессах | Профили для программ (по путям) |
| Сложность | Высокая | Средняя |

```bash
# SELinux
getenforce                           # Enforcing / Permissive / Disabled
sudo setenforce 0                    # временно Permissive (для диагностики)
sudo ausearch -m avc -ts recent      # последние блокировки
sudo sealert -a /var/log/audit/audit.log  # человекочитаемый отчёт

# AppArmor
sudo aa-status                       # статус всех профилей
sudo aa-complain /usr/sbin/nginx     # режим обучения (логирует, не блокирует)
sudo aa-enforce /usr/sbin/nginx      # режим блокировки
cat /var/log/syslog | grep apparmor  # логи блокировок
```

> **Не отключайте MAC.** Если сервис не запускается из-за SELinux/AppArmor — переведите в permissive/complain, найдите причину в логах, исправьте политику. Отключение MAC — антипаттерн.

## Куда смотреть дальше

Безопасность — область, где устаревшая информация опаснее отсутствия информации. Три источника, которые стоит знать:

| Ресурс | Для чего | Когда обращаться |
|--------|---------|-----------------|
| **SANS Institute** (www.sans.org) | Обучение, еженедельная рассылка уязвимостей, шаблоны security-политик | Хотите системно разобраться в InfoSec или нужен шаблон политики для команды |
| **CERT/CC** (www.cert.org) | Базы критических уязвимостей, координация инцидентов | Сработал алерт, нужно понять масштаб проблемы и рекомендации вендора |
| **Insecure.org** (www.insecure.org) | Nmap, инструменты пентеста, конкретные эксплойты | Тестируете свой периметр, хотите понять что видит атакующий |

Для понимания TLS изнутри: *Serious Cryptography* (Jean-Philippe Aumasson, No Starch Press, 2017) — объясняет криптографические примитивы без академического формализма.