# AUR Guide

## Overview

AUR (Arch User Repository) — репозиторий с пакетами созданными пользователями. Содержит программы которых нет в официальных репозиториях.

**Что вы узнаете:**
- Что такое AUR и как он работает
- Поиск пакетов в AUR
- Установка через yaourt или yay
- Проверка безопасности
- Создание собственного пакета (advanced)

## Prerequisites

- Установленный Arch Linux
- Знакомство с pacman
- Понимание что такое PKGBUILD

## Что такое AUR?

AUR содержит пакеты которые:
- Не часто используются (отсутствуют в офиц. репо)
- Новые или development версии
- Пользовательские модификации популярных пакетов

**Не все пакеты в AUR безопасны** — проверяйте PKGBUILD перед установкой!

## Поиск пакетов

### На сайте

Посетите https://aur.archlinux.org и ищите пакет.

### Через yay (рекомендуется)

```bash
yay -S yay                     # сначала установите yay сам
yay firefox-bin                # поиск пакета
yay -Ss "visual"               # поиск по названию/описанию
yay -Si package                # информация о пакете
```

### Через yaourt

```bash
yaourt -Ss package             # поиск
yaourt -Si package             # информация
```

## Установка YAY

**Способ 1: Из AUR (рекомендуется)**

```bash
cd /tmp
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

**Способ 2: Из официального репо Chaotic-AUR**

```bash
sudo pacman -S yay
```

## Использование YAY

```bash
yay                            # обновить всё (pacman + AUR)
yay -S package                 # установить пакет
yay -R package                 # удалить пакет
yay -Ss term                   # поиск
yay -c                         # очистить кэш
yay --devel -Su                # обновить development пакеты
```

## Установка вручную из AUR

```bash
# 1. Скачайте PKGBUILD
cd /tmp
git clone https://aur.archlinux.org/package.git
cd package

# 2. Проверьте PKGBUILD
cat PKGBUILD                   # прочитайте что там написано

# 3. Постройте пакет
makepkg -si                    # собрать и установить
# -s = install dependencies
# -i = install when done
```

## Безопасность AUR

### Проверки перед установкой

```bash
# 1. Посмотрите комментарии на AUR сайте
# 2. Проверьте PKGBUILD на очевидные проблемы
# 3. Проверьте что пакет не очень старый
# 4. Смотрите на количество скачиваний и звёзд
```

### Red flags

⚠️ Не устанавливайте если:
- Пакет использует `sudo` в install скрипте
- PKGBUILD содержит подозрительный код
- Пакет не обновлялся очень долго
- Пакет требует пароль без видимой причины

## Создание собственного пакета (Advanced)

### PKGBUILD структура

```bash
# PKGBUILD
pkgname=myapp
pkgver=1.0
pkgrel=1
pkgdesc="My awesome application"
arch=('x86_64')
url="https://github.com/user/myapp"
license=('GPL3')
depends=('bash')
makedepends=('gcc')
source=("$pkgname-$pkgver.tar.gz::https://github.com/user/myapp/archive/v$pkgver.tar.gz")
sha256sums=('HASH')

build() {
    cd "$srcdir/$pkgname-$pkgver"
    make
}

package() {
    cd "$srcdir/$pkgname-$pkgver"
    make DESTDIR="$pkgdir/" install
}
```

## Troubleshooting

### Ошибка "target not found"

```bash
yay -Syu                       # сначала обновте систему
yay -S package --rebuild       # пересоберите пакет
```

### Конфликт файлов

```bash
yay -S package --overwrite '*' # перезаписать файлы
```

### Пакет не находится

```bash
# Может быть переименован
yay -Ss похожее_имя            # поищите похожий пакет
# Или попросите в AUR forums
```

## Key Takeaways

- **AUR** = пользовательские пакеты (Arch User Repository)
- **yay** — лучший helper для работы с AUR
- **Проверяйте PKGBUILD** перед установкой
- **yay -S** = установить из pacman или AUR
- **Не все AUR пакеты безопасны** — будьте осторожны

## Related

- [[02-pacman-guide|Pacman Guide]] — официальные репозитории
- [[docs/01-linux/02-distro-specific/arch-linux/04-maintenance|Maintenance]] — обслуживание
- [[docs/01-linux/02-distro-specific/README|Arch Index]] — индекс

## See Also

- [AUR Official](https://aur.archlinux.org/)
- [AUR Wiki](https://wiki.archlinux.org/title/Arch_User_Repository)
- [yay](https://github.com/Jguer/yay)
