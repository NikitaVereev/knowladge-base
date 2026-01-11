# Ubuntu Maintenance

## Daily

```bash
sudo apt update && sudo apt upgrade   # обновляйте регулярно
df -h                                 # проверяйте место на диске
```

## Weekly

```bash
sudo apt autoremove                   # удалить неиспользуемые пакеты
sudo apt autoclean                    # удалить старые .deb файлы
```

## Monthly

```bash
# Полная очистка
sudo apt clean
sudo apt autoremove --purge

# Журналы
sudo journalctl --vacuum=100M

# Обновление драйверов
sudo ubuntu-drivers autoinstall
```

## Оптимизация

```bash
# Отключите ненужные сервисы
systemctl list-unit-files --state=enabled
sudo systemctl disable service

# анализ boot времени
systemd-analyze

# Очистите temp files
sudo rm -rf /tmp/*
```

## Key Takeaways

- Ubuntu требует обновлений (но реже чем Arch)
- Регулярная очистка экономит место
- LTS версии стабильнее

## Related

- [[02-apt-guide|APT Guide]]
- [[docs/linux/02-distro-specific/README|Ubuntu Index]]
