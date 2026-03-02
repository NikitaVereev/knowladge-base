---
title: "Окружение shell: переменные, PATH, dotfiles"
type: explanation
tags: [bash, shell, environment, path, dotfiles, bashrc, profile, readline, variables]
sources:
  book: "Внутреннее устройство Linux — Брайан Уорд, Главы 2–3"
  docs: "https://www.gnu.org/software/bash/manual/bash.html#Bash-Startup-Files"
related:
  - "[[bash/explanation/shell-language]]"
  - "[[bash/explanation/shell-internals]]"
  - "[[bash/how-to/write-scripts]]"
  - "[[linux/explanation/permissions-model]]"
  - "[[linux/reference/cheatsheet]]"
---

# Окружение shell: переменные, PATH, dotfiles

> **TL;DR:** Shell хранит два типа переменных: shell-переменные (локальные) и переменные окружения (наследуются дочерними процессами, `export`). `PATH` определяет где искать команды. Файлы инициализации: `.bash_profile` (login shell) и `.bashrc` (interactive non-login). Readline управляет редактированием строки (Ctrl+A/E/R/W).

## Зачем это знать

- `command not found` — обычно проблема с PATH
- Переменная задана, но не видна в скрипте — не сделан `export`
- Настройки не применяются — путаница между `.bashrc` и `.bash_profile`
- Эффективная работа в терминале — горячие клавиши readline

## Переменные оболочки vs переменные окружения

### Переменная оболочки (shell variable)

Существует только в текущем экземпляре shell. Дочерние процессы её **не видят**.

```bash
MY_VAR="hello"
echo $MY_VAR          # hello
bash -c 'echo $MY_VAR'  # (пусто) — дочерний bash не видит
```

### Переменная окружения (environment variable)

Передаётся всем дочерним процессам через механизм `fork()`. Создаётся через `export`.

```bash
export MY_VAR="hello"
bash -c 'echo $MY_VAR'  # hello — дочерний процесс видит

# Или в одну строку
export DB_HOST="localhost"
```

> Направление только «вниз»: дочерний процесс получает **копию** окружения. Изменение переменной в дочернем процессе не влияет на родительский.

### Временное окружение для одной команды

```bash
# Переменная существует только для этой команды
LANG=C sort file.txt
TZ=UTC date
DB_HOST=prod DB_PORT=5432 python app.py
```

### Просмотр переменных

```bash
env                     # все переменные окружения
printenv HOME           # конкретная переменная окружения
echo $HOME              # значение (shell + env)
set                     # все переменные (shell + env + функции)
declare -p VAR          # тип и значение переменной
```

### Стандартные переменные окружения

| Переменная | Назначение | Пример |
|---|---|---|
| `PATH` | Каталоги поиска команд | `/usr/local/bin:/usr/bin:/bin` |
| `HOME` | Домашняя директория | `/home/john` |
| `USER` | Имя текущего пользователя | `john` |
| `SHELL` | Оболочка пользователя | `/bin/bash` |
| `PWD` | Текущая директория | `/home/john/project` |
| `OLDPWD` | Предыдущая директория (`cd -`) | `/tmp` |
| `LANG` / `LC_*` | Локаль (язык, кодировка) | `en_US.UTF-8` |
| `TERM` | Тип терминала | `xterm-256color` |
| `EDITOR` | Редактор по умолчанию | `vim` |
| `PAGER` | Программа для просмотра (`less`, `more`) | `less` |
| `PS1` | Формат приглашения (prompt) | `\u@\h:\w\$` |
| `HISTSIZE` | Размер истории команд | `1000` |

## PATH — поиск команд

Когда вы вводите команду (например `ls`), shell ищет исполняемый файл по каталогам, перечисленным в `PATH`, слева направо. Первое совпадение — выполняется.

```bash
echo $PATH
# /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Где именно находится команда
which python3           # /usr/bin/python3
type python3            # python3 is /usr/bin/python3
which -a python3        # все совпадения во всех каталогах PATH
```

### Изменение PATH

```bash
# Добавить каталог в конец (текущая сессия)
export PATH="$PATH:/opt/myapp/bin"

# Добавить в начало (приоритет выше)
export PATH="/opt/myapp/bin:$PATH"

# Постоянно — добавить в ~/.bashrc или ~/.profile
echo 'export PATH="$PATH:$HOME/.local/bin"' >> ~/.bashrc
```

### Почему «command not found»

1. Программа не установлена → установить
2. Программа установлена не в каталог из PATH → добавить каталог в PATH
3. PATH перезаписан (потерян) → проверить dotfiles на ошибки
4. Нет прав на исполнение → `chmod +x`
5. Скрипт без `./` → shell не ищет в текущей директории (`.` обычно нет в PATH из соображений безопасности)

## Dotfiles — файлы инициализации

### Файлы инициализации bash

Bash выполняет разные файлы в зависимости от типа запуска:

```
                    ┌──────────────────┐
                    │  Login shell?    │
                    └────────┬─────────┘
                        ╱          ╲
                      Да            Нет
                      ↓              ↓
              /etc/profile      Interactive?
              ~/.bash_profile       ╱    ╲
              (~/.bash_login)     Да      Нет
              (~/.profile)        ↓       ↓
                              ~/.bashrc  BASH_ENV
                                         (если задан)
```

| Файл | Когда читается | Что в нём |
|---|---|---|
| `/etc/profile` | Login shell (все пользователи) | Системные переменные, PATH |
| `~/.bash_profile` | Login shell (конкретный пользователь) | Личные переменные, вызов `.bashrc` |
| `~/.bashrc` | Interactive non-login shell | Алиасы, функции, prompt, настройки |
| `~/.profile` | Login shell (если нет `.bash_profile`) | Совместимость с sh |

**Login shell** — при входе в систему (SSH, `su -`, `login`, TTY). Определяется по `-` в начале `argv[0]` или флагу `--login`.

**Non-login interactive** — открытие нового терминала в GUI, запуск `bash` из другого shell.

> Типичный паттерн: `.bash_profile` делает `source ~/.bashrc`, чтобы настройки были доступны в обоих случаях.

```bash
# ~/.bash_profile (типичное содержимое)
# Загрузить .bashrc если есть
[ -f ~/.bashrc ] && source ~/.bashrc

# Переменные окружения (только для login)
export EDITOR=vim
export PATH="$PATH:$HOME/bin"
```

```bash
# ~/.bashrc (типичное содержимое)
# Алиасы
alias ll='ls -la'
alias gs='git status'

# Prompt
PS1='\[\e[32m\]\u@\h\[\e[0m\]:\[\e[34m\]\w\[\e[0m\]\$ '

# Функции
mkcd() { mkdir -p "$1" && cd "$1"; }
```

### Glob и dotfiles

По умолчанию glob-паттерны (`*`, `?`) **не** захватывают файлы, начинающиеся с точки:

```bash
ls *           # НЕ покажет .bashrc, .ssh/
ls .*          # только dotfiles
ls -a          # всё
```

### Распространённые dotfiles

| Файл | Программа |
|---|---|
| `~/.bashrc`, `~/.bash_profile` | bash |
| `~/.ssh/config`, `~/.ssh/authorized_keys` | SSH |
| `~/.gitconfig` | git |
| `~/.vimrc` | vim |
| `~/.config/` | XDG-совместимые приложения |
| `~/.local/bin/` | Пользовательские скрипты (добавить в PATH) |
| `~/.inputrc` | readline (настройка ввода) |

## Readline: редактирование в командной строке

Библиотека readline используется bash (и многими другими программами) для обработки ввода. Горячие клавиши работают «из коробки».

### Навигация

| Клавиша | Действие |
|---|---|
| `Ctrl+A` | В начало строки |
| `Ctrl+E` | В конец строки |
| `Ctrl+F` / `→` | Вперёд на символ |
| `Ctrl+B` / `←` | Назад на символ |
| `Alt+F` | Вперёд на слово |
| `Alt+B` | Назад на слово |

### Редактирование

| Клавиша | Действие |
|---|---|
| `Ctrl+U` | Удалить от курсора до начала строки |
| `Ctrl+K` | Удалить от курсора до конца строки |
| `Ctrl+W` | Удалить слово перед курсором |
| `Alt+D` | Удалить слово после курсора |
| `Ctrl+Y` | Вставить последнее удалённое (yank) |
| `Ctrl+T` | Поменять местами два символа |
| `Ctrl+L` | Очистить экран (как `clear`) |

### История

| Клавиша | Действие |
|---|---|
| `Ctrl+R` | Обратный поиск по истории (начните вводить) |
| `Ctrl+G` | Отмена поиска |
| `Ctrl+P` / `↑` | Предыдущая команда |
| `Ctrl+N` / `↓` | Следующая команда |
| `!!` | Повторить последнюю команду |
| `!$` | Последний аргумент предыдущей команды |

## Специальные символы shell

| Символ | Назначение | Пример |
|---|---|---|
| `*` | Любые символы (glob) | `ls *.log` |
| `?` | Один любой символ | `ls file?.txt` |
| `[abc]` | Один из символов | `ls file[123].txt` |
| `{a,b}` | Перебор вариантов (brace expansion) | `cp file.{txt,bak}` |
| `~` | Домашняя директория | `cd ~/projects` |
| `$` | Подстановка переменной | `echo $HOME` |
| `$(...)` | Подстановка команды | `echo $(date)` |

Подробнее о кавычках и подстановках: [[bash/explanation/shell-language]].

## Текстовые редакторы

### nano — простой редактор

```bash
nano file.txt
# Ctrl+O — сохранить
# Ctrl+X — выход
# Ctrl+K — вырезать строку
# Ctrl+U — вставить
# Ctrl+W — поиск
```

### vi/vim — мощный, но с порогом входа

```bash
vim file.txt
# Два основных режима:
#   Normal (по умолчанию) — навигация и команды
#   Insert (i/a/o)        — ввод текста
#   Esc                   — вернуться в Normal

# Normal: :w — сохранить, :q — выход, :wq — оба, :q! — выйти без сохранения
# Normal: dd — удалить строку, yy — копировать, p — вставить, u — undo
# Normal: /pattern — поиск, n — следующее совпадение
```

## Получение помощи

### man — страницы руководства

```bash
man ls                  # документация по ls
man 5 passwd            # секция 5 (формат файла), не команда passwd
man -k keyword          # поиск по описаниям (apropos)
```

Секции man:

| Секция | Содержимое | Пример |
|---|---|---|
| 1 | Пользовательские команды | `man 1 passwd` (команда) |
| 2 | Системные вызовы | `man 2 open` |
| 3 | Библиотечные функции | `man 3 printf` |
| 5 | Форматы файлов | `man 5 passwd` (формат /etc/passwd) |
| 8 | Системное администрирование | `man 8 mount` |

### Другие источники

```bash
command --help          # краткая справка (большинство команд)
info command            # подробнее, чем man (иногда)
help builtin            # встроенные команды bash (cd, export, alias...)
```

## Смена пароля и оболочки

```bash
passwd                  # сменить свой пароль
chsh -s /bin/zsh        # сменить оболочку (из /etc/shells)
chsh -l                 # список допустимых оболочек
```

## Подводные камни

| Ситуация | Совет |
|----------|-------|
| Переменная не видна в скрипте | `export VAR` — без export переменная не наследуется |
| Настройки из `.bashrc` не применяются при SSH | SSH = login shell → читает `.bash_profile`. Добавить `source ~/.bashrc` в `.bash_profile` |
| `PATH` пуст/сломан | `export PATH="/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"` — восстановить вручную |
| `./script.sh` работает, `script.sh` нет | `.` (текущий каталог) не в PATH. Это нормально и безопасно |
| Alias не работает в скрипте | Алиасы по умолчанию отключены в неинтерактивном режиме. Используйте функции |

## Связанные материалы

- [[bash/explanation/shell-language]] — кавычки, подстановки, коды возврата
- [[bash/explanation/shell-internals]] — подоболочки, sourcing, exec
- [[bash/how-to/write-scripts]] — написание скриптов
- [[linux/reference/cheatsheet]] — справочник команд
