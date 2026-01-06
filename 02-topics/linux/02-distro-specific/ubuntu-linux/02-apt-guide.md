---
created: 2026-01-06
updated: 2026-01-06
type: reference
---

# apt: управление пакетами

## 🎯 ЗОЛОТОЕ ПРАВИЛО

```bash
# ⚠️ ВСЕГДА делайте вместе!
sudo apt update && sudo apt upgrade
```

**Почему?** Если установить пакет без `update`, возникнут конфликты версий.

---

## 📚 ОСНОВНЫЕ КОМАНДЫ

### Обновление

```bash
sudo apt update               # обновить кэш пакетов
sudo apt upgrade              # обновить пакеты в рамках версии
sudo apt full-upgrade         # более агрессивное обновление
sudo apt autoremove           # удалить ненужные зависимости
```

### Установка

```bash
sudo apt install package      # один пакет
sudo apt install pkg1 pkg2    # несколько
sudo apt install ./package.deb # локальный .deb файл
```

### Удаление

```bash
sudo apt remove package       # удалить (конфиги остаются)
sudo apt purge package        # полное удаление + конфиги
sudo apt autoremove           # ненужные зависимости
```

### Поиск

```bash
apt search term               # поиск
apt search --names-only term  # только названия
apt show package              # информация о пакете
apt list --installed          # установленные
apt list --upgradable         # доступные обновления
```

---

## ⚙️ БЕЗОПАСНЫЕ ПРАКТИКИ

### Регулярные обновления

```bash
# Раз в неделю
sudo apt update && sudo apt upgrade -y

# Автоматические обновления (опционально)
sudo apt install unattended-upgrades
sudo systemctl enable unattended-upgrades
```

### Что НИКОГДА не делайте

```bash
❌ rm -rf /var/cache/apt        # потеряете откат
❌ sudo apt remove gcc          # нужен для компиляции
❌ sudo apt purge $(apt list --installed | cut -d/ -f1)  # удалит всё!
```

---

## 🚨 ПРОБЛЕМЫ И РЕШЕНИЯ

### "E: Could not get lock /var/lib/apt/lists/lock"

```bash
ps aux | grep -i apt           # ищем зависший процесс
sudo killall -9 apt apt-get    # если зависла
sudo rm /var/lib/apt/lists/lock
sudo apt update
```

### "Package has no installation candidate"

```bash
sudo apt update                # обновить список
apt search package             # поискать похожие
```

### "Unable to locate package"

```bash
# Пакета нет в репозиториях
apt search package

# Или может быть в PPA
sudo add-apt-repository ppa:user/ppa-name
sudo apt update
sudo apt install package
```

### "Unmet dependencies"

```bash
sudo apt install --fix-broken
sudo apt install -f            # fix
```

### Откатить пакет на старую версию

```bash
apt-cache policy package       # доступные версии
sudo apt install package=version
```

---

## 📋 ШПАРГАЛКА apt

```bash
# Основное
sudo apt update && sudo apt upgrade

# Установка
sudo apt install package

# Удаление
sudo apt remove package        # или purge
sudo apt autoremove

# Поиск
apt search term
apt show package

# Проблемы
sudo apt install --fix-broken
apt-cache policy package
```

---

## 🔗 ДАЛЬШЕ

[PPAs и репозитории](./03-ppa-guide.md)
