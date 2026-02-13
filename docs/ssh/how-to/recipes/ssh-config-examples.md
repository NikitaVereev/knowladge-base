---
title: "Рецепт: Примеры SSH config"
type: how-to
tags: [ssh, recipe, config, examples, bastion, vagrant, github]
related:
  - "[[ssh/how-to/configure-client]]"
  - "[[ssh/how-to/tunnels]]"
---

# Рецепт: Примеры ~/.ssh/config

> Готовые блоки для копирования. Адаптируйте IP, имена и пути под себя.

## Базовый шаблон

```
# === Defaults ===
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    AddKeysToAgent yes
    IdentitiesOnly yes

# === Production ===
Host prod
    HostName 203.0.113.10
    User deploy
    Port 2222
    IdentityFile ~/.ssh/prod_ed25519

# === Staging ===
Host staging
    HostName staging.example.com
    User deploy
    IdentityFile ~/.ssh/staging_ed25519

# === Dev (Vagrant VM) ===
Host dev
    HostName 127.0.0.1
    User vagrant
    Port 2222
    IdentityFile .vagrant/machines/default/virtualbox/private_key
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
```

## Bastion / Jump Host

```
Host bastion
    HostName bastion.example.com
    User admin
    IdentityFile ~/.ssh/bastion_key

Host db
    HostName 10.0.1.50
    User postgres
    ProxyJump bastion

Host app-*
    User deploy
    ProxyJump bastion

Host app-1
    HostName 10.0.1.10

Host app-2
    HostName 10.0.1.11
```

```bash
ssh db        # автоматически через bastion
ssh app-1     # тоже через bastion
```

## Tunnel presets

```
Host tunnel-db
    HostName bastion.example.com
    User admin
    LocalForward 5432 db.internal:5432
    LocalForward 6379 redis.internal:6379
    RequestTTY no

Host tunnel-proxy
    HostName server.example.com
    User deploy
    DynamicForward 1080
    RequestTTY no
```

## GitHub / GitLab

```
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_ed25519

Host gitlab.com
    HostName gitlab.com
    User git
    IdentityFile ~/.ssh/gitlab_ed25519

# Несколько GitHub-аккаунтов
Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/work_ed25519

Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/personal_ed25519
```

```bash
# Клонировать с work-аккаунта:
git clone git@github-work:company/repo.git
```

## Multiplexing (быстрые повторные подключения)

```
Host *
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 600
```

```bash
mkdir -p ~/.ssh/sockets
```
