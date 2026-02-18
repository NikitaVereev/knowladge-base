---
title: "Linux"
type: index
tags: [linux, os, kernel, administration, arch, ubuntu]
---

# Linux

Операционная система с открытым исходным кодом. Ядро (kernel) управляет оборудованием, дистрибутив добавляет утилиты, пакетный менеджер и философию.

## Explanation (Концепции)

| Документ | Описание |
|----------|----------|
| [[linux/explanation/distributions]] | Три семейства: Debian (apt), Red Hat (dnf), Arch (pacman). LTS vs Rolling |
| [[linux/explanation/filesystem]] | FHS, «всё — файл», типы файлов, symlinks, скрытые файлы |
| [[linux/explanation/permissions-model]] | rwx, chmod, chown, SUID/SGID/sticky, sudo |
| [[linux/explanation/process-model]] | PID, состояния, сигналы, nice, foreground/background, daemons |
| [[linux/explanation/systemd]] | PID 1, units, targets, timers, journalctl, boot process |
| [[linux/explanation/shutdown]] | Процесс завершения: targets, SIGTERM → SIGKILL, sync, umount, graceful shutdown |
| [[linux/explanation/user-files]] | /etc/passwd, /etc/shadow, /etc/group, nsswitch.conf, формат полей, процесс логина |

## Tutorials (Пошаговые уроки)

| # | Документ | Что изучаем |
|---|----------|-------------|
| 01 | [[linux/tutorials/01-getting-started]] | Что такое Linux, архитектура, первые команды |
| 02 | [[linux/tutorials/02-package-management]] | apt, pacman, dnf — установка и управление пакетами |
| 03 | [[linux/tutorials/03-filesystem-and-commands]] | Навигация, файлы, grep, find, пайпы, перенаправления |
| 04 | [[linux/tutorials/04-shell-and-scripting]] | Bash: переменные, условия, циклы, функции, скрипты |
| 05 | [[linux/tutorials/05-networking-basics]] | IP, порты, DNS, ss, curl, ping, firewall основы |

## How-to (Практические руководства)

### Общие

| Документ | Описание |
|----------|----------|
| [[linux/how-to/manage-users]] | useradd, usermod, группы, sudo, passwd |
| [[linux/how-to/manage-services]] | systemctl start/stop/enable, создание юнитов |
| [[linux/how-to/manage-packages]] | Продвинутое: зависимости, orphans, pinning, откат |
| [[linux/how-to/configure-network]] | IP, DNS, NetworkManager, nmcli, static IP |
| [[linux/how-to/configure-firewall]] | ufw, iptables/nftables, открытие портов |
| [[linux/how-to/manage-disks]] | fdisk, mkfs, mount, fstab, LVM, SMART |
| [[linux/how-to/monitor-system]] | top/htop, free, df, du, iostat, journalctl |
| [[linux/how-to/harden-server]] | SSH, firewall, fail2ban, unattended-upgrades, audit |
| [[linux/how-to/write-bash-scripts]] | Структура скрипта, set -euo, аргументы, логирование |

### Arch Linux

| Документ | Описание |
|----------|----------|
| [[linux/how-to/arch/install]] | Установка с нуля: разметка, pacstrap, chroot, GRUB |
| [[linux/how-to/arch/pacman-and-aur]] | pacman, yay/paru, AUR, зеркала, кэш |
| [[linux/how-to/arch/maintenance]] | Обновления, .pacnew, orphans, очистка |
| [[linux/how-to/arch/troubleshooting]] | Не загружается, pacman сломан, драйверы, rescue |

### Ubuntu

| Документ | Описание |
|----------|----------|
| [[linux/how-to/ubuntu/install]] | Графический установщик, разметка, после установки |
| [[linux/how-to/ubuntu/apt-and-ppa]] | apt, PPA, snap, репозитории |
| [[linux/how-to/ubuntu/maintenance]] | Обновления, очистка, upgrade между LTS |
| [[linux/how-to/ubuntu/troubleshooting]] | Recovery mode, apt broken, драйверы, snap |

## Recipes (Готовые решения)

| Рецепт | Описание |
|--------|----------|
| [[linux/how-to/recipes/initial-server-setup]] | Первая настройка: user, SSH, firewall, timezone, обновления |
| [[linux/how-to/recipes/ssh-hardening]] | Ключи, отключение пароля, fail2ban, 2FA |
| [[linux/how-to/recipes/nginx-virtualhost]] | Nginx: virtualhost, reverse proxy, SSL |
| [[linux/how-to/recipes/backup-script]] | rsync/tar/dd, скрипт бэкапа, cron, 3-2-1 |

## Reference (Справочники)

| Документ | Описание |
|----------|----------|
| [[linux/reference/cheatsheet]] | Команды: навигация, файлы, процессы, сеть, диски |
| [[linux/reference/filesystem-hierarchy]] | FHS таблица: директории, важные файлы, спец. пути |
| [[linux/reference/permissions-table]] | chmod числа, rwx для файлов/директорий, спец. биты |
| [[linux/reference/systemd-reference]] | systemctl, journalctl, unit-файлы, targets, таймеры |

## Быстрый старт
```bash
# Узнать свой дистрибутив
cat /etc/os-release

# Обновить систему
sudo apt update && sudo apt upgrade      # Ubuntu/Debian
sudo pacman -Syu                          # Arch
sudo dnf upgrade                          # Fedora

# Базовые утилиты
sudo apt install git vim curl wget htop tmux tree   # Ubuntu
sudo pacman -S git vim curl wget htop tmux tree      # Arch
```

## Связанные разделы

- [[docker/index]] — Docker (контейнеризация)
- [[ansible/index]] — Ansible (автоматизация серверов)
- [[kubernetes/index]] — Kubernetes (оркестрация контейнеров)