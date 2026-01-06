---
created: 2026-01-06
updated: 2026-01-06
type: reference
---

# Обслуживание Ubuntu — АКТУАЛЬНЫЕ КОМАНДЫ

## 🔄 ОБНОВЛЕНИЕ СИСТЕМЫ

### Регулярные обновления (LTS версия)

```bash
sudo apt update && sudo apt upgrade
```

Выполняйте **раз в неделю** или раз в две недели.

### Обновление между версиями Ubuntu

**На новую LTS версию (стабильно):**
```bash
sudo apt update
sudo apt full-upgrade

# Обновить на новую версию
sudo do-release-upgrade

# Следовать инструкциям (Y/N)
# Перезагрузиться
sudo reboot
```

---

## 🧹 ОЧИСТКА СИСТЕМЫ (БЕЗОПАСНО)

### Удалить неиспользуемые пакеты

```bash
sudo apt autoremove
sudo apt autoremove -y           # без подтверждения
```

### Очистить кэш apt

```bash
# Удалить старые версии (БЕЗОПАСНО - можно откатить)
sudo apt clean

# Удалить весь кэш (⚠️ потом нельзя откатить!)
sudo apt autoclean
```

### Очистить логи systemd

```bash
# Размер логов
journalctl --disk-usage

# Удалить логи старше 3 дней (БЕЗОПАСНО)
sudo journalctl --vacuum-time=3d

# Удалить логи старше месяца
sudo journalctl --vacuum-time=30d
```

### Ограничить размер логов

```bash
sudo nano /etc/systemd/journald.conf

# Добавить:
SystemMaxUse=100M
MaxRetentionSec=4weeks

sudo systemctl restart systemd-journald
```

---

## 📊 МЕСТО НА ДИСКЕ

```bash
# По разделам
df -h

# Размер домашней папки
du -sh ~

# Размер системы
du -sh /usr
du -sh /var
du -sh /opt

# Самые большие файлы в home
find ~ -type f -size +100M -exec ls -lh {} \;

# Кэш браузеров
du -sh ~/.cache
```

### Очистить место вручную

```bash
# Кэш браузеров (если нужно место)
rm -rf ~/.cache/google-chrome/Default/Cache
rm -rf ~/.mozilla/firefox/*/cache

# Корзина
rm -rf ~/.local/share/Trash/*

# Старые загрузки (⚠️ проверьте содержимое!)
rm -rf ~/Downloads/*
```

---

## 🔧 СЕРВИСЫ И SYSTEMD

```bash
# Список сервисов при загрузке
systemctl list-unit-files --state=enabled

# Статус сервиса
sudo systemctl status service

# Управление
sudo systemctl start service
sudo systemctl stop service
sudo systemctl restart service
sudo systemctl enable service    # при загрузке
sudo systemctl disable service   # отключить

# Логи сервиса
sudo journalctl -u service -n 50
sudo journalctl -u service -f    # real-time
```

---

## 📊 МОНИТОРИНГ

```bash
# Время загрузки
systemd-analyze

# Медленные сервисы
systemd-analyze blame | head -10

# Процессы и память
top
htop                            # если установлен

# Память
free -h

# Версия Ubuntu
lsb_release -a
```

---

## 🚨 ПРОБЛЕМЫ И РЕШЕНИЯ

### Зависла установка/обновление

```bash
ps aux | grep apt
sudo killall -9 apt apt-get

sudo rm /var/lib/apt/lists/lock
sudo apt update
```

### После обновления нет загрузки

```bash
# С Live USB
sudo mount /dev/sda3 /mnt       # Linux раздел
sudo grub-install --root-directory=/mnt /dev/sda
sudo grub-mkconfig --output=/mnt/boot/grub/grub.cfg

sudo reboot
```

### Нет места на диске

```bash
sudo apt clean
sudo apt autoremove
sudo journalctl --vacuum-time=0   # удалить все логи!
```

### WiFi не работает после обновления

```bash
sudo systemctl restart NetworkManager
sudo systemctl restart systemd-networkd
sudo reboot
```

---

## ⚠️ ЧТО НЕ ДЕЛАЙТЕ

```bash
❌ rm -rf /var/cache/apt           # потеряете откат
❌ sudo apt remove gcc             # нужен для компиляции
❌ sudo journalctl --vacuum-time=0 # потеряете все логи!
❌ Отключать NetworkManager        # может быть критичен
```

---

## 📋 ШПАРГАЛКА

```bash
# Обновление
sudo apt update && sudo apt upgrade

# Очистка
sudo apt autoremove
sudo apt clean
sudo journalctl --vacuum-time=3d

# Информация
df -h
du -sh ~
lsb_release -a
systemd-analyze

# Обновление версии Ubuntu
sudo do-release-upgrade
```

---

## 🔗 ДАЛЬШЕ

[Решение проблем Ubuntu](./05-troubleshooting.md)
