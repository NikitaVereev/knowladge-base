---
title: "Конвенция коммитов"
type: how-to
tags: [git, conventional-commits, commitlint, husky, changelog, semver]
sources:
  docs: "https://www.conventionalcommits.org/en/v1.0.0/"
  commitlint: "https://github.com/conventional-changelog/commitlint"
  release-please: "https://github.com/googleapis/release-please"
related:
  - "[[git/explanation/branching-models]]"
  - "[[git/how-to/recipes/gitconfig-and-aliases]]"
  - "[[git/how-to/interactive-rebase]]"
  - "[[git/reference/cheatsheet]]"
---

# Конвенция коммитов

> **TL;DR:** Формат `type(scope): description` даёт машиночитаемую историю, автоматический CHANGELOG и семантическое версионирование. Без тулинга конвенция — просто пожелание; commitlint + husky делают её обязательной.

## Формат сообщения

```
<type>(<scope>): <description>    ← обязательно (subject line, ≤72 символа)
                                   ← пустая строка
[body]                             ← опционально (что и ПОЧЕМУ, а не как)
                                   ← пустая строка
[footer(s)]                        ← опционально (BREAKING CHANGE, Refs, Closes)
```

### Типы

| Type       | Когда                                  | SemVer   |
|------------|----------------------------------------|----------|
| `feat`     | Новая функциональность                 | MINOR    |
| `fix`      | Исправление бага                       | PATCH    |
| `docs`     | Только документация                    | —        |
| `style`    | Форматирование, пробелы, точки с запятой (не CSS) | — |
| `refactor` | Ни feat, ни fix — изменение структуры  | —        |
| `perf`     | Улучшение производительности           | PATCH    |
| `test`     | Добавление / исправление тестов        | —        |
| `build`    | Сборка, зависимости (npm, webpack)     | —        |
| `ci`       | CI конфигурация (GitHub Actions, etc.) | —        |
| `chore`    | Всё остальное (не затрагивает src/test)| —        |
| `revert`   | Откат предыдущего коммита              | —        |

### Scope (опционально)

Область изменения — модуль, компонент, слой:

```bash
feat(auth): add JWT refresh token rotation
fix(api/users): handle duplicate email on registration
docs(readme): add deployment section
ci(docker): switch to multi-stage build
```

Скоупы команда фиксирует сама. Хороший набор для типичного проекта:

```
auth, api, ui, db, config, docker, ci, deps
```

### Breaking Changes

Два способа пометить несовместимое изменение (оба дают MAJOR-версию):

```bash
# Способ 1: ! после type/scope
feat(api)!: change auth endpoint response format

# Способ 2: footer BREAKING CHANGE
feat(api): change auth endpoint response format

BREAKING CHANGE: /api/auth/login now returns { accessToken, refreshToken }
instead of { token }. All clients must update token handling.
```

> **Важно:** `BREAKING CHANGE:` в footer — ровно так, с двоеточием. Это часть спецификации, не произвольный текст.

### Body и Footer

```bash
fix(api): prevent race condition in order processing

Multiple concurrent requests for the same order could create
duplicate entries. Added distributed lock using Redis SETNX
with TTL to ensure exactly-once processing.

Refs: JIRA-1234
Closes #892
```

Body отвечает на вопрос **«почему»**, а не «что» — diff покажет что изменилось, но не покажет причину.

## Примеры — хорошие vs плохие

```bash
# ❌ Плохо
git commit -m "fix"                         # что fix?
git commit -m "update"                      # что обновлено?
git commit -m "feat: some changes"          # не описывает суть
git commit -m "fix: fix bug"               # тавтология
git commit -m "feat(auth): Added new feature to handle user authentication flow with JWT tokens and refresh"
                                            # > 72 символа, расплывчато

# ✅ Хорошо
git commit -m "feat(auth): add JWT refresh token rotation"
git commit -m "fix(db): handle connection pool exhaustion on spike"
git commit -m "refactor(orders): extract pricing logic into service"
git commit -m "test(payments): add integration tests for Stripe webhook"
git commit -m "chore(deps): bump express from 4.18.2 to 4.19.0"
```

## Настройка commitlint + husky

### Установка

```bash
# commitlint — линтер сообщений коммитов
npm install -D @commitlint/cli @commitlint/config-conventional

# husky — git hooks через npm
npm install -D husky
npx husky init  # создаёт .husky/ и добавляет prepare-скрипт
```

### Конфигурация commitlint

```js
// commitlint.config.js
export default {
  extends: ['@commitlint/config-conventional'],
  rules: {
    // type должен быть из списка [2 = error, 'always', [...values]]
    'type-enum': [2, 'always', [
      'feat', 'fix', 'docs', 'style', 'refactor',
      'perf', 'test', 'build', 'ci', 'chore', 'revert',
    ]],
    // scope — lowercase
    'scope-case': [2, 'always', 'lower-case'],
    // subject — не пустой, ≤72 символа, не заканчивается точкой
    'subject-empty': [2, 'never'],
    'subject-max-length': [2, 'always', 72],
    'subject-full-stop': [2, 'never', '.'],
    // body — переносы по 100 символов
    'body-max-line-length': [2, 'always', 100],
  },
};
```

### Подключение через husky

```bash
# Создать hook commit-msg
echo 'npx --no -- commitlint --edit $1' > .husky/commit-msg
```

Теперь каждый `git commit` проходит через commitlint:

```bash
git commit -m "update stuff"
# ⧗   input: update stuff
# ✖   subject may not be empty [subject-empty]
# ✖   type may not be empty [type-empty]
# ✖   found 2 problems, 0 warnings

git commit -m "feat(auth): add password reset flow"
# ✔ (проходит)
```

### Для проектов без npm (bash-альтернатива)

Если проект не на Node.js — используй git hook напрямую (уже есть в [[git/how-to/recipes/gitconfig-and-aliases]]):

```bash
#!/bin/bash
# .git/hooks/commit-msg
commit_msg=$(cat "$1")
pattern="^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\(.+\))?(!)?: .{1,72}"

if ! echo "$commit_msg" | grep -qE "$pattern"; then
    echo "❌ Conventional Commits format required:"
    echo "   type(scope): description"
    echo ""
    echo "   Examples:"
    echo "   feat(auth): add login endpoint"
    echo "   fix: resolve memory leak in worker"
    exit 1
fi
```

```bash
chmod +x .git/hooks/commit-msg
```

## Автоматический CHANGELOG

Конвенция коммитов → парсинг истории → генерация changelog + bump версии.

### release-please (рекомендуется для GitHub)

GitHub Action от Google: парсит историю коммитов, создаёт Release PR с обновлённым CHANGELOG и версией. Мержишь PR — получаешь git tag и GitHub Release.

**Prerequisite:** Settings → Actions → General → Workflow permissions: **Read and write permissions** + **✅ Allow GitHub Actions to create and approve pull requests**

> **Примечание:** На приватных репозиториях branch protection (запрет прямого push в main) доступен только начиная с GitHub Pro ($4/мес). На публичных — бесплатно.

```yaml
# .github/workflows/release-please.yml
name: release-please

on:
  push:
    branches:
      - main

permissions:
  contents: write
  pull-requests: write

jobs:
  release-please:
    runs-on: ubuntu-latest
    steps:
      - uses: googleapis/release-please-action@v4
        with:
          release-type: node  # node, python, rust, go, elixir, etc.
```

Что происходит:

1. Push в main → action анализирует коммиты с последнего релиза
2. Создаёт/обновляет Release PR с CHANGELOG и version bump
3. Merge Release PR → tag + GitHub Release автоматически

> **Важно:** По умолчанию используется `GITHUB_TOKEN`, но ресурсы созданные через него не триггерят другие workflows. Если нужен CI на Release PR — используй Personal Access Token.

### commit-and-tag-version (локальная альтернатива)

Поддерживаемый форк deprecated `standard-version`. Для проектов без GitHub Actions или когда нужен релиз из CLI:

```bash
npm install -D commit-and-tag-version
```

```jsonc
// package.json
{
  "scripts": {
    "release": "commit-and-tag-version",
    "release:minor": "commit-and-tag-version --release-as minor",
    "release:major": "commit-and-tag-version --release-as major"
  }
}
```

```bash
npm run release
# 1. Анализирует коммиты с последнего тега
# 2. Определяет тип версии (patch/minor/major) по типам коммитов
# 3. Обновляет package.json version
# 4. Генерирует/обновляет CHANGELOG.md
# 5. Создаёт коммит "chore(release): 2.1.0"
# 6. Ставит git tag v2.1.0
```

### Что выбрать

| | release-please | commit-and-tag-version |
|---|---|---|
| Где работает | GitHub Actions | Локально / любой CI |
| Подход | Release PR → merge → release | CLI команда → commit + tag |
| Контроль | Через merge PR | Ручной запуск |
| Монорепо | Да (manifest config) | Нет |
| Для кого | GitHub-проекты с CI/CD | Локальная разработка, non-GitHub |

Результат в `CHANGELOG.md` (оба инструмента генерируют похожий формат):

```markdown
## [2.1.0] - 2026-02-15

### Features
* **auth:** add JWT refresh token rotation (a1b2c3d)
* **api:** add rate limiting per API key (d4e5f6a)

### Bug Fixes
* **db:** handle connection pool exhaustion (b7c8d9e)

### BREAKING CHANGES
* **api:** /api/auth/login response format changed
```

### Связь с SemVer

```
fix:            → PATCH (1.0.0 → 1.0.1)
feat:           → MINOR (1.0.0 → 1.1.0)
BREAKING CHANGE → MAJOR (1.0.0 → 2.0.0)
```

Остальные типы (`docs`, `style`, `refactor`, `test`, `ci`, `chore`) не влияют на версию — они не попадают в CHANGELOG (если не настроить иначе).

## Подводные камни

> **Важно:** Конвенция работает, только если её enforce-ить тулингом. «Мы договорились» без husky/commitlint — первый же пятничный деплой всё сломает.

| Проблема | Симптом | Решение |
|----------|---------|---------|
| Один огромный коммит `feat: redesign` | Невозможно откатить часть | Атомарные коммиты: один коммит = одно логическое изменение |
| Неправильный type | `feat` для багфикса сломает semver | Code review + commitlint |
| Слишком широкий scope | `feat(app): ...` — бесполезен | Фиксированный список скоупов в commitlint |
| `fix: fix bug` | Не несёт информации | subject описывает **что** исправлено, body — **почему** |
| Merge-коммиты ломают changelog | Дубликаты в CHANGELOG | Squash merge или rebase workflow |
| husky не запускается в CI | Hooks не выполняются | `HUSKY=0` в CI env или `--no-verify` только в CI-скриптах |

## Связанные материалы

- [[git/explanation/branching-models]] — модели ветвления, именование веток
- [[git/how-to/recipes/gitconfig-and-aliases]] — git hooks без npm, commit-msg hook
- [[git/how-to/interactive-rebase]] — reword / squash коммитов перед merge
