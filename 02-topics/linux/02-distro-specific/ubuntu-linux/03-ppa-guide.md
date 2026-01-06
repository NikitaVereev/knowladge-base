---
created: 2026-01-06
updated: 2026-01-06
type: reference
---

# PPAs и репозитории Ubuntu

## 🎯 ЧТО ТАКОЕ PPA?

**PPA** (Personal Package Archive) — сторонний репозиторий на Launchpad.

- **Неофициальный** — не от Canonical
- **Сторонние пакеты** — которых нет в официальных репозиториях
- **БОЛЬШЕ РИСКОВ** — менее надежны

---

## ➕ ДОБАВЛЕНИЕ PPA

### Способ 1: add-apt-repository (рекомендуется)

```bash
sudo add-apt-repository ppa:user/ppa-name
sudo apt update
sudo apt install package
```

### Способ 2: Вручную

```bash
sudo nano /etc/apt/sources.list.d/ppa-name.list

# Добавить строку:
deb http://ppa.launchpad.net/user/ppa-name/ubuntu focal main

sudo apt update
```

---

## 📦 ПОПУЛЯРНЫЕ PPAs

```bash
# Python свежие версии
sudo add-apt-repository ppa:deadsnakes/ppa

# Mozilla Firefox
sudo add-apt-repository ppa:mozillateam/firefox-next

# NVIDIA драйверы
sudo add-apt-repository ppa:graphics-drivers/ppa

# LibreOffice
sudo add-apt-repository ppa:libreoffice/ppa

# VLC
sudo add-apt-repository ppa:videolan/stable-daily
```

---

## ❌ УДАЛЕНИЕ PPA

### Способ 1: add-apt-repository

```bash
sudo add-apt-repository --remove ppa:user/ppa-name
sudo apt update
```

### Способ 2: Удалить файл

```bash
sudo rm /etc/apt/sources.list.d/ppa-name.list
sudo apt update
```

### Откатить пакеты из PPA

```bash
sudo apt install ppa-purge
sudo ppa-purge ppa:user/ppa-name
# Система вернёт версии из official репозиториев
```

---

## 🔒 БЕЗОПАСНОСТЬ PPAs

### ПРАВИЛА (КРИТИЧНО)

1. ✅ Используйте PPAs **только от известных источников**
2. ✅ Ограничьте **количество PPAs** (не 50 штук!)
3. ✅ Проверяйте **дату последнего обновления**
4. ✅ Смотрите **комментарии на Launchpad**
5. ✅ **Регулярно обновляйтесь** — PPAs получают обновления быстро

### ЧЕРНЫЙ СПИСОК (НИКОГДА)

```bash
❌ PPA от неизвестных авторов
❌ PPAs с 0 votes и не обновлялись 2+ года
❌ PPAs требующие sudo без причины
❌ PPAs из неофициальных источников (не Launchpad)
```

---

## 🚨 ПРОБЛЕМЫ И РЕШЕНИЯ

### "Signed by an unknown key" ошибка

```bash
# PPA ключ не установлен
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys KEY_ID

# Или добавить ключ напрямую
wget -qO - https://ppa.launchpad.net/user/archive/ubuntu/KEY.asc | sudo apt-key add -

sudo apt update
```

### PPA конфликтует с другим пакетом

```bash
sudo add-apt-repository --remove ppa:user/ppa-name
sudo apt update
sudo ppa-purge ppa:user/ppa-name
```

### Пакет не обновляется из PPA

```bash
# Проверить приоритет
apt-cache policy package

# Может быть нужно увеличить приоритет
sudo nano /etc/apt/preferences.d/package

# Добавить:
Package: package-name
Pin: release o=LP-PPA-user-ppa-name
Pin-Priority: 999
```

---

## 📋 ШПАРГАЛКА PPAs

```bash
# Добавить PPA
sudo add-apt-repository ppa:user/ppa-name
sudo apt update

# Удалить PPA
sudo add-apt-repository --remove ppa:user/ppa-name
sudo apt update

# Откатить пакеты из PPA
sudo ppa-purge ppa:user/ppa-name

# Проверить версии
apt-cache policy package
```

---

## 🔗 ДАЛЬШЕ

[Обслуживание Ubuntu](./04-maintenance.md)
