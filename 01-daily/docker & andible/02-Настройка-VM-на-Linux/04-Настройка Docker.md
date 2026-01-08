#### Обновление пакетов

Ага, конечно, вот дока https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository
И вот postintsll https://docs.docker.com/engine/install/linux-postinstall

1. Выполните команду `apt-get update` для обновления списка пакетов.

#### Установка необходимых пакетов

1. Установите следующие пакеты для подготовки системы к установке Docker:
    - `transport-https`
    - сертификаты
    - `curl`
    - `lsb-release`
    - `gnupg`

#### Добавление Docker GPG ключа

1. Добавьте официальный GPG ключ Docker командой `curl -fsSL` [`https://download.docker.com/linux/ubuntu/gpg`](https://download.docker.com/linux/ubuntu/gpg) `| sudo apt-key add -`.

#### Добавление Docker репозитория

1. Добавьте Docker репозиторий в систему:
    - Используйте `sudo add-apt-repository "deb [arch=amd64]` [`https://download.docker.com/linux/ubuntu`](https://download.docker.com/linux/ubuntu) `$(lsb_release -cs) stable"`.
    - Повторно выполните `apt-get update`.

#### Установка Docker

1. Установите Docker, используя `sudo apt-get install docker-ce docker-ce-cli containerd.io`.

#### Добавление пользователя в группу Docker

1. Чтобы использовать Docker без привилегий суперпользователя, добавьте своего пользователя в группу Docker:
    - `sudo usermod -aG docker ваш_пользователь`.
    - Вам потребуется выйти из системы и войти снова, чтобы изменения вступили в силу.

#### Проверка установки

1. После перелогинивания проверьте работоспособность Docker командой `docker ps`.
2. Запустите тестовый контейнер `docker run hello-world` чтобы убедиться, что Docker работает корректно.