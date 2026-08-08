+++
title = "esp rust: настройка defmt"
date = 2026-07-06
description = "Заметка по настройке defmt"
+++

## Настройка defmt на ESP32-C6 (esp-hal + probe-rs)

Заметка по переходу с `rtt-target` на `defmt` для логирования через RTT.

## 1. Зависимости

```toml
[dependencies]
defmt     = "1.1.1"
defmt-rtt = "1.3.0"
```

`rtt-target` можно удалить — его заменяет `defmt-rtt`.

## 2. Линковка: секция `.defmt`

Ошибка при запуске:

```
Error: Failed to parse defmt data
defmt version found, but no `.defmt` section - check your linker configuration
```

Причина: линкер-скрипт `defmt.x` (идёт вместе с крейтом `defmt-rtt`) не подключён.

Фикс: добавить флаг в `.cargo/config.toml`:

```toml
[build]
rustflags = [
  "-C", "link-arg=-Tdefmt.x",
]
```

## 3. Инициализация транспорта

`defmt` сам по себе только форматирует сообщения — куда их слать, решает отдельный крейт-транспорт.
Раз в проекте уже был RTT-канал (через `rtt-target`), логичный выбор — `defmt-rtt`:

```rust
use defmt_rtt as _; // подключает RTT как транспорт для defmt
```

Важно: `rtt_target::rtt_init_print!()` нужно убрать — два способа инициализировать RTT-канал конфликтуют.

## 4. Обязательная временная метка

Без неё сборка падает на линковке:

```
undefined symbol: _defmt_timestamp
```

Каждый defmt-фрейм включает timestamp по протоколу, поэтому нужно явно её задать:

```rust
defmt::timestamp!(
    "{=u64:us}",
    esp_hal::time::Instant::now()
        .duration_since_epoch()
        .as_micros()
);
```

Где `:us` говорит хосту (probe-rs) отформатировать число как секунды с дробной частью
(например, `12.345678`), а не выводить сырые микросекунды.

Если метка не нужна вообще — можно оставить пустой формат, символ всё равно останется
определён для линковки:

```rust
defmt::timestamp!("");
```

## 5. "Собирается, но ничего не печатает"

Вероятная причина — уровень логов не выставлен через переменную окружения `DEFMT_LOG` (аналог `RUST_LOG` из `env_logger`).
По умолчанию уровень `info` не проходит.

Быстрая проверка:

```bash
DEFMT_LOG=info cargo run --release
```

Чтобы не выставлять переменную каждый раз, можно прописать её в `.cargo/config.toml`:

```toml
[env]
DEFMT_LOG = "info"
```

## Итоговый чек-лист

- [x] `defmt` + `defmt-rtt` в зависимостях, `rtt-target` удалён
- [x] `-Tdefmt.x` в `rustflags`
- [x] `use defmt_rtt as _;` в `main.rs`, старая `rtt_init_print!()` убрана
- [x] `defmt::timestamp!` определена (иначе линковка падает)
- [x] `DEFMT_LOG` выставлен (напрямую или через `[env]` в `.cargo/config.toml`)
