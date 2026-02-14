---
title: "Модели ветвления"
type: explanation
tags: [git, explanation, branching, gitflow, trunk-based, github-flow]
related:
  - "[[git/tutorials/02-branching]]"
  - "[[git/tutorials/03-collaboration]]"
---

# Модели ветвления

> **TL;DR:** GitHub Flow — простой (main + feature branches + PR). Trunk-Based — ещё проще (main + short-lived branches).
> Git Flow — сложный (main, develop, feature, release, hotfix). Для большинства проектов: GitHub Flow.

## GitHub Flow (рекомендуется)

```
main ────●────●────●────●────●────
          \       /      \       /
feature    ●──●──●       ●──●──●
```

1. `main` всегда deployable
2. Создать feature-ветку от main
3. Коммитить, пушить
4. Открыть Pull Request
5. Code review → merge в main
6. Deploy

**Когда:** Большинство проектов, CI/CD, небольшие команды.

## Trunk-Based Development

```
main ────●────●────●────●────●────
          \  /      \  /
short      ●        ●     (1-2 дня max)
```

Как GitHub Flow, но ветки живут 1-2 дня максимум. Feature flags вместо долгих веток.

**Когда:** Команды с сильным CI/CD, микросервисы, Google/Facebook стиль.

## Git Flow

```
main     ────●─────────────●──────────●────
              \             ↑           ↑
develop  ──●───●──●──●──●──●──●──●──●──●──
            \     /    \      /
feature      ●──●       ●──●
                    \
release              ●──●
                          \
hotfix                     ●
```

Ветки: `main` (production), `develop` (интеграция), `feature/*`, `release/*`, `hotfix/*`.

**Когда:** Версионированные продукты, релизные циклы, большие команды.

## Сравнение

| | GitHub Flow | Trunk-Based | Git Flow |
|---|------------|-------------|----------|
| Сложность | Низкая | Низкая | Высокая |
| Ветки | main + feature | main + short | main + develop + feature + release |
| Deploy | После каждого merge | Непрерывно | По релизам |
| Для кого | Большинство | CI/CD-зрелые команды | Версионированные продукты |

## Конвенция именования веток

```
feature/auth-login     # новая функциональность
fix/broken-link        # исправление бага
hotfix/security-patch  # срочное исправление
chore/update-deps      # рутина
docs/api-reference     # документация
```

## Конвенция коммитов (Conventional Commits)

```
feat: add user login
fix: resolve null pointer in auth
docs: update API reference
chore: bump dependencies
refactor: extract auth module
test: add unit tests for payment
```