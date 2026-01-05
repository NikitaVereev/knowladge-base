---
created: 2026-01-05
updated: 2026-01-05
type: reference
---

# Процессы и сервисы

## Процессы

**Процесс** — запущенная программа с собственным PID (Process ID).

**Родительский процесс (PPID)** — процесс, который запустил данный процесс.

**Сигнал** — способ коммуникации между процессами и ядром.

---

## Просмотр процессов

### ps

```bash
ps                           # процессы текущего пользователя
ps aux                       # все процессы в системе
ps aux | grep firefox        # найти конкретный процесс
ps -ef --forest              # дерево процессов
ps -p 1234                   # информация о PID 1234
```

**Столбцы:**
- `USER` — пользователь, запустивший процесс
- `PID` — ID процесса
- `%CPU` — использование CPU
- `%MEM` — использование памяти
- `VSZ` — виртуальная память (KB)
- `RSS` — физическая память (KB)
- `STAT` — состояние (S=sleeping, R=running, Z=zombie)
- `COMMAND` — команда

### top

Интерактивное отслеживание процессов:

```bash
top                          # real-time мониторинг
top -u username              # процессы пользователя
top -p 1234                  # конкретный процесс
# Внутри: q выход, k убить процесс, h справка
```

### htop

Красивая версия top:

```bash
htop                         # вместо top
htop -u username             # процессы пользователя
htop -p 1234                 # конкретный процесс
# Виддит: стрелки навигация, K убить, Q выход
```

### pgrep

```bash
pgrep firefox                # PID процесса по названию
pgrep -l firefox             # с названием команды
pgrep -u username            # процессы пользователя
```

### pidof

```bash
pidof firefox                # PID по названию программы
pidof -s firefox             # только одного процесса
```

---

## Управление процессами

### Фоновые процессы

```bash
command &                    # запустить в фоне
jobs                         # список фоновых процессов
fg %1                        # вернуть на передний план
bg %1                        # продолжить в фоне
Ctrl+Z                       # приостановить процесс
```

### Сигналы

```bash
kill PID                     # SIGTERM (сигнал 15, мягкое завершение)
kill -9 PID                  # SIGKILL (сигнал 9, принудительное)
kill -l                      # список всех сигналов
kill -TERM PID               # то же что просто kill
kill -HUP PID                # перезагрузить (для daemon'ов)
killall firefox              # убить все процессы по названию
```

**Популярные сигналы:**
- `SIGHUP (1)` — hangup, перезагрузка
- `SIGINT (2)` — interrupt (Ctrl+C)
- `SIGTERM (15)` — terminate (нежное завершение)
- `SIGKILL (9)` — kill (принудительное завершение)
- `SIGSTOP (19)` — stop (приостановка)
- `SIGCONT (18)` — continue (возобновление)

### nice и renice

Приоритет процесса:

```bash
nice -n 10 command           # запустить с приоритетом -10..10
renice -n 5 -p PID           # изменить приоритет существующего
# Меньше = больший приоритет
```

---

## systemd и сервисы

**systemd** — система управления процессами и сервисами (init система).

**Сервис** — программа, работающая в фоне, управляется systemd.

### systemctl

```bash
systemctl list-units         # все сервисы
systemctl list-units --all   # включая неактивные
systemctl status service     # статус сервиса
systemctl start service      # запустить
systemctl stop service       # остановить
systemctl restart service    # перезагрузить
systemctl reload service     # перезагрузить конфиг без остановки
systemctl enable service     # автозагрузка при старте
systemctl disable service    # отключить автозагрузку
systemctl is-active service  # активен ли сервис
systemctl is-enabled service # будет ли загружаться при старте
```

**Примеры:**
```bash
sudo systemctl start nginx              # запустить nginx
sudo systemctl enable nginx             # при каждой загрузке
sudo systemctl status apache2           # статус apache2
sudo systemctl restart ssh              # перезагрузить ssh
```

### Файлы сервисов

```
/etc/systemd/system/        # сервисы системы
/usr/lib/systemd/system/    # встроенные сервисы
```

**Формат:**
```
[Unit]
Description=My Service
After=network.target

[Service]
Type=simple
User=username
ExecStart=/path/to/command
Restart=always

[Install]
WantedBy=multi-user.target
```

### journalctl

Логи systemd:

```bash
journalctl                   # все логи
journalctl -u service        # логи сервиса
journalctl -f                # real-time логи
journalctl -n 50             # последние 50 строк
journalctl --since "1 hour ago"  # последний час
journalctl -p err            # только ошибки
```

---

## Демоны (Daemons)

**Демон** — процесс, работающий в фоне без контроля пользователя.

**Типичные демоны:**
- `sshd` — SSH сервер
- `nginx`, `apache2` — веб-сервер
- `postgresql`, `mysql` — базы данных
- `rsyslog`, `journald` — логирование

---

## Проблемы и решения

### Процесс не отвечает

```bash
# Попробуйте мягкое завершение
kill -TERM PID
sleep 2

# Если не помогло, принудительно
kill -9 PID
```

### Найти процесс, занимающий дисковое пространство

```bash
ps aux | head -1 && ps aux | sort -k4 -rn | head -n 5  # по памяти
ps aux | head -1 && ps aux | sort -k3 -rn | head -n 5  # по CPU
```

### Сервис не запускается

```bash
sudo systemctl status service       # проверить ошибку
sudo journalctl -u service -n 50    # логи сервиса
sudo systemctl restart service      # попробовать перезагрузить
```

### Процесс создал много детей (fork bomb)

```bash
# Быстро выполните
killall -9 program
# или
kill -9 -1  # убить все процессы (осторожно!)
```

### Как узнать что запустить при загрузке

```bash
systemctl list-unit-files --state=enabled
# Это все сервисы, которые автозагруются
```

### Создать свой сервис

```bash
# 1. Создать файл сервиса
sudo nano /etc/systemd/system/myservice.service

# 2. Перезагрузить systemd конфиги
sudo systemctl daemon-reload

# 3. Включить и запустить
sudo systemctl enable --now myservice
```

---

## Шпаргалка

```bash
# Просмотр процессов
ps aux                       # все процессы
top                          # real-time мониторинг
pgrep firefox                # найти PID

# Управление
kill PID                     # завершить процесс
kill -9 PID                  # принудительно завершить
nice -n 10 command           # с приоритетом

# Сервисы
sudo systemctl start service # запустить сервис
sudo systemctl enable service # автозагрузка
sudo systemctl status service # статус
journalctl -u service -f     # логи в real-time
```

---

## Дальше

[Управление пакетами](./04-package-management-overview.md)
