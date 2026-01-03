---
created: 2026-01-03
tags: [linux, package-managers, system-administration, reference]
type: reference
---

# Linux Package Managers - сравнение менеджеров пакетов

## Основная идея

Каждый Linux дистрибутив имеет свой менеджер пакетов для установки, обновления и удаления ПО.

**Основные менеджеры:**
- **Arch/Manjaro** - pacman + yay (AUR)
- **Debian/Ubuntu** - apt
- **Fedora/CentOS/RHEL** - dnf (или yum)
- **openSUSE** - zypper
- **Alpine** - apk (lightweight)

**Концепции:**
- **Пакет** - готовое приложение/библиотека
- **Репозиторий** - хранилище пакетов
- **Зависимости** - какие пакеты требуются
- **Версионирование** - какая версия установлена

---

## ЧАСТЬ 1: Arch Linux - pacman & yay

### pacman (официальные пакеты)

```bash
# УСТАНОВКА
sudo pacman -S package_name
sudo pacman -S package1 package2 package3

# ОБНОВЛЕНИЕ
sudo pacman -Syu              # Обновить всё

# ПОИСК
pacman -Ss keyword            # Поиск в репо
pacman -Qs keyword            # Поиск установленных
pacman -Si package            # Инфо о пакете

# УДАЛЕНИЕ
sudo pacman -R package        # Удалить
sudo pacman -Rs package       # + зависимости
sudo pacman -Rns package      # + конфиги

# ИНФОРМАЦИЯ
pacman -Qi package            # Инфо об установленном
pacman -Ql package            # Файлы пакета
pacman -Qo /usr/bin/file      # Какому пакету принадлежит

# ОЧИСТКА
sudo pacman -Sc               # Удалить старые версии
pacman -Qdt                   # Показать orphan пакеты
```

### yay (AUR + официальные)

```bash
# УСТАНОВКА ИЗ AUR
yay package_name

# ОБНОВЛЕНИЕ (все + AUR)
yay -Syu

# ПОИСК везде
yay -Ss keyword

# УПРАВЛЕНИЕ AUR
yay -Qm                       # Список AUR пакетов
yay -Sua                      # Обновить только AUR

# ОЧИСТКА
yay -Yc                       # Удалить orphan'ы
```

### Особенности Arch

```
✅ Простой синтаксис
✅ Rolling release (всегда свежее)
✅ AUR (много пакетов)
❌ Требует частых обновлений
❌ Нужно следить за breaking changes
```

---

## ЧАСТЬ 2: Debian/Ubuntu - apt

### apt (основной менеджер)

```bash
# УСТАНОВКА
sudo apt install package_name
sudo apt install package1 package2

# ОБНОВЛЕНИЕ СПИСКА
sudo apt update              # Обновить списки репозиториев

# ОБНОВЛЕНИЕ ПАКЕТОВ
sudo apt upgrade             # Обновить установленные
sudo apt full-upgrade        # + удалить конфликтующие (осторожно!)
sudo apt dist-upgrade        # Для major версионных скачков

# ПОИСК
apt search keyword           # Поиск
apt show package             # Инфо о пакете

# УДАЛЕНИЕ
sudo apt remove package      # Удалить
sudo apt purge package       # + конфиги
sudo apt autoremove          # Удалить unused зависимости

# ИНФОРМАЦИЯ
apt list --installed         # Список установленных
apt-cache depends package    # Зависимости

# ОЧИСТКА
sudo apt clean               # Удалить кэш всех пакетов
sudo apt autoclean           # Удалить кэш старых версий
```

### apt vs apt-get

```bash
# apt-get (старый, низкоуровневый)
sudo apt-get install package

# apt (новый, удобнее)
sudo apt install package

# apt лучше для пользователей (красиво показывает прогресс)
# apt-get лучше для скриптов (стабильнее интерфейс)
```

### PPA (Personal Package Archives)

```bash
# Добавить PPA (для Ubuntu)
sudo add-apt-repository ppa:username/ppa-name
sudo apt update
sudo apt install package-from-ppa

# Удалить PPA
sudo add-apt-repository --remove ppa:username/ppa-name
sudo apt update

# Посмотреть добавленные PPA
grep -r "^deb" /etc/apt/sources.list.d/

# ОСТОРОЖНО с PPA!
# - Могут быть нестабильными
# - Авторы могут бросить проект
# - Меньше тестирования чем официальные
```

### Особенности Debian/Ubuntu

```
✅ Стабильные (LTS версии на 5 лет)
✅ Огромный репозиторий
✅ Много руководств и help
❌ Пакеты иногда старые
❌ Может потребоваться PPA для новых версий
```

---

## ЧАСТЬ 3: Fedora/RHEL/CentOS - dnf

### dnf (новый менеджер)

```bash
# УСТАНОВКА
sudo dnf install package_name
sudo dnf install package1 package2

# ОБНОВЛЕНИЕ
sudo dnf check-update        # Проверить доступные обновления
sudo dnf update              # Обновить пакеты
sudo dnf upgrade             # То же самое (alias)

# ПОИСК
dnf search keyword           # Поиск
dnf info package             # Инфо о пакете
dnf provides /path/to/file   # Какой пакет содержит файл

# УДАЛЕНИЕ
sudo dnf remove package      # Удалить
sudo dnf autoremove          # Удалить unused зависимости

# ИНФОРМАЦИЯ
dnf list installed           # Список установленных
dnf repolist                 # Список репозиториев
dnf deplist package          # Зависимости

# ОЧИСТКА
sudo dnf clean all           # Удалить кэш

# ГРУП-ПАКЕТЫ (наборы)
dnf groups list              # Показать доступные группы
sudo dnf groupinstall "Development Tools"  # Установить группу
sudo dnf groupremove "X Window System"     # Удалить группу
```

### yum (старый менеджер, заменен на dnf)

```bash
# yum ещё работает но dnf лучше
sudo yum install package

# yum теперь alias для dnf в новых версиях
yum --version  # Покажет что это dnf
```

### rpm (низкоуровневый менеджер)

```bash
# УСТАНОВИТЬ пакет напрямую (без зависимостей!)
sudo rpm -i package.rpm

# УДАЛИТЬ
sudo rpm -e package_name

# ИНФОРМАЦИЯ
rpm -qi package              # Инфо
rpm -ql package              # Список файлов
rpm -qf /path/to/file        # Который пакет

# ОСТОРОЖНО с rpm!
# rpm не разрешает зависимости автоматически
# Используйте dnf/yum для автоматического разрешения
```

### Особенности Fedora/RHEL

```
✅ Передовые технологии (новое железо, новые версии)
✅ Стабильный RHEL для production
✅ Хороший для серверов
❌ Меньше сторонних пакетов чем Debian
❌ Более частые breaking changes
```

---

## ЧАСТЬ 4: openSUSE - zypper

### zypper (менеджер openSUSE)

```bash
# УСТАНОВКА
sudo zypper install package_name
zypper in package_name       # Сокращение

# ОБНОВЛЕНИЕ
sudo zypper refresh          # Обновить списки
sudo zypper update           # Обновить пакеты
sudo zypper dist-upgrade     # Обновить систему целиком

# ПОИСК
zypper search keyword
zypper se keyword            # Сокращение
zypper info package

# УДАЛЕНИЕ
sudo zypper remove package
zypper rm package            # Сокращение

# ИНФОРМАЦИЯ
zypper packages --installed  # Установленные
zypper patches               # Доступные patche'ы
zypper repos                 # Репозитории

# ОЧИСТКА
sudo zypper clean

# PATTERN'Ы (аналог groupinstall)
zypper patterns              # Показать паттерны
sudo zypper install -t pattern kde  # Установить KDE pattern
```

### YaST (графический инструмент)

```bash
# YaST имеет графический интерфейс
sudo yast
# или
sudo yast2

# Очень мощный инструмент для конфигурации системы
```

### Особенности openSUSE

```
✅ Стабильный
✅ Хороший инструмент YaST
✅ Снимки (snapshots) по-умолчанию
✅ Transactional updates
❌ Меньше популярности чем Debian/Fedora
❌ Меньше community помощи
```

---

## ЧАСТЬ 5: Alpine Linux - apk

### apk (lightweight менеджер)

```bash
# УСТАНОВКА
apk add package_name
apk add package1 package2

# ОБНОВЛЕНИЕ ИНДЕКСА
apk update

# ОБНОВЛЕНИЕ ПАКЕТОВ
apk upgrade              # Обновить с текущей версией
apk upgrade --available  # Обновить до доступных версий

# ПОИСК
apk search keyword
apk search -d keyword    # С описанием

# УДАЛЕНИЕ
apk del package
apk del package1 package2

# ИНФОРМАЦИЯ
apk info package         # Инфо
apk info -L package      # Список файлов

# ОЧИСТКА
apk cache clean
```

### Особенности Alpine

```
✅ Очень лёгкий (5-10MB базовой системы)
✅ Быстрый
✅ Идеален для Docker контейнеров
❌ Минимум предустановленных инструментов
❌ Меньше пакетов чем другие дистрибутивы
❌ musl libc вместо glibc (совместимость может быть проблема)
```

---

## ЧАСТЬ 6: Таблица сравнения основных менеджеров

| Команда | Arch (pacman) | Debian (apt) | Fedora (dnf) | openSUSE (zypper) | Alpine (apk) |
|---------|--------------|--------------|--------------|-------------------|--------------|
| Установка | `pacman -S` | `apt install` | `dnf install` | `zypper in` | `apk add` |
| Обновление | `pacman -Syu` | `apt update && apt upgrade` | `dnf upgrade` | `zypper up` | `apk update && apk upgrade` |
| Поиск | `pacman -Ss` | `apt search` | `dnf search` | `zypper se` | `apk search` |
| Удаление | `pacman -R` | `apt remove` | `dnf remove` | `zypper rm` | `apk del` |
| Инфо | `pacman -Si` | `apt show` | `dnf info` | `zypper info` | `apk info` |
| Очистка | `pacman -Sc` | `apt clean` | `dnf clean` | `zypper clean` | `apk cache` |
| Orphan'ы | `pacman -Qdt` | `apt autoremove` | `dnf autoremove` | N/A | N/A |

---

## ЧАСТЬ 7: Выбор дистрибутива по менеджеру

### Хочу свежие пакеты и AUR
```
→ Arch Linux (rolling release)
→ Endeavour OS (Arch-based)
```

### Хочу стабильность и долгую поддержку
```
→ Debian (стабильная ветка на 5+ лет)
→ Ubuntu LTS (5 лет поддержки)
→ CentOS / RHEL (10+ лет поддержки)
```

### Хочу новые технологии и передовые features
```
→ Fedora (каждые 6 месяцев новые версии)
→ openSUSE Tumbleweed (rolling release с проверкой качества)
```

### Хочу контейнеры и minimalist систему
```
→ Alpine Linux (5-10MB)
→ Busybox (极минималистичный)
```

### Хочу графический интерфейс для управления
```
→ openSUSE + YaST
→ Ubuntu + GNOME Software
```

---

## ЧАСТЬ 8: Миграция между дистрибутивами

### От Debian к Arch

```bash
# Синтаксис отличается:
# Debian: sudo apt install package
# Arch:   sudo pacman -S package

# Команды:
Debian              Arch
--
apt search          pacman -Ss
apt install         pacman -S
apt remove          pacman -R
apt upgrade         pacman -Syu
apt autoremove      pacman -Qdt | pacman -R -
apt show            pacman -Si

# Иные концепции:
# - Debian имеет stable/testing/unstable
# - Arch это rolling release (всегда bleeding edge)
# - AUR в Arch даёт больше пакетов чем репо Debian
```

### От Ubuntu к Fedora

```bash
# Ubuntu:  apt
# Fedora:  dnf

# Примеры:
Ubuntu              Fedora
--
apt install         dnf install
apt update          dnf makecache
apt upgrade         dnf upgrade
apt autoremove      dnf autoremove
apt search          dnf search

# Различия:
# - Ubuntu LTS имеет 5 лет поддержки
# - Fedora имеет только 13 месяцев для каждой версии
# - Нужно обновляться на новую версию часто
```

### Сохранить список пакетов при миграции

```bash
# Экспортировать список пакетов
pacman -Q > packages.txt              # Arch
apt list --installed > packages.txt   # Debian
dnf list installed > packages.txt     # Fedora

# На новой системе переустановить
# Arch:
sudo pacman -S $(cat packages.txt | awk '{print $1}')

# Debian:
sudo apt install $(cat packages.txt | awk '{print $1}')

# Fedora:
sudo dnf install $(cat packages.txt | awk '{print $1}')

# ВНИМАНИЕ: не все пакеты могут быть в новом дистрибутиве!
# Могут быть другие имена пакетов
# Используйте как ориентир, не как точную копию
```

---

## ЧАСТЬ 9: Продвинутые техники

### Поиск какому пакету принадлежит команда

```bash
# Arch
pacman -Qo $(which command)           # какому пакету принадлежит

# Debian
dpkg -S $(which command)
# или
apt-file search command

# Fedora
dnf provides $(which command)

# openSUSE
zypper what-provides $(which command)
```

### Установить пакет из другой версии

```bash
# Debian
sudo apt install package/bullseye    # Установить из bullseye вместо текущей версии

# Fedora
sudo dnf install --releasever=37 package  # Из версии 37

# Arch
# Обычно невозможно (rolling release)
# Можно использовать downgrade из AUR
yay -S downgrade
sudo downgrade package
```

### Зафиксировать версию пакета (hold)

```bash
# Debian
sudo apt-mark hold package           # Не обновлять
sudo apt-mark unhold package         # Разрешить обновления

# Fedora (нет встроенного hold)
# Вариант: исключить из обновлений в /etc/dnf/dnf.conf
echo "exclude=package" | sudo tee -a /etc/dnf/dnf.conf
sudo dnf upgrade

# Arch
# Нет встроенного механизма
# Вариант: IgnorePkg в /etc/pacman.conf
echo "IgnorePkg = package" | sudo tee -a /etc/pacman.conf
```

### Скомпилировать из исходников если нет пакета

```bash
# Скачать исходники
wget https://example.com/package-1.0.tar.gz
tar xzf package-1.0.tar.gz
cd package-1.0

# Типичный процесс:
./configure      # Подготовка
make             # Компиляция
sudo make install  # Установка

# Или с CMake:
cmake .
make
sudo make install

# Или с Meson:
meson build
ninja -C build
sudo ninja -C build install

# ОСТОРОЖНО:
# - Может быть медленно (компиляция долгая)
# - Нет автоматических обновлений
# - Может конфликтовать с пакетами
# - Используйте только если нет пакета
```

---

## ЧАСТЬ 10: Шпаргалка для быстрого старта

### Arch Linux (pacman)

```bash
sudo pacman -Syu                    # Обновить систему
sudo pacman -S package_name         # Установить
sudo pacman -R package_name         # Удалить
pacman -Ss keyword                  # Поиск
pacman -Qi package                  # Инфо
yay -S aur_package                  # Из AUR
```

### Debian/Ubuntu (apt)

```bash
sudo apt update && sudo apt upgrade # Обновить систему
sudo apt install package_name       # Установить
sudo apt remove package_name        # Удалить
apt search keyword                  # Поиск
apt show package                    # Инфо
sudo add-apt-repository ppa:user/ppa  # Добавить PPA
```

### Fedora/CentOS (dnf)

```bash
sudo dnf upgrade                    # Обновить систему
sudo dnf install package_name       # Установить
sudo dnf remove package_name        # Удалить
dnf search keyword                  # Поиск
dnf info package                    # Инфо
sudo dnf groupinstall "Development Tools"  # Группа
```

### openSUSE (zypper)

```bash
sudo zypper up                      # Обновить систему
sudo zypper in package_name         # Установить
sudo zypper rm package_name         # Удалить
zypper se keyword                   # Поиск
zypper info package                 # Инфо
sudo yast                           # Графический интерфейс
```

### Alpine (apk)

```bash
apk update && apk upgrade           # Обновить систему
apk add package_name                # Установить
apk del package_name                # Удалить
apk search keyword                  # Поиск
apk info package                    # Инфо
```

---

## ЧАСТЬ 11: Проблемы и решения

### "Repository not found" или пакета нет в репозитории

```bash
# Arch
pacman -Ss package
# Если не найдено - может быть в AUR
yay -Ss package

# Debian
apt search package
# Если не найдено - добавить PPA
sudo add-apt-repository ppa:user/ppa
sudo apt update

# Fedora
dnf search package
# Или включить EPEL репо
sudo dnf install epel-release
```

### "Dependency conflict" - конфликтующие зависимости

```bash
# Arch
sudo pacman -Syu  # Обновление часто решает
# Или использовать yay для AUR пакетов

# Debian
sudo apt update
sudo apt upgrade
# Если ещё конфликт:
sudo apt install package/версия

# Fedora
sudo dnf install package
# dnf хороший в разрешении конфликтов
```

### Пакет частью broken (не запускается)

```bash
# Переустановить пакет полностью:

# Arch
sudo pacman -S --force package

# Debian
sudo apt install --reinstall package

# Fedora
sudo dnf reinstall package

# openSUSE
sudo zypper install --force package
```

---

## ЧАСТЬ 12: Интеграция с системой

### Автоматические обновления (unattended-upgrades)

```bash
# Debian/Ubuntu
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
# Включить в /etc/apt/apt.conf.d/50unattended-upgrades

# Fedora (dnf-automatic)
sudo dnf install dnf-automatic
sudo systemctl enable --now dnf-automatic.timer

# Arch
# Обычно не рекомендуется (rolling release требует внимания)
```

### Мониторинг обновлений

```bash
# Arch
sudo pacman -Qu                     # Показать обновления

# Debian
apt list --upgradable

# Fedora
sudo dnf check-update

# openSUSE
sudo zypper list-updates
```

---

## Связанные заметки

### ← Перед этим (предусловие)
- [[linux-system-basics]] - основы Linux

### → Выбор для Arch Linux
- [[pacman-complete-guide]] - менеджер пакетов Arch

### ↔ Выбор для других дистро
- Debian/Ubuntu в самом файле (apt)
- Fedora/RHEL в самом файле (dnf)
- openSUSE в самом файле (zypper)

### 📚 Главный индекс
- [[00-start-here-index]]


## Источники

- Arch Wiki: Pacman
- Debian Wiki: Apt
- Fedora Docs: dnf
- openSUSE Wiki: zypper
- Alpine Linux Wiki: apk

---

Создано: 2026-01-03
