+++
title = "db-mcp: подключаем бд к ии"
date = 2026-05-30
description = "Легкий MCP-сервер для подключения к БД написанный на Rust, позволяющий Claude и другим инструментам запрашивать данные из PostgreSQL, MySQL, SQLite и ClickHouse безопасно и приватно"
+++

Нужна была приватная утилита для работы и домашних проектов: читать данные из баз, но ничего не писать. 
Готовые решения не понравились, а те что были — JS/Python-пакеты.
А мне нужен был просто бинарь: скачал и запустил, без всяких Node.js и прочих.
Плюс всегда есть желание сделать что-то своё :)

Так появился **db-mcp** ([github](https://github.com/zeslava/db-mcp)) — небольшой бинарь, который:
- Читает данные из PostgreSQL, MySQL/MariaDB, SQLite и ClickHouse через URL-схему
- Работает с Claude, OpenCode, Jan, Zed и другими MCP-клиентами
- Простой и безопасный: только SELECT, параметризованные запросы
- Собирается один раз со всеми бэкендами сразу — выбираете БД при запуске
- Под Linux — полностью статический бинарь (musl, без glibc в рантайме)

## Почему это было нужно

На работе часто нужно давать Claude доступ к данным: достать информацию, проанализировать, помочь с контекстом для написания скриптов. Нужно было:
- **Безопасно** (только чтение, параметризованные запросы)
- **Приватно** (работает локально, ничего не отправляется в облако)
- **Универсально** (не только Claude — я использую OpenCode, Zed, Jan для разных задач)


## Как это работает

**db-mcp** — это stdio MCP-сервер на Rust. URL базы передаётся флагом `--database-url` или переменной `DATABASE_URL`:

```bash
db-mcp --database-url postgres://user:pass@localhost:5432/mydb
db-mcp --database-url mysql://user:pass@localhost:3306/mydb
db-mcp --database-url sqlite:///absolute/path/to/data.db
db-mcp --database-url clickhouse://default:pass@localhost:8123/default
```

Сервер определяет движок по схеме URL и подключается. Дальше Claude/OpenCode/Jan может использовать три инструмента:

| Tool | Параметры | Что делает |
|------|-----------|------------|
| `list_tables` | — | Список пользовательских таблиц |
| `describe_table` | `table`, `schema?` | Колонки, типы, nullability |
| `query` | `sql` | Выполняет SELECT, возвращает JSON-строки |

`query` возвращает обычный JSON, так что модель получает реальные значения, с которыми можно работать:

```json
[
  { "id": 1, "email": "ada@example.com", "created_at": "2026-05-01 09:12:33" },
  { "id": 2, "email": "linus@example.com", "created_at": "2026-05-03 14:50:01" }
]
```

### Безопасность по дизайну

Ограничение «только SELECT» живёт в сервере, а не в адаптерах. Всё, что не начинается с `SELECT` (без учёта регистра, после `trim`), отклоняется — это блокирует и трюки с CTE вроде `WITH ... INSERT ... RETURNING`. Каждый адаптер использует параметризованные запросы, никакой конкатенации SQL строками. Для дополнительного слоя защиты подключайтесь под ролью с правами только на чтение.

## Настройка для разных инструментов

Все клиенты запускают один и тот же бинарь, отличается только путь к конфигу. Я предпочитаю передавать URL через `DATABASE_URL`, чтобы он не светился в списке аргументов.

### Claude Code (CLI)

**cli**
```bash
claude mcp add db \
  --env DATABASE_URL=postgres://user:pass@localhost:5432/mydb \
  -- /absolute/path/to/db-mcp
```

**claude.json**
```json
{
  "mcpServers": {
    "db-mcp": {
      "type": "stdio",
      "command": "/absolute/path/to/db-mcp",
      "args": [],
      "env": {
        "DATABASE_URL": "postgres://user:pass@localhost:5432/mydb"
      }
    }
  }
}
```

Перезагружаем Claude — инструменты появляются.

### OpenCode

В `opencode.json` (в проекте или `~/.config/opencode/opencode.json`):

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "database": {
      "type": "local",
      "command": ["db-mcp", "--database-url", "sqlite:///home/me/work/data.sqlite"],
      "enabled": true
    }
  }
}
```

### Jan

Jan имеет встроенную поддержку MCP (Settings → MCP Servers). Эквивалентный конфиг — запись command + args + env:

```json
{
  "database": {
    "command": "/absolute/path/to/db-mcp",
    "args": [],
    "env": {
      "DATABASE_URL": "postgres://prod.example.com:5432/analytics"
    }
  }
}
```

### Zed

В `settings.json` (кастомные MCP-серверы живут в `context_servers`, а не в `lsp`):

```json
{
  "context_servers": {
    "db-mcp": {
      "enabled": true,
      "remote": false,
      "command": "/absolute/path/to/db-mcp",
      "args": [],
      "env": {
        "DATABASE_URL": "postgres://user:pass@localhost:5432/mydb"
      }
    }
  }
}
```

## На практике

### Сценарий 1: помощь с контекстом при разработке

```
Я в Claude Code:
"Помоги написать миграцию для добавления нового поля.
Посмотри текущую структуру таблицы users."

Claude:
1. Дёргает db-mcp (describe_table) → узнаёт текущую схему
2. Видит какие поля уже есть, какие constraints
3. Генерирует правильную миграцию для именно вашей БД
```

### Сценарий 2: анализ данных в OpenCode

```
В OpenCode анализирую метрики:
"Сколько активных пользователей в этом месяце?
Какие самые популярные функции?"

OpenCode использует db-mcp (query), модель смотрит реальные данные
и даёт анализ с конкретными цифрами.
```

### Сценарий 3: отладка в Jan

```
Jan с db-mcp помогает разобраться с багом:
"В таблице orders есть записи без user_id?"

Jan запрашивает через db-mcp, видит что есть 12 таких,
предлагает как это почистить.
```

## Установка

Быстрая установка (Linux и macOS Apple Silicon) — определяет ОС/архитектуру, проверяет контрольную сумму, по умолчанию ставит в `~/.local/bin`:

```bash
curl --proto '=https' --tlsv1.2 -sSf \
  https://raw.githubusercontent.com/zeslava/db-mcp/main/install.sh | sh
```

Или скачиваете тарболл из релизов вручную (Linux x86_64/aarch64, macOS arm64, Windows x86_64):

```bash
TARGET=x86_64-unknown-linux-gnu
curl -sSL "https://github.com/zeslava/db-mcp/releases/latest/download/db-mcp-${TARGET}.tar.gz" \
  | tar -xz
install -m 755 "db-mcp-${TARGET}/db-mcp" "$HOME/.local/bin/db-mcp"
```

Или собираете сами:

```bash
git clone https://github.com/zeslava/db-mcp
cd db-mcp
cargo build --release
./target/release/db-mcp --database-url postgres://localhost/mydb
```

## Преимущества такого подхода

- ✅ Один бинарь для четырёх движков (PostgreSQL, MySQL/MariaDB, SQLite, ClickHouse)
- ✅ Работает везде: Claude, OpenCode, Jan, Zed, любой MCP-клиент
- ✅ Никакого runtime (JS, Python) — чистый Rust, статический бинарь
- ✅ Безопасно по дизайну: только SELECT, параметризованные запросы
- ✅ Приватно: работает локально или в вашей инфре
- ✅ Простая настройка: просто URL БД

## Дальше в планах:

- Поддержка других БД (и не только бд)
- Логирование и аудит запросов
- Ограничение доступа по таблицам (для более узких прав)
- Режим сервера
