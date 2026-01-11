# Ubuntu Troubleshooting

## System won't boot

```bash
# Загрузитесь с USB Live
# Зайдите в GRUB меню
# Выберите "Advanced options" -> предыдущее ядро

# Или переустановите GRUB
sudo mount /dev/sda2 /mnt
sudo grub-install --root-directory=/mnt /dev/sda
```

## No internet connection

```bash
# Проверьте интерфейсы
ip link show
sudo ip link set enp0s3 up

# Для Ethernet с DHCP
sudo dhclient enp0s3

# Для WiFi
nmcli device wifi list
nmcli device wifi connect SSID password PASSWORD
```

## Update breaks system

```bash
# Откатитесь на предыдущее ядро через GRUB
# Или переустановите broken пакет
sudo apt install --reinstall broken-package

# Или удалите проблемный пакет
sudo apt remove problem-package
```

## Snap package issues

```bash
# Отключите snap (если хотите)
sudo apt remove snapd

# Или переустановите snap
sudo apt install snapd
sudo snap refresh
```

## Dependencies broken

```bash
sudo apt install -f
sudo apt --fix-broken install
sudo apt --fix-missing install
```

## Disk full

```bash
du -sh /* | sort -rh
du -sh ~/.cache/*

sudo apt clean
sudo apt autoclean
rm -rf ~/.cache/
```

## Grub boot menu

```bash
# Если Grub не показывается
sudo update-grub

# Для переустановки Grub
sudo grub-install /dev/sda
sudo update-grub
```

## Display/Graphics issues

```bash
# Обновите видеодрайверы
sudo ubuntu-drivers autoinstall

# Или выберите вручную
ubuntu-drivers devices
sudo apt install nvidia-driver-535  # например
```

## Key Takeaways

- Ubuntu обычно стабильна (проблемы редки)
- Всегда есть сервис помощи в GUI
- Can boot from USB и исправить проблемы

## Related

- [[docs/01-linux/02-distro-specific/ubuntu-linux/01-installation|Installation]]
- [[docs/01-linux/02-distro-specific/README|Ubuntu Index]]

## See Also

- [Ubuntu Community Help](https://help.ubuntu.com/community/CommonProblems)
