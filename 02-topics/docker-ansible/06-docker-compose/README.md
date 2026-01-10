### 06-docker-compose

Управление multi-container приложениями с Docker Compose.

---

#### 📚 Содержание

##### **[[01-compose-intro]]**

**Введение в Docker Compose:**
- Что такое Docker Compose
- Когда использовать (development, testing, small production)
- Установка Docker Compose
- Основные команды (up, down, logs, ps, exec)

**YAML синтаксис:**
- Основы YAML (структура, отступы, типы данных)
- Списки и объекты
- Строки и многострочный текст
- Совместимость с JSON
- Валидация YAML

**Простая конфигурация:**
- Структура docker-compose.yml
- Версии Docker Compose
- Блоки: services, networks, volumes
- Базовые примеры (nginx, postgres)

**Сетевое взаимодействие:**
- Автоматическое создание сети
- Service Discovery по имени контейнера
- DNS разрешение между сервисами
- Портмэппинг и expose

**Практические примеры:**
- Simple web + database
- Node.js + MongoDB
- Python + Redis
- Nginx + backend

---

##### **[[02-compose-advanced]]**

**Advanced Docker Compose:**
- Build configuration (build context, dockerfile path, args)
- Зависимости между сервисами (depends_on, condition)
- Restart policies (no, always, on-failure)
- Environment variables (.env файл)
- Переменные в конфигурации (${VAR_NAME})

**Profiles и selective startup:**
- Разделение сервисов по профилям (dev, prod, test)
- Активация профилей при запуске
- Условный запуск сервисов

**Volume management в Compose:**
- Named volumes
- Bind mounts
- Anonymous volumes
- Sharing volumes между сервисами

**Override files:**
- docker-compose.override.yml (автоматический)
- Custom override файлы (-f флаг)
- Слияние конфигураций
- Development vs Production конфиги

**Shared configurations:**
- DRY (Don't Repeat Yourself) принцип
- Использование якорей YAML (&) и ссылок (*)
- Модульные конфиги
- Extends (в v3.0+)

**Масштабирование и оркестрирование:**
- Масштабирование сервисов (docker-compose up --scale)
- Health checks
- Logging и мониторинг
- Performance tuning

**CI/CD интеграция:**
- Запуск Docker Compose в CI/CD
- Тестирование в Docker Compose
- Сбор артефактов

**Практические примеры:**
- Full stack приложение (frontend + backend + БД + cache)
- Микросервисная архитектура (API + Workers + Message Queue)
- Development setup (code sync, hot reload)
- Production-ready конфиг

---

#### 🔗 Структура раздела

```
06-docker-compose/
├── README.md (этот файл)
├── 01-compose-intro.md (основы, YAML, simple конфиги)
└── 02-compose-advanced.md (advanced patterns, profiles, overrides)
```

---

#### ✅ Checklist

**После изучения 01-compose-intro:**
- ✅ Понимаешь что такое Docker Compose и когда использовать
- ✅ Можешь писать простую docker-compose.yml
- ✅ Знаешь основные команды (up, down, logs, ps)
- ✅ Понимаешь сетевое взаимодействие между сервисами
- ✅ Можешь создавать simple multi-container приложение

**После изучения 02-compose-advanced:**
- ✅ Понимаешь build конфигурацию
- ✅ Можешь использовать environment переменные
- ✅ Знаешь о profiles и conditional startup
- ✅ Понимаешь volumes и bind mounts в Compose
- ✅ Можешь использовать override файлы
- ✅ Знаешь о shared configurations и DRY принципе
- ✅ Можешь создавать production-ready конфигурацию