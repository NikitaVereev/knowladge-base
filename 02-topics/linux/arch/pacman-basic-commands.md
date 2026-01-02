---
created: 2026-01-01
tags: [arch-linux, pacman, tools]
type: reference
---

## Основная идея

pacman - менеджер пакетов Arch Linux. Управляет пакетами (программы, библиотеки, инструменты).

**Три источника пакетов:**
- `core` - базовые пакеты (ядро, glibc, systemd)
- `extra` - дополнительные (большинство программ)
- `community` - от сообщества
- `multilib` - 32-битные библиотеки (для 64-битной системы)
- AUR (Arch User Repository) - пользовательские пакеты (не официальные)

---

## ЧАСТЬ 1: Установка пакетов

### Простая установка

```bash
# Установить один пакет
sudo pacman -S package_name

# Например:
sudo pacman -S firefox
sudo pacman -S git
sudo pacman -S nginx

# Установить несколько пакетов сразу
sudo pacman -S package1 package2 package3

# Например:
sudo pacman -S git vim neofetch
```

### Что происходит при установке

```
sudo pacman -S firefox
  ↓
pacman скачивает информацию из репозиториев
  ↓
Проверяет зависимости (какие еще пакеты нужны)
  ↓
Показывает список для установки
  ↓
Asks: "Proceed with installation? [Y/n]"
  ↓
Скачивает все пакеты
  ↓
Проверяет контрольные суммы (безопасность)
  ↓
Распаковывает и устанавливает
```

### Установка конкретной версии

```bash
# Обычно не нужно, но если требуется старая версия:
sudo pacman -S package_name=1.2.3

# Посмотреть доступные версии
pacman -Sp package_name
```

### Установка без подтверждения

```bash
# Полезно в скриптах (не будет спрашивать "Proceed?")
sudo pacman -S --noconfirm package_name

# Все опции вместе:
sudo pacman -S --noconfirm --needed package1 package2
# --needed = не переустанавливать если уже установлен
```

---

## ЧАСТЬ 2: Обновление системы

### Стандартное обновление

```bash
# Обновить все пакеты (ОСНОВНАЯ КОМАНДА!)
sudo pacman -Syu

# Что это означает:
# -S = синхронизировать из репозиториев
# -y = обновить список репозиториев (refresh)
# -u = upgrade (обновить установленные пакеты)
```

**Полный процесс:**
```
sudo pacman -Syu
  ↓
Обновляет список репозиториев (мегабайты информации)
  ↓
Сравнивает: какие пакеты новые
  ↓
Показывает что обновится
  ↓
Asks: "Proceed with installation? [Y/n]"
  ↓
Скачивает, проверяет, устанавливает
```

### Принудительное обновление баз

```bash
# Обновить базы даже если уже свежие (редко нужно)
sudo pacman -Syyu

# Используйте если:
# - Зеркалу не доверяете (пересинхронизировать)
# - Проблемы с обновлением

# Два раза -y означает: обновить баз ПРИНУДИТЕЛЬНО
```

### Обновить БЕЗ устанавливания сразу (проверка)

```bash
# Посмотреть что обновится, но не устанавливать
sudo pacman -Syy    # Только обновить списки
# Потом
pacman -Qu          # Показать что можно обновить

# Или так (simpler):
pacman -Qu          # Показать обновления
```

### Что обновлять часто?

```bash
# КАЖДУЮ НЕДЕЛЮ минимум:
sudo pacman -Syu

# Особенно:
# - После перерыва (месяц без обновлений)
# - Перед установкой новых пакетов
# - Если есть security patche'с
```

**Важно:** Arch Linux требует частых обновлений (rolling release). Если не обновляться месяц - могут быть проблемы.

---

## ЧАСТЬ 3: Удаление пакетов

### Простое удаление

```bash
# Удалить пакет (оставить конфиги)
sudo pacman -R package_name

# Например:
sudo pacman -R firefox
```

### Удалить пакет + его зависимости

```bash
# Удалить пакет и все зависимости которые больше не используются
sudo pacman -Rs package_name

#例:
sudo pacman -Rs firefox

# Это умно удаляет: если зависимость используется другим пакетом - оставляет
# Если зависимость не используется ничем - удаляет
```

### Удалить пакет + зависимости + конфиги

```bash
# Полное удаление (включая конфигурационные файлы)
sudo pacman -Rns package_name

# ИСПОЛЬЗУЙТЕ ЭТО для полной очистки перед переустановкой
```

### Пример: полная переустановка

```bash
# 1. Удалить полностью
sudo pacman -Rns nginx

# 2. Переустановить
sudo pacman -S nginx

# 3. Новая конфигурация будет создана с default значениями
```

### Удалить несколько пакетов

```bash
# Один за другим
sudo pacman -Rns package1 package2 package3

# Или указать файл со списком пакетов
sudo pacman -R - < packages.txt
```

---

## ЧАСТЬ 4: Поиск пакетов

### Поиск в репозиториях

```bash
# Поиск пакета в официальных репозиториях
pacman -Ss keyword

# Примеры:
pacman -Ss python          # Все пакеты с "python"
pacman -Ss nginx           # Поиск nginx
pacman -Ss "web server"    # Поиск по фразе

# Результат показывает:
# repo/package_name version
#     Description
```

### Поиск установленных пакетов

```bash
# Поиск среди ТЕХ ЧТО УЖЕ УСТАНОВЛЕНО
pacman -Qs keyword

# Примеры:
pacman -Qs python          # Найти python из установленных
pacman -Qs gcc             # Найти gcc
```

### Информация о пакете

```bash
# Полная информация об установленном пакете
pacman -Qi package_name

# Показывает:
# - Версию
# - Размер
# - Зависимости
# - Конфликты
# - Когда установлен
# - Установщик

# Примеры:
pacman -Qi nginx
pacman -Qi python
```

### Информация о пакете в репозитории (еще не установлен)

```bash
# Информация о пакете из репозиториев
pacman -Si package_name

# Показывает почти то же что -Qi но для пакета в репо
pacman -Si firefox
```

### Посмотреть файлы пакета

```bash
# Какие файлы установил пакет (после установки)
pacman -Ql package_name

# Примеры:
pacman -Ql nginx       # Все файлы nginx
pacman -Ql vim | head  # Первые файлы vim

# Полезно для поиска конфигов:
pacman -Ql nginx | grep conf
```

### Найти какому пакету принадлежит файл

```bash
# Обратный поиск: какой пакет установил этот файл?
pacman -Qo /usr/bin/python3

# Результат:
# /usr/bin/python3 is owned by python 3.11.0-1

# Полезно когда:
# - Нашли странный файл
# - Не знаете какой пакет его установил
```

### Поиск орфан-пакетов (ненужные)

```bash
# Показать пакеты которые установлены но не требуются ничем
pacman -Qdt

# Это зависимости которые больше не нужны
# Можно удалить:
sudo pacman -Qdt | pacman -R -

# Или проще:
sudo pacman -Rns $(pacman -Qdtq)
```

---

## ЧАСТЬ 5: Управление кэшем

### Очистить кэш

```bash
# Удалить кэш скачанных пакетов (старые версии)
sudo pacman -Sc

# Это БЕЗОПАСНО - удаляет только старые версии которые не установлены
```

### Полная очистка кэша

```bash
# Удалить ВСЕ из кэша (включая текущие версии)
sudo pacman -Scc

# ОСТОРОЖНО! После этого переустановка будет скачивать заново
# Используйте если мало места на диске
```

### Посмотреть размер кэша

```bash
# Сколько займет кэш
du -sh /var/cache/pacman/pkg

# Пример вывода:
# 2.3G /var/cache/pacman/pkg

# После очистки будет меньше
```

---

## ЧАСТЬ 6: Решение проблем и ошибки

### "error: target not found"

```bash
# Ошибка: пакет не найден в репозиториях
sudo pacman -S nonexistent_package
# error: target not found: nonexistent_package

# Решение:
# 1. Проверить написание
pacman -Ss "search_keyword"

# 2. Пакет может быть в AUR (не официальный)
# Используйте yay вместо pacman
```

### "error: failed to prepare transaction (could not satisfy dependencies)"

```bash
# Конфликт зависимостей - pacman не может установить

# Решение:
# 1. Посмотреть что нужно пакету
pacman -Si problem_package

# 2. Часто помогает обновление
sudo pacman -Syu

# 3. Или установить нужную зависимость вручную
sudo pacman -S dependency_name
```

### "error: could not open file: /var/lib/pacman/local/..."

```bash
# База данных pacman повреждена

# Решение (в порядке):
# 1. Обновить базы
sudo pacman -Sy

# 2. Если не поможет - восстановить
sudo pacman-key --init
sudo pacman-key --populate archlinux

# 3. Если совсем плохо - удалить и пересоздать
sudo rm -r /var/lib/pacman/local/
sudo pacman -Sy
```

### Пакет установлен но команды не работают

```bash
# Пример: установили git но "git: command not found"

# Решение:
# 1. Проверить что установлено
pacman -Qi git

# 2. Посмотреть файлы пакета
pacman -Ql git | grep bin

# 3. Может PATH не обновлен - перезагрузить shell
source ~/.bashrc
# или просто открыть новый terminal
```

---

## ЧАСТЬ 7: Полезные советы и трюки

### Обновить И очистить кэш вместе

```bash
# Обновить + удалить старые версии пакетов из кэша
sudo pacman -Syu && sudo pacman -Sc

# Хорошая практика: делать раз в неделю
```

### Установить пакет с опциями

```bash
# Есть много пакетов которые требуют опций при установке
# Например MySQL может быть разными версиями

# Посмотреть опции:
pacman -Si package_name | grep -i optdepends
# или
pacman -Qi package_name | grep -i optdepends
```

### Список всех установленных пакетов

```bash
# Посмотреть все установленные пакеты
pacman -Q

# Сохранить список
pacman -Q > ~/packages.txt

# На новой системе переустановить все:
sudo pacman -S - < ~/packages.txt
```

### Найти большие пакеты (займет много места)

```bash
# Какие пакеты займут больше всего места?
pacman -Qi | grep Size | sort -h | tail -20

# Показывает 20 самых больших пакетов
```

### Автозаполнение команд

```bash
# Добавить в ~/.bashrc:
complete -W "$(pacman -Ssq)" pacman

# Теперь можно писать:
pacman -S fire<TAB>
# И будет:
pacman -S firefox
```

### Colorized output

```bash
# pacman может выводить цветным - включено по умолчанию
# Если нет - отредактировать /etc/pacman.conf:
sudo nano /etc/pacman.conf

# Найти строку:
#Color

# Раскомментировать:
Color
```

---

## ЧАСТЬ 8: Параллели с другими менеджерами пакетов

| Операция | Pacman | Ubuntu/Debian | Fedora/CentOS |
|----------|--------|---------------|--------------|
| Установка | `pacman -S` | `apt install` | `dnf install` |
| Обновление | `pacman -Syu` | `apt upgrade` | `dnf upgrade` |
| Удаление | `pacman -R` | `apt remove` | `dnf remove` |
| Поиск | `pacman -Ss` | `apt search` | `dnf search` |
| Очистка кэша | `pacman -Sc` | `apt autoclean` | `dnf clean` |
| Инфо о пакете | `pacman -Qi` | `apt show` | `dnf info` |

---

## ЧАСТЬ 9: Шпаргалка (быстрый справочник)

### Самые нужные команды

```bash
# УСТАНОВКА
sudo pacman -S package               # Установить
sudo pacman -S p1 p2 p3             # Несколько

# УДАЛЕНИЕ
sudo pacman -R package               # Удалить
sudo pacman -Rs package              # + зависимости
sudo pacman -Rns package             # + конфиги

# ОБНОВЛЕНИЕ
sudo pacman -Syu                     # Обновить всё
sudo pacman -Sy                      # Только списки

# ПОИСК
pacman -Ss keyword                   # В репо
pacman -Qs keyword                   # Установленные
pacman -Qi package                   # Инфо об установленном
pacman -Ql package                   # Файлы пакета

# КЭШ
sudo pacman -Sc                      # Очистить старое
sudo pacman -Scc                     # Полная очистка

# ПОЛЕЗНОЕ
pacman -Qdt                          # Орфан-пакеты
pacman -Qdtq                         # Только имена орфанов
pacman -Q                            # Все установленные
```

### Шаблон поиска и установки

```bash
# 1. Поиск что доступно
pacman -Ss nginx

# 2. Информация о пакете
pacman -Si nginx

# 3. Установка
sudo pacman -S nginx

# 4. Проверка что установлено
pacman -Qi nginx

# 5. Посмотреть файлы
pacman -Ql nginx

# 6. Удаление если не нужно
sudo pacman -Rns nginx
```

---

## ЧАСТЬ 10: Интеграция с AUR и yay

### Когда pacman недостаточно

```bash
# pacman работает только с официальными репозиториями
# Для пользовательских пакетов (AUR) используйте yay

# Установка yay:
sudo pacman -S git
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si

# После этого yay работает как pacman но + AUR пакеты
yay -S some_aur_package
```

### pacman vs yay

| Операция | Pacman | Yay |
|----------|--------|-----|
| Официальные пакеты | ✅ | ✅ |
| AUR пакеты | ❌ | ✅ |
| Обновление всего | `pacman -Syu` | `yay -Syu` |
| Поиск везде | ❌ | ✅ |

---

## Связанные заметки

- [[aur-yay-commands]] - AUR и yay менеджер
- [[arch-maintenance]] - обслуживание Arch Linux
- [[arch-troubleshooting]] - решение проблем Arch
- [[linux-package-managers]] - другие менеджеры пакетов

## Источники

- `man pacman` - полная документация pacman
- Arch Wiki: Pacman
- Arch Wiki: Official repositories
- ArchLinux.org документация

---
Создано: 2026-01-01 18:02
