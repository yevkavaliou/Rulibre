# Исходная среда и воспроизводимые проверки

Проблема №2 из `PRE_UI_PLAN.md`, 11 октября 2026 года (Europe/Warsaw).
Исходная ревизия: `7d926068619909207518ebddc1754e23de95ae05`.
Рабочая ветка: `codex/pre-ui-baseline`. Коммит/PR этого этапа пока не создан.

## Среда

| Инструмент | Результат |
| --- | --- |
| ОС / host | Windows, build 26100; `x86_64-pc-windows-msvc` |
| PowerShell | 7.6.5 |
| `rustc -Vv` | 1.93.1, commit `01f6ddf7588f42ae2d7eb0a2f21d44e8e96674cf`, LLVM 21.1.8 |
| `cargo -V` | 1.93.1 (`083ac5135`, 2025-12-15) |
| `rustup target list --installed` | Только `x86_64-pc-windows-msvc` |
| `ebook-convert --version` | Calibre 8.16.2, `C:\Program Files\Calibre2\ebook-convert.exe` |
| `kepubify` | Не найден в PATH |
| `trunk` | Не найден в PATH |
| `cargo-tauri` / Tauri CLI | Не найден в PATH |

Первые попытки в песочнице столкнулись с `Access is denied` и невозможностью
доступа к crates.io. Повторные проверки с разрешённым доступом к toolchain,
Cargo cache и сети дали результаты ниже. Это ограничения запуска агента,
а не ошибки исходного Rust-кода. Дополнительные инструменты не устанавливались.

## Исходные результаты

`cargo test` и `cargo check` без выбора пакета используют default-members:
`rulibre-core` и `rulibre`, включая диагностический `scan_metadata`.
Проверки выполнены с исходным Cargo.lock; зависимости и публичный API не менялись.

| Команда из корня репозитория | Exit | Результат / журнал |
| --- | --- | --- |
| `cargo test --locked` | 0 | В оригинале 0 тестов; [журнал](baseline/original-test.log) |
| `cargo check --locked` | 0 | Ядро, TUI, `scan_metadata`; [журнал](baseline/original-check.log) |
| `cargo fmt --all -- --check` | 1 | Исходные расхождения форматирования только GUI; [diff](baseline/original-fmt.log) |
| `cargo check --locked -p rulibre-gui` | 101 | Tauri build script: отсутствует `icons/icon.ico`, необходимый для Windows Resource; [журнал](baseline/original-gui-host.log) |
| `cargo check --locked -p rulibre-gui --target wasm32-unknown-unknown` | 101 | E0463: отсутствует WASM target; [журнал](baseline/original-wasm.log) |

GUI исходники не исправлялись. Побочные изменения генерируемых схем после
проверки host убраны. Запуск окна и упаковка Tauri не проверялись.

## Повторение обязательных проверок

Нужны Rust/Cargo с поддержкой edition 2024, rustfmt и host linker.
При первом запуске нужен доступ к crates.io; далее можно использовать Cargo cache.
Из корня репозитория:

```powershell
pwsh -File scripts/check-baseline.ps1
```

Или на любой ОС выполнить последовательно:

```sh
cargo test --locked
cargo check --locked
cargo fmt -p rulibre-core -p rulibre -- --check
```

После добавления инфраструктуры все эти проверки прошли: 3 unit-теста
подменяемого запуска процессов и 3 интеграционных теста. Журналы:
[тесты](baseline/current-test.log), [check](baseline/current-check.log).
Проверка форматирования всего workspace остаётся отдельной командой с известным
исходным сбоем GUI; скрипт проверяет обязательные ядро/TUI.

## Фикстуры

`crates/rulibre-core/src/test_support.rs` создаёт уникальный временный каталог
через атомарный `create_dir` и удаляет его при Drop. Тесты могут идти параллельно,
не меняют PATH и не обращаются к личным конфигам, библиотеке или реальным томам.

Библиотека содержит Unicode и пробелы в пути, два каталога книг, OPF с namespace,
двумя авторами, серией и ISBN, файлы EPUB/KEPUB/PDF/MOBI/TXT, обложку,
`.DS_Store` и служебные каталоги Calibre. OPF лежит в
`crates/rulibre-core/tests/fixtures/metadata.opf`. Файлы книг — текстовые заглушки
для файловых тестов, непригодные для проверки настоящей конвертации.

Каталоги устройств содержат `.kobo/version` и `Kindle/documents`.
Интеграционный тест передаёт явно созданные Device и готовые KEPUB/MOBI,
проверяет равенство содержимого назначения и источника. Он не вызывает
автообнаружение томов и не подтверждает поддержку физического ридера.

В converter добавлен приватный ProcessRunner: production делегирует в прежний
`Command::output`, тестовый runner записывает программу и отдельные аргументы
и выдаёт заданные Output/ошибки запуска. Проверены отсутствие программы,
ненулевой exit, успешная ветка kepubify, ветка ebook-convert и stderr.
Реальные процессы в тестах не запускаются. Поведение конвертации не исправлялось.

## Матрица для финальной проверки №15

| Проверка | На этапе №2 | Что повторить в №15 |
| --- | --- | --- |
| Windows ядро/TUI/scan_metadata | test/check и форматирование прошли | test/check/fmt/Clippy и итоговый интеграционный сценарий |
| Linux / macOS | Не проверены | CI test/check/fmt/Clippy, платформенная логика томов |
| Windows Tauri host | Исходный сбой отсутствующего icon.ico | Устранить исходный дефект, повторить host check и сборку |
| WASM frontend | Target отсутствует | Установить target, выполнить отдельный check |
| Trunk / Tauri packaging / IPC | CLI отсутствуют, не проверены | Сборка и smoke-test после подготовки GUI среды |
| Настоящий ebook-convert | Проверена только версия | Конвертация валидного тестового ebook и обработка ошибок |
| Настоящий kepubify | Не доступен | Проверить с установленным конвертером |
| Физические Kobo / Kindle | Не проверены | Отдельный smoke-test обнаружения, подготовки и передачи |

Эта матрица фиксирует перенос проверок; выполнение этапа №15 не заявляется.
