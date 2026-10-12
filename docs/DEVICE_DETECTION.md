# Обнаружение смонтированных ридеров

Проблема №9, 11 октября 2026 года.

## API и модель

Перечисление файловых систем отделено от распознавания:

- volumes::mounted_volumes() -> Result<VolumeReport, Error>: тома с OS ID,
  точкой монтирования, меткой (если доступна), типом FS и read_only, плюс предупреждения.
- device::detect_devices() -> Result<DiscoveryReport, Error>: полный список
  подтверждённых устройств и предупреждения по отдельным томам. Ошибка таблицы
  томов возвращается как Err; ошибка одного тома не скрывает остальные.
- detect_devices_from_volumes(&volumes): тот же классификатор для явного снимка,
  включая синтетические тестовые тома.
- manual_device(path, kind): явный выбор типа с проверкой актуального списка томов.
  manual_device_from_volumes(path, kind, &volumes) позволяет передать снимок
  собственного enumerator или тестовую фикстуру; достоверность такого снимка
  является ответственностью вызывающего кода.

Device теперь содержит id, kind, mount_point: PathBuf, label, filesystem,
read_only и evidence: Vec<DetectionEvidence>. Модель Device/DeviceEvent общая
для ядра и WASM frontend; форма событий Connected/Disconnected сохраняется.
ID состоит из типа ридера и идентификатора тома. Снимки сортируются по ID и пути;
несколько путей одного тома не создают несколько устройств.

single_device() возвращает устройство только при ровно одном распознанном ридере.
TUI/Tauri используют этот контракт: среди нескольких не выбирается первый.
Экраны ручного выбора не добавлялись. API ручного выбора есть в ядре; его UI/IPC
не добавлены. Старое наблюдение через was_connected пока сохранено; замена ридера
без промежуточного None и управляемая остановка наблюдателей относятся к №10.
Ошибки/предупреждения опроса пишутся в stderr; при критической ошибке опроса
текущий выбор сохраняется до следующего успешного опроса.

## Правила распознавания и подтверждение источниками

| Тип | Обязательные признаки автоматического распознавания |
| --- | --- |
| Kobo | Обычный каталог .kobo; файл .kobo/version с непустым первым полем и числовой dotted-версией в третьем CSV-поле; обычный файл .kobo/KoboReader.sqlite с 16-байтным заголовком SQLite |
| Kindle | Обычный каталог documents; обычный каталог system; файл system/version.txt с префиксом Kindle и числовой dotted-версией |

Версии читаются максимум до 4097 байт; свыше 4096 признак не подтверждается.
База Kobo не открывается через SQLite и не изменяется: читается только заголовок.
Ссылки/junction в каталогах признаков и ссылки вместо файлов не подтверждают ридер.
Обычный диск, одна папка documents, имя каталога Kindle/KOBOeReader и одна метка
тома сами по себе не являются основанием. Известная метка без подтверждённых
файлов даёт предупреждение, устройство не возвращается.

Метку и путь можно переименовать: характерные файлы достаточны независимо от
имени тома. Метка сохраняется как наблюдение в evidence; неизвестная метка не
мешает распознаванию. Известные метки KOBOeReader/Kobo и Kindle/Kindle Main Memory/
Kindle Storage Card используются для выявления противоречий. Сочетание сигнатур
обоих типов или противоречие известной метки и файлов исключает автоматический
выбор и возвращает предупреждение. Корень с metadata.db исключается.

Источники правил:

- [Драйвер Kobo в Calibre](https://github.com/kovidgoyal/calibre/blob/master/src/calibre/devices/kobo/driver.py): .kobo/version, CSV и поле firmware под индексом 2, путь .kobo/KoboReader.sqlite. Это основания выбранной политики, а не проверка наших физических устройств.
- [Формат SQLite](https://sqlite.org/fileformat.html#magic_header_string): 16-байтный заголовок. Проверка заголовка не утверждает целостность всей базы.
- [Официальная инструкция Kobo](https://help.kobo.com/hc/en-us/articles/360024775093-Add-non-protected-PDF-and-ePub-files-to-your-Kobo-eReader-using-your-computer): обычное имя диска KOBOeReader.
- [Наблюдение владельца Kindle Paperwhite 2](https://technicallycompetent.com/blog/kindle-pw2-2025/): реальный system/version.txt с формой Kindle 5.4.0 (...). Это первичный отчёт о конкретном устройстве; перенос правила на другие модели — консервативная эвристика.
- [Драйвер Kindle в Calibre](https://github.com/kovidgoyal/calibre/blob/master/src/calibre/devices/kindle/driver.py): каталоги хранения и метки Kindle Main Memory/Kindle Storage Card. Calibre также использует USB-идентификацию; Rulibre на этом этапе использует только смонтированные файловые системы.

Распознавание проверяет характерную структуру, а не подлинность USB-оборудования.
Диск с намеренно скопированной полной структурой ридера неотличим этой эвристикой.
Для моделей/прошивок без выбранных признаков предусмотрен ручной путь.
Ни одна конкретная модель пока не заявляется проверенной нами на физическом ридере.

## Перечисление по ОС и идентификация

| ОС | Перечисление | Метка и ID | Что проверено |
| --- | --- | --- | --- |
| Windows | FindFirstVolumeW/FindNextVolumeW, GetVolumePathNamesForVolumeNameW, GetVolumeInformationW, GetDriveTypeW; буквы дисков и mounted folders | Метка из API; ID из volume GUID, не буквы | Сборка и настоящий read-only smoke-test: 4 локальных тома, 0 ридеров, 0 предупреждений |
| Linux | /proc/self/mountinfo; optional fields и octal-экранирование путей; bind mount подкаталога исключён | /dev/disk/by-label с декодированием udev; ID по UUID из /dev/disk/by-uuid, fallback major:minor + source | Разбор фикстур и меток протестирован на Windows; Linux-сборка и живое перечисление не проверены |
| macOS | libc::getmntinfo(MNT_NOWAIT), реальные statfs и только MNT_LOCAL | Метка не извлекается (None); fallback ID по mount source /dev/disk... | Код добавлен; сборка и живое перечисление не проверены |

ID Windows/UUID Linux не зависят от отображаемого пути. Fallback Linux/macOS
идентифицирует текущий OS storage, а не серийный номер физического ридера;
повторное подключение или повторное использование /dev имени может изменить
идентичность или дать тот же fallback для другого накопителя. Клонированные UUID
также не гарантируют уникальность оборудования. На macOS getmntinfo использует
статический буфер libc; вызовы этого модуля защищены mutex, снимок копируется.

Документация native API:
[Windows volumes](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-findfirstvolumew),
[Windows mount paths](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-getvolumepathnamesforvolumenamew),
[Linux mountinfo](https://docs.kernel.org/filesystems/proc.html#proc-pid-mountinfo-information-about-mounts),
[macOS getmntinfo](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man3/getmntinfo.3.html).

Поддерживаются локальные дисковые FS из явного списка в volumes::local_filesystem;
network/pseudo FS и MTP/FUSE bridges исключены. Это не реализация MTP: устройства
без смонтированной локальной файловой системы не обнаруживаются этим API и не
принимаются через manual_device. Отдельный backend MTP остаётся вне плана.

## Ручной путь

Требуется существующий корень одного из актуальных локальных томов. Подкаталог,
обычный не смонтированный каталог, отсутствующий путь, read-only FS, Calibre-
библиотека и MTP отклоняются. Для выбранного Kobo должен существовать обычный
каталог .kobo, для Kindle — обычный documents. Каталоги не создаются; probe-запись
на устройство не выполняется. Права на запись ещё могут привести к I/O-ошибке
при фактической передаче; защита копирования и повторная проверка mount — №12.

Пользователь явно утверждает тип: ручной выбор допускается при неполных или
противоречивых автоматических сигнатурах и отмечается ManualSelection. Одна
папка documents достаточна только при таком явном выборе смонтированного тома,
никогда при автоматическом распознавании. Ошибки валидации содержат операцию и путь.

## Проверки и ограничения

cargo test --locked --offline: 56 автоматических тестов прошли, 13 новых
(10 тестов распознавания/ручного пути и 3 теста Linux parser/udev). Покрыты обычный
диск, метки без сигнатур, documents-only флешка, переименование Kobo/Kindle,
несколько ридеров, детерминированный порядок и дедупликация, изменение пути при
сохранении ID, противоречия, повреждённые признаки, отсутствие/неправильный тип
файла, реальная Windows-блокировка чтения, junction, ручной путь, read-only,
Calibre-библиотека, MTP/network, mountinfo escapes и malformed records.
Проверки передачи из предыдущих этапов адаптированы к модели и остаются успешными.

cargo check --locked --offline (ядро/TUI/scan_metadata), форматирование ядра/TUI
и изменённых GUI-адаптеров прошли. Clippy ядра прошёл с прежним предупреждением
collapsible_if в metadata.rs. Tauri host check и 3 теста транспорта прошли;
новый тест проверяет идентификатор, Unicode-путь и evidence в DeviceEvent.
Для host использовались временные icon.ico/dist, удалённые после проверки,
генерируемые схемы сохранены. Запуск окна и настоящий IPC не проверены.
WASM target отсутствует; модель frontend адаптирована, его сборка не подтверждена.

Отдельно выполнен настоящий opt-in smoke-test Windows volume API. Команда:

~~~powershell
cargo test --locked --offline -p rulibre-core native_volume_enumeration_smoke -- --ignored --nocapture
~~~

Это чтение таблицы реальных томов и признаков в их корнях, не проверка физического
Kobo/Kindle. Автоматические тесты используют синтетические каталоги и снимки,
не выдают их за смонтированные физические устройства. Проверки выполнялись вне
Windows sandbox с TMP/TEMP в target/device-tests-temp из-за ограничений
canonicalize, junction/hard links и поиска конфигурации Clippy.
Linux/macOS runtime и сборки, физические ридеры, модели MTP не проверены.
Результаты перенесены в матрицу №15; коммит/PR не создан.