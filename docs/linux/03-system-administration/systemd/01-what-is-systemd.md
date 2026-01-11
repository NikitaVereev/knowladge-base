# What is systemd

## NAME

systemd: современная система инициализации Linux. Управляет сервисами, демонами, загрузкой и логированием системы.

## SYNOPSIS

```bash
systemctl start service              # запустить сервис
systemctl enable service             # включить при загрузке
journalctl -u service                # просмотр логов сервиса
systemctl list-units --type=target    # список targets
```

## DESCRIPTION

systemd заменил старый SysVinit и стал стандартом в современном Linux (используется в 99% дистрибутивов).

### История

- **SysVinit** (1983) — shell скрипты, медленная загрузка
- **systemd** (2010) — конфиги, параллельная загрузка, быстро

### Архитектура

```
systemd (PID 1)
├── systemctl (управление сервисами)
├── journald (встроенное логирование)
├── Units (единицы управления)
│   ├── .service (сервисы)
│   ├── .target (состояния)
│   ├── .socket (сокеты)
│   ├── .mount (монтирование)
│   └── .timer (периодические задачи)
└── Targets (состояния системы)
    ├── multi-user.target
    ├── graphical.target
    ├── rescue.target
    └── emergency.target
```

## ADVANTAGES

| Аспект | systemd | SysVinit |
|--------|---------|----------|
| Загрузка | Параллельно (быстро) | Последовательно (медленно) |
| Язык | Конфиги .ini | Shell скрипты |
| Управление | systemctl | service |
| Логирование | journalctl (встроено) | syslog (отдельно) |
| Мониторинг | systemctl status | нет единого способа |

## UNITS

systemd управляет Units — конфигурационными файлами.

### Типы Units

```
.service   — сервисы (программы, демоны)
.target    — целевые состояния (группы units)
.socket    — сокеты (слушание портов)
.mount     — монтирование файловых систем
.timer     — периодические задачи
.device    — устройства
.path      — мониторинг файлов
.slice     — группировка ресурсов
```

## TARGETS

Targets определяют состояние системы. Замена runlevels.

### Основные Targets

```
multi-user.target    — обычное состояние (без графики)
graphical.target     — с графическим интерфейсом
rescue.target        — режим восстановления
emergency.target     — аварийный режим
poweroff.target      — выключение
reboot.target        — перезагрузка
```

## JOURNALD

Встроенная система логирования.

```bash
journalctl              # все логи
journalctl -u service   # логи сервиса
journalctl -f           # в реальном времени
journalctl -p err       # только ошибки
journalctl -n 50        # последние 50 строк
journalctl --since today # логи за сегодня
```

## BOOT PROCESS

```
1. Bootloader загружает ядро
2. Ядро запускает systemd (PID 1)
3. systemd читает конфиги из /etc/systemd/
4. systemd определяет default target
5. systemd запускает units из target
6. Units запускаются параллельно
7. Система готова (login или X11)
```

## FILESYSTEM

```bash
/etc/systemd/system/    — конфиги (редактировать)
/lib/systemd/system/    — установленные units
/run/systemd/           — runtime файлы
~/.config/systemd/      — пользовательские units
```

## KEY COMMANDS

```bash
systemctl start service           # запустить
systemctl stop service            # остановить
systemctl enable service          # включить при загрузке
systemctl disable service         # отключить при загрузке
systemctl status service          # статус
systemctl list-units              # все units
systemctl daemon-reload           # перезагрузить конфиги
journalctl -u service -f          # логи в реальном времени
systemctl list-units --type=target # все targets
systemctl get-default             # текущий target
```

## KEY TAKEAWAYS

- **systemd везде** — стандарт в Linux
- **Параллельный запуск** — быстрая загрузка
- **systemctl** — главная команда
- **journalctl** — просмотр логов
- **Units и Targets** — основные концепции
- **daemon-reload** — обязателен после изменений

## SEE ALSO

- [[02-units-services|Units and Services]]
- [[03-package-management-advanced|Package Management]]
- [[docs/linux/03-system-administration/systemd/README|systemd README]]