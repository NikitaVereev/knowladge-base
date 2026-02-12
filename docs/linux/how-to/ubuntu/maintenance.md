---
title: "Обслуживание Ubuntu"
type: how-to
tags: [linux, ubuntu, maintenance, update, cleanup, lts, upgrade]
sources:
  original: "_inbox/01-linux/02-distro-specific/ubuntu-linux/04-maintenance.md"
related:
  - "[[linux/how-to/ubuntu/apt-and-ppa]]"
  - "[[linux/how-to/ubuntu/troubleshooting]]"
---

# Обслуживание Ubuntu

> **TL;DR:** `sudo apt update && sudo apt upgrade` регулярно.
> `autoremove` + `autoclean` для очистки. LTS → LTS upgrade раз в 2 года.

## Регулярные задачи

### Еженедельно

```bash
sudo apt update && sudo apt upgrade
sudo apt autoremove
df -h                              # проверить место на диске
```

### Ежемесячно

```bash
sudo apt clean                     # очистить весь кэш .deb
sudo apt autoremove --purge        # + удалить конфиги
sudo journalctl --vacuum-size=200M # ограничить журнал
sudo snap refresh                  # обновить snap-пакеты

# Проверить что автоматически запускается
systemctl list-unit-files --state=enabled
```

## Upgrade между версиями

```bash
# Проверить текущую версию
lsb_release -a

# Обновить до следующей LTS (например 22.04 → 24.04)
sudo apt update && sudo apt upgrade
sudo do-release-upgrade

# Только LTS → LTS (рекомендуется)
# Файл /etc/update-manager/release-upgrades:
# Prompt=lts
```

## Очистка диска

```bash
# Что занимает место
du -sh /var/cache/apt/archives/    # кэш apt
du -sh /var/log/                   # логи
du -sh /snap/                      # snap-пакеты
du -sh ~/                          # домашняя

# Очистить
sudo apt clean
sudo journalctl --vacuum-time=2weeks
sudo snap set system refresh.retain=2  # хранить 2 версии snap

# Старые ядра
sudo apt autoremove --purge
```

## Отключение ненужных сервисов

```bash
# Список включённых
systemctl list-unit-files --state=enabled

# Отключить (пример)
sudo systemctl disable bluetooth
sudo systemctl disable cups

# Анализ времени загрузки
systemd-analyze blame
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| Не обновляли год | `sudo apt update && sudo apt upgrade` может занять долго. Не прерывать! |
| `do-release-upgrade` не предлагает | Проверить `/etc/update-manager/release-upgrades`, `Prompt=lts` |
| snap занимает 10+ GB | `snap list --all` → `snap remove pkg --revision=N` для старых ревизий |
| Unattended-upgrades мешают | `sudo dpkg-reconfigure unattended-upgrades` |