---  
title: Knowledge Base  
description: Личная база знаний (Quartz/Obsidian) со структурой Diátaxis по доменам.  
tags: [index, quartz, diataxis, knowledge-base, structure, navigation]  
aliases: [Home, Главная]  
---  
# Knowledge Base  
  
## Как устроено  
- Все заметки лежат в `content/` и используют YAML frontmatter.​  
- Каждая заметка относится ровно к одному типу Diátaxis: tutorial | howto | reference | explanation. ​  
- Внутренние ссылки — в формате wikilinks: `[[path/to/note]]` (без `.md`).​  
  
## Домены  
- [[devops/index]]  
- [[docs/_inbox/01-linux/index]]  
- [[backend/index]] (planned)  
- [[frontend/index]] (planned)  
- [[git/index]] (planned)  
- [[mobile/index]] (planned)  
- [[other/index]] (planned)  
  
## Как добавлять заметки  
1. Выбери домен: devops | linux | backend | frontend | git | mobile | other. ​  
2. Выбери один тип Diátaxis и не смешивай форматы в одном файле.​  
3. Создай файл по шаблону: `content/<domain>/<type>/<name>.md` (kebab-case).​  
4. Если тема “смешанная”, оставь один тип, а остальное вынеси в `See also` ссылками на будущие заметки (planned).​  
  
## Быстрый вход  
- Linux: [[docs/_inbox/01-linux/index]]  
- Docker/Ansible: [[devops/index]]

Path: content/linux/index.md  
---  
title: Linux  
description: Навигация по заметкам Linux (tutorial/howto/reference/explanation).  
tags: [linux, index, navigation, diataxis]  
aliases: [GNU/Linux]  
---  
# Linux  
  
## Разделы  
- [[linux/tutorial/index]]  
- [[linux/howto/index]]  
- [[linux/reference/index]]  
- [[linux/explanation/index]]  
  
## С чего начать (planned)  
- [[linux/tutorial/linux-installation-basics]] (planned)  
- [[linux/reference/users-groups-permissions]] (planned)  
- [[linux/reference/systemd-units-services]] (planned)

Path: content/linux/tutorial/index.md  
---  
title: Linux Tutorials  
description: Обучающие маршруты по Linux “с нуля”.  
tags: [linux, tutorial, index, learning]  
aliases: [Linux Learning Paths]  
---  
# Linux Tutorials  
  
## Здесь будут  
- Пошаговые маршруты с контрольными точками и проверкой результата.​  
  
## Planned  
- [[linux/tutorial/linux-installation-basics]] (planned)

Path: content/linux/howto/index.md  
---  
title: Linux How-to  
description: Практические инструкции “как сделать X” в Linux.  
tags: [linux, howto, index, troubleshooting]  
aliases: [Linux Tasks]  
---  
# Linux How-to  
  
## Здесь будут  
- Короткие решения конкретных задач с разделами “Проверка” и “Если не работает”.​  
  
## Planned  
- [[linux/howto/ssh-keygen-ed25519]] (planned)

Path: content/linux/reference/index.md  
---  
title: Linux Reference  
description: Справочные заметки по Linux (команды, форматы, таблицы).  
tags: [linux, reference, index, commands]  
aliases: [Linux Cheatsheets]  
---  
# Linux Reference  
  
## Здесь будут  
- Справочники без “почему” и без обучающих маршрутов, только факты/команды/таблицы.​  
  
## Planned  
- [[linux/reference/users-groups-permissions]] (planned)  
- [[linux/reference/systemd-units-services]] (planned)

Path: content/linux/explanation/index.md  
---  
title: Linux Explanations  
description: Объяснения концепций Linux (модели, компромиссы, устройство).  
tags: [linux, explanation, index, concepts]  
aliases: [Linux Concepts]  
---  
# Linux Explanations  
  
## Здесь будут  
- Объяснения “как устроено/почему так”, без процедур “сделай раз-два-три”.​

Path: content/devops/index.md  
---  
title: DevOps  
description: Навигация по заметкам DevOps (Docker, Ansible и др.) в формате Diátaxis.  
tags: [devops, index, navigation, docker, ansible]  
aliases: [SRE]  
---  
# DevOps  
  
## Разделы  
- [[devops/tutorial/index]]  
- [[devops/howto/index]]  
- [[devops/reference/index]]  
- [[devops/explanation/index]]  
  
## Популярные темы (planned)  
- [[devops/tutorial/docker-compose-basics]] (planned)  
- [[devops/reference/dockerfile-basics]] (planned)  
- [[devops/reference/ansible-inventory]] (planned)

Path: content/devops/tutorial/index.md  
---  
title: DevOps Tutorials  
description: Обучающие маршруты по Docker/Ansible/инфраструктуре.  
tags: [devops, tutorial, index, learning]  
aliases: []  
---  
# DevOps Tutorials  
  
## Здесь будут  
- Пошаговые сценарии “с нуля” с проверкой результата.​  
  
## Planned  
- [[devops/tutorial/docker-compose-basics]] (planned)  
- [[devops/tutorial/docker-ansible-setup-ubuntu]] (planned)

Path: content/devops/howto/index.md  
---  
title: DevOps How-to  
description: Практические инструкции по DevOps задачам.  
tags: [devops, howto, index, operations]  
aliases: []  
---  
# DevOps How-to  
  
## Здесь будут  
- Короткие инструкции “сделай X” + проверка + диагностика.​

Path: content/devops/reference/index.md  
---  
title: DevOps Reference  
description: Справочник по командам и конфигам (Docker/Ansible и др.).  
tags: [devops, reference, index, docker, ansible]  
aliases: []  
---  
# DevOps Reference  
  
## Здесь будут  
- Команды, опции, шаблоны конфигов, таблицы — без обучения и без объяснений причин.​  
  
## Planned  
- [[devops/reference/docker-compose-commands]] (planned)  
- [[devops/reference/dockerfile-instructions]] (planned)  
- [[devops/reference/ansible-playbook-structure]] (planned)

Path: content/devops/explanation/index.md  
---  
title: DevOps Explanations  
description: Объяснения концепций DevOps (модели, компромиссы, устройство).  
tags: [devops, explanation, index, concepts]  
aliases: []  
---  
# DevOps Explanations  
  
## Здесь будут  
- Объяснения концепций (например: “почему так устроено”), без пошаговых процедур.