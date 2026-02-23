---
title: "SSH как сетевая служба Linux"
type: explanation
tags: [ssh, sshd, daemon, systemd, service, network, scp, sftp]
sources:
  docs: "https://man.openbsd.org/sshd.8"
  book: "Внутреннее устройство Linux — Брайан Уорд, Глава 10.3"
related:
  - "[[ssh/explanation/how-ssh-works]]"
  - "[[ssh/how-to/harden-server]]"
  - "[[ssh/how-to/configure-client]]"
  - "[[ssh/reference/cheatsheet]]"
---

# SSH как сетевая служба Linux

> **TL;DR:** sshd — это daemon, который слушает порт 22, форкается при каждом подключении и создаёт изолированный процесс для каждой сессии. В современных дистрибутивах управляется через systemd. Помимо интерактивных сессий, SSH — транспорт для scp, sftp, rsync и даже монтирования через sshfs.

## Зачем это знать

[[ssh/explanation/how-ssh-works]] объясняет **протокол** — как шифруется канал и проверяются ключи. Этот файл — про **службу**: как sshd работает на уровне ОС, как он запускается, форкается, взаимодействует с systemd и какие инструменты используют SSH как транспорт.

OpenSSH (www.openssh.com) — стандартная реализация SSH для Unix. Предустановлена практически во всех дистрибутивах Linux. Поддерживается **только версия 2** протокола (поддержка SSH v1 удалена из-за уязвимостей).

Понимание архитектуры службы нужно для:

- диагностики проблем с подключением (почему клиент не может достучаться)
- управления сервером (перезапуск без разрыва сессий, обновление ключей)
- понимания модели безопасности (privilege separation, chroot)

## Ключевые концепции

### Процессная модель sshd

sshd работает по классической Unix-модели **fork-per-connection**:

```
                       sshd (master)
                       PID 1234, root
                       слушает :22
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         sshd (child)  sshd (child)  sshd (child)
         PID 2001      PID 2045      PID 2089
         user: deploy  user: admin   user: deploy
         session 1     session 2     session 3
```

1. **Master-процесс** запускается от root, слушает TCP-порт (по умолчанию 22), записывает свой PID в `/var/run/sshd.pid`
2. При новом подключении master делает `fork()` — создаёт дочерний процесс
3. Дочерний процесс выполняет аутентификацию и, после успеха, сбрасывает привилегии до целевого пользователя
4. Master продолжает слушать — новые подключения не блокируются

Эта же модель fork-per-connection используется большинством сетевых демонов. Исключения: высоконагруженные серверы (nginx, Apache) создают пул рабочих процессов заранее, а UDP-серверы вообще не форкаются — они просто получают пакеты и реагируют.

```bash
# Увидеть процессы sshd в реальном времени
ps aux | grep sshd
# root   1234 ... /usr/sbin/sshd -D        ← master
# root   2001 ... sshd: deploy [priv]       ← privilege separation monitor
# deploy 2002 ... sshd: deploy@pts/0        ← пользовательская сессия
```

### Privilege Separation

Начиная с OpenSSH 7.5, privilege separation включена по умолчанию и неотключаема. Суть — **разделение на два процесса** после аутентификации:

```
sshd (child, root)          sshd (unprivileged)
├─ Владеет сокетом          ├─ Работает от имени пользователя
├─ Обрабатывает криптографию ├─ Обслуживает shell/команды
└─ Минимальный код           └─ Изолирован в chroot (до аутентификации)
```

Зачем: если в обработчике SSH-протокола есть уязвимость, атакующий окажется в урезанном chroot-окружении без привилегий, а не сразу с root-доступом.

### Host Keys — идентичность сервера

При первой установке sshd генерирует **host keys** — ключевые пары, идентифицирующие сервер:

```bash
ls -la /etc/ssh/ssh_host_*
# ssh_host_ed25519_key      ← приватный (600, root:root)
# ssh_host_ed25519_key.pub  ← публичный (644)
# ssh_host_rsa_key
# ssh_host_rsa_key.pub
# ssh_host_ecdsa_key
# ssh_host_ecdsa_key.pub
```

При подключении клиент сверяет host key сервера с записью в `~/.ssh/known_hosts`. Если ключ изменился (переустановка ОС, MITM-атака) — SSH блокирует подключение.

```bash
# Ручная генерация host keys (если отсутствуют)
sudo ssh-keygen -A

# Показать fingerprint сервера (для верификации)
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
# 256 SHA256:abc123... root@server (ED25519)
```

> **Важно:** При клонировании серверов (VM, образы) **обязательно перегенерируйте host keys**. Иначе все клоны будут иметь одинаковую идентичность — это серьёзная проблема безопасности. И наоборот: при **замене** сервера на новый (обновление железа, миграция) можно **импортировать** старые host keys на новую машину, чтобы пользователи не получали предупреждения о несовпадении ключей.

```bash
# Удалить старые + создать новые
sudo rm /etc/ssh/ssh_host_*
sudo ssh-keygen -A
sudo systemctl restart sshd
```

## Управление службой через systemd

### Стандартный юнит

В большинстве дистрибутивов sshd поставляется как systemd-сервис. Но есть нюансы с установкой:

- **Debian/Ubuntu:** SSH-сервер **не устанавливается** по умолчанию. При установке пакета `openssh-server` автоматически создаются host keys, запускается sервер и добавляется в автозагрузку
- **Fedora/RHEL:** sshd **установлен**, но **отключён**. Нужно вручную включить через `systemctl enable sshd`
- **Arch Linux:** аналогично Fedora — пакет есть, служба не активна

```bash
# Статус службы
systemctl status sshd          # RHEL/Fedora/Arch
systemctl status ssh            # Debian/Ubuntu (имя отличается!)

# Управление
sudo systemctl start sshd      # Запустить
sudo systemctl enable sshd     # Автозапуск при загрузке
sudo systemctl restart sshd    # Перезапуск (⚠️ разрывает активные сессии!)
sudo systemctl reload sshd     # Перечитать конфиг без разрыва сессий
```

> **Подводный камень:** `restart` убивает master-процесс → все дочерние процессы получают сигнал и завершаются. Если вы подключены по SSH и перезапускаете sshd — ваша текущая сессия выживет (она уже форкнута), но если что-то пошло не так с конфигом, **новые подключения не пройдут**, и вы потеряете доступ. Используйте `reload` (SIGHUP) когда возможно.

### Socket Activation

Некоторые дистрибутивы (Fedora, новые версии Arch) используют **socket activation** вместо постоянно работающего daemon:

```bash
# Проверить, используется ли socket activation
systemctl list-units | grep ssh
# sshd.socket   loaded active listening   OpenSSH Server Socket
# sshd@*.service loaded active running     OpenSSH per-connection server
```

При socket activation:

- systemd сам слушает порт 22 (`sshd.socket`)
- При входящем подключении systemd запускает `sshd@.service`
- Если подключений нет — процесс sshd не работает (экономия ресурсов)

> **Примечание:** socket activation — не лучшая идея для sshd, потому что серверу иногда приходится генерировать host keys при первом запуске, а этот процесс может занять значительное время. Автономный режим (standalone) распространён гораздо шире.

```bash
# Если используется socket activation
sudo systemctl enable --now sshd.socket    # включить
sudo systemctl disable sshd.service        # отключить standalone-режим
```

### Проверка конфигурации перед перезапуском

Конфигурация sshd хранится в `/etc/ssh/sshd_config` (не путать с клиентским `ssh_config`!). Формат — пары «ключ — значение». Закомментированные строки обычно показывают значения по умолчанию:

```bash
Port 22
#AddressFamily any
#ListenAddress 0.0.0.0
#HostKey /etc/ssh/ssh_host_ed25519_key
```

Полное описание параметров — в `man sshd_config(5)`. Практические рекомендации по hardening — в [[ssh/how-to/harden-server]].

```bash
# Тест синтаксиса — ВСЕГДА перед restart/reload
sudo sshd -t
# Если ошибок нет — молчание. Ошибки выводятся в stderr.

# Расширенный тест — показывает все применяемые настройки
sudo sshd -T | head -30
```

## SSH как транспорт: клиенты передачи файлов

SSH — это не только интерактивные сессии. Протокол используется как зашифрованный транспорт для нескольких инструментов:

### scp — простое копирование

```bash
# Локальный файл → сервер
scp deploy.tar.gz user@server:/opt/releases/

# Сервер → локально
scp user@server:/var/log/app.log ./

# Рекурсивно с сжатием
scp -rC user@server:/etc/nginx/ ./nginx-backup/
```

scp прост, но имеет ограничения: не умеет возобновлять передачу, не сохраняет символические ссылки. Для серьёзной работы — rsync.

> **Примечание:** начиная с OpenSSH 9.0, scp по умолчанию использует SFTP-протокол внутри, а не устаревший SCP/RCP.

### Передача через конвейер (pipe)

SSH умеет работать в конвейерах. Классический приём — копирование каталога через tar:

```bash
# Упаковать локальный каталог → передать по SSH → распаковать на сервере
tar zcvf - myproject/ | ssh user@server tar zxvf -

# То же в обратную сторону
ssh user@server "tar zcf - /var/log/app/" | tar zxf - -C ./backup/
```

Это быстрее scp для множества мелких файлов — один поток вместо тысяч отдельных операций.

### sftp — интерактивная передача

```bash
sftp user@server
# sftp> ls
# sftp> cd /opt/releases
# sftp> put deploy.tar.gz
# sftp> get config.yml
# sftp> exit
```

sftp — полноценный файловый протокол поверх SSH. В отличие от FTP, не требует отдельного порта для данных, работает через единственное SSH-подключение.

### rsync — синхронизация через SSH

```bash
# Синхронизация каталога (по умолчанию rsync использует SSH)
rsync -avz --progress ./dist/ user@server:/var/www/app/

# Обратная синхронизация (сервер → локально)
rsync -avz user@server:/var/log/app/ ./logs/

# С указанием SSH-порта и ключа
rsync -avz -e "ssh -p 2222 -i ~/.ssh/deploy_key" ./dist/ user@server:/var/www/
```

rsync — стандарт для деплоя и бэкапов: передаёт только изменившиеся части файлов, поддерживает возобновление, сохраняет permissions.

### sshfs — монтирование удалённой ФС

```bash
# Установка
sudo apt install sshfs         # Debian/Ubuntu
sudo pacman -S sshfs           # Arch

# Монтирование
mkdir ~/remote
sshfs user@server:/var/www ~/remote

# Работать как с локальной папкой
ls ~/remote
vim ~/remote/index.html

# Отмонтировать
fusermount -u ~/remote
```

sshfs работает через FUSE — удобно для разработки, но не подходит для production: каждая файловая операция идёт через сеть.

## SSH-клиенты на не-Unix платформах

SSH — межплатформенный протокол. Основные клиенты:

| Платформа | Клиент | Передача файлов |
|-----------|--------|----------------|
| **Windows 10+** | Встроенный OpenSSH (`ssh.exe`) | `scp.exe`, `sftp.exe` |
| **Windows (GUI)** | PuTTY | WinSCP, FileZilla |
| **macOS** | Встроенный OpenSSH | `scp`, `sftp`, Cyberduck |
| **Android** | Termux, JuiceSSH | Termux (`scp`, `rsync`) |
| **iOS** | Blink Shell, Termius | Встроенные SFTP-клиенты |

> Windows 10 (начиная с 2018) включает полноценный OpenSSH-клиент и даже сервер как Optional Feature. PuTTY по-прежнему популярен, но для работы из командной строки нативный `ssh.exe` удобнее: он совместим с `~/.ssh/config` и тем же форматом ключей.

## Подводные камни

> **Важно:** При обновлении OpenSSH на сервере проверяйте Release Notes. Новые версии могут отключать устаревшие алгоритмы, и старые клиенты перестанут подключаться. Симптом: `no matching key exchange method found`.

| Ситуация | Причина | Решение |
|----------|---------|---------|
| `Connection refused` | sshd не запущен или порт заблокирован | `systemctl status sshd` + проверить firewall |
| `Connection timed out` | Firewall дропает пакеты (не reject, а drop) | `telnet server 22` для диагностики |
| `Host key verification failed` | Ключ сервера изменился | `ssh-keygen -R server-ip` если это ожидаемо |

При несовпадении host key SSH выдаёт характерное предупреждение:

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Offending key in /home/user/.ssh/known_hosts:12
```

Чаще всего это означает, что администратор переустановил ОС или обновил оборудование. Но **всегда проверяйте** с администратором — это может быть и MITM-атака. Номер строки в `known_hosts` указан в сообщении — удалите её или используйте `ssh-keygen -R host`.

| Ситуация | Причина | Решение |
|----------|---------|---------|
| sshd не стартует после правки конфига | Ошибка синтаксиса | `sshd -t` **до** рестарта |
| `no matching key exchange` | Сервер обновился, старый алгоритм отключён | Обновить клиент или добавить алгоритм в конфиг |
| Сессия зависает при закрытии | Фоновый процесс держит PTY | `ssh -O exit server` или `~.` (escape sequence) |

## Связанные материалы

- [[ssh/explanation/how-ssh-works]] — протокол SSH: шифрование, аутентификация, алгоритмы
- [[ssh/how-to/harden-server]] — практический hardening sshd_config + fail2ban
- [[ssh/how-to/configure-client]] — `~/.ssh/config`: алиасы, jump hosts, multiplexing
- [[ssh/reference/cheatsheet]] — все команды на одной странице
