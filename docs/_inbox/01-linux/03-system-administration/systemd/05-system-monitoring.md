# System Monitoring

## NAME

Мониторинг системы: ресурсы, производительность, логи и оповещения.

## SYNOPSIS

```bash
# Ресурсы в реальном времени
top                                 # процессы и ресурсы
htop                                # улучшенная версия top
free -h                             # память
df -h                               # дисковое пространство
iostat                              # I/O статистика

# Сеть
ifstat                              # статистика сетевых интерфейсов
nethogs                             # какой процесс потребляет сеть

# Логи
journalctl                          # все логи
journalctl -f                       # в реальном времени
journalctl -p err                   # только ошибки
```

## DESCRIPTION

Мониторинг системы необходим для выявления проблем до того как они станут критичными.

## CPU MONITORING

### top

```bash
top                                 # интерактивный мониторинг
top -p 1234                         # мониторить конкретный процесс
top -u username                     # мониторить процессы пользователя
```

### ps

```bash
ps aux                              # все процессы
ps aux | grep process_name          # найти конкретный процесс
ps aux --sort=-%cpu                 # сортировать по CPU
```

### Load Average

```bash
uptime                              # load average
cat /proc/loadavg                   # более подробно
w                                   # кто залогинен и load
```

## MEMORY MONITORING

### free

```bash
free -h                             # память в человеческом виде
free -m                             # в мегабайтах
free -g                             # в гигабайтах
watch -n 1 free -h                  # обновлять каждую секунду
```

### /proc/meminfo

```bash
cat /proc/meminfo                   # подробная информация
grep MemAvailable /proc/meminfo     # доступная память
```

## DISK MONITORING

### df (disk free)

```bash
df -h                               # доступное место
df -i                               # inode информация
df -H                               # десятичные единицы
```

### du (disk usage)

```bash
du -sh /home                        # размер директории
du -sh /home/*                      # размер подпапок
du -sh * | sort -rh                 # сортировать по размеру
```

### iostat

```bash
iostat                              # I/O статистика
iostat -x 1                         # расширенная, каждую секунду
iostat -m 1 5                       # в MB, 5 раз
```

## NETWORK MONITORING

### ifstat

```bash
ifstat                              # статистика интерфейсов
ifstat -i eth0                      # конкретный интерфейс
```

### nethogs

```bash
sudo nethogs                        # какой процесс потребляет сеть
sudo nethogs -d 1                   # обновлять каждую секунду
```

### netstat

```bash
netstat -tuln                       # открытые порты
netstat -tan | grep ESTABLISHED     # установленные соединения
```

## JOURNALCTL MONITORING

### Просмотр логов

```bash
journalctl                          # все логи
journalctl -f                       # в реальном времени (follow)
journalctl -n 50                    # последние 50 строк
journalctl -p err                   # только ошибки
journalctl -p 0..3                  # критичные ошибки
journalctl --since today            # за сегодня
journalctl --since "1 hour ago"     # за последний час
```

### Фильтрация

```bash
journalctl -u ssh                   # логи конкретного сервиса
journalctl _UID=1000                # логи конкретного пользователя
journalctl -k                       # логи ядра
journalctl /usr/bin/myapp           # логи конкретного приложения
```

## SYSTEM LOAD

### Когда высокая нагрузка

```bash
top                                 # какой процесс грузит CPU
iostat                              # I/O активность
netstat -tan                        # сетевая активность
journalctl -f -p err                # ошибки в логах
```

## PERFORMANCE ANALYSIS

### systemd-analyze

```bash
systemd-analyze                     # время загрузки
systemd-analyze blame               # какие сервисы долго загружаются
systemd-analyze critical-chain      # критический путь загрузки
systemd-analyze plot > boot.svg     # график загрузки
```

## ALERTING

### Email при проблемах

```bash
# При высокой нагрузке
if [ $(uptime | awk -F',' '{print $(NF-2)}' | awk '{print $1}') > 4 ]; then
    echo "High load detected" | mail -s "Alert" admin@example.com
fi
```

### Systemd service for monitoring

```ini
[Unit]
Description=System Monitor
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/monitor.sh
Restart=always

[Install]
WantedBy=multi-user.target
```

## KEY METRICS

- **Load Average** — среднее количество процессов в очереди
- **CPU Usage** — процент использования процессора
- **Memory Usage** — процент использования оперативной памяти
- **Disk I/O** — интенсивность чтения/записи на диск
- **Network** — пропускная способность сети
- **Process Count** — количество процессов в системе

## KEY TAKEAWAYS

- **Регулярно мониторить** — выявлять проблемы рано
- **Логи первыми** — всегда смотрите journalctl
- **top/htop** — для быстрой диагностики
- **Установить alerting** — быть уведомленным о проблемах
- **Историческая база** — сохранять метрики для анализа

## SEE ALSO

- [[01-what-is-systemd|What is systemd]]
- [[02-units-services|Units and Services]]
- [[03-package-management-advanced|Package Management]]
- [[04-backup-and-recovery|Backup and Recovery]]
- [[docs/_inbox/01-linux/03-system-administration/systemd/README|systemd README]]