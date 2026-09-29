# Проверки Agent 2.0

## Миграция presentation layer на WinUI 3 — 25.09.2026

- WPF production UI заменён на WinUI 3 / Windows App SDK 2.5.1; `H2HBench.Core`, PresentMon, WGC/FFmpeg, DXGI/D3D12, форматы `result.json`/`frames.dat`, DPAPI, mutex и API-потоки сохранены.
- `H2HBench.slnx`: Release build успешен, 0 ошибок. Core runner: все 79 проверок прошли, production POST не выполнялись.
- Single-file self-contained smoke: `ExitCode=0`, WMI hardware scan и DispatcherQueue подтверждены. Визуальный прогон Dashboard выполнен при 125% DPI; проверены centering, clipping, compact-колонки Quick/Live/Last Result и sidebar.
- На этой машине Smart App Control блокирует новые неподписанные managed binaries (`ERROR_SYSTEM_INTEGRITY_POLICY_VIOLATION`). Политика не изменялась; подтверждено, что single-file executable запускается. Production installer должен быть подписан доверенным сертификатом.
- Реальный ETW/game matrix после смены presentation layer не повторялся. Предыдущие измерения ниже относятся к WPF UI и нативному benchmark backend, который миграция не меняла.

Проверено 22.09.2026, Windows11 build26200, i5-13450HX10C/16T, RTX3050 6GB Laptop,16GB DDR5, Micron SSD. SDK10.0.401. Сборка Release и self-contained publish win-x64 успешны. NuGet restore --locked-mode с доступом к официальному NuGet завершён без предупреждений; compiler warnings0/errors0.

## Автоматические проверки

55 проверок console regression runner прошли. Формулы FPS/tails/percentiles/invalid values, swapchains, реальный формат PresentMon2.3.1/QPC, state machine, consent, allowlist/privacy, FG/partial upload rejection, stable installation ID, traversal rejection, RAW, cancel/recovery,600000samples, legacy POST mock, hashes, CPU cancellation, synthetic units/nulls, challenge token storage, lost chunk response/resume, frozen finish bytes, import unverified и legacy ownership proof. Production POST не выполнялись. Лог: artifacts/regression-final.txt.

## Реальная запись и нагрузка

PresentMon запущен с правами ETW. 60-секундный тест **собственного анимированного WPF-окна**, не игрового benchmark:8531frame samples, покрытие59.94278сек, среднее142.319FPS. Эти значения проверяют collector, а не производительность игры.

Одновременно WGC+NVENC: H2641920×1080,30fps,64.133333сек (включает подготовку/countdown),1924encoded frames,7428919bytes, audio отсутствует. Target8Mbps — целевой VBR bitrate, статическое окно даёт меньший фактический размер. FFprobe подтвердил формат, полныйSHA256:7f043ee1a963e8dc8750ae3c10fd54051393f17f627f2fa3892d15d10896531e. Кадр MP4 визуально проверен: встроены FPS/FT/settings/UTC/Agent2.0/TestID/LOCAL marker. Это composited pixels, не внешнее окно overlay.

Quick и Full synthetic реально выполнены. Full3прохода,0ошибок; максимальный CV4.3689%. CPU single15.463Mops/s, multi213.653Mops/s; RAM12069.2MB/s и64.22ns; SSD3243.78MB/s read,17187read/63820write IOPS; DX12 graphics2138.75draws/s иcompute720.90GOPS. Это метрики новой методики, не H2HScore и не сравнение с прежним агентом.

Наблюдались реальные NVIDIA load/clock/temp/power/VRAM, system CPU/RAM. CPU package clock/temperature на этой конфигурации не были доступны; неизвестные показания не подставлялись. Некорректный нулевой package power исключён. DXGI VRAM5.86035GiB вместо усечённых4GB.

Материалы: artifacts/qa-final/capture-smoke.json, synthetic-smoke.json, video-probe.json, video-frame.png, capture-video.mp4. Данные диагностики не включены в пользовательскую portable-сборку.

## Границы проверки

- Проверено WPF окно и собственный offscreenD3D12 workload. Полная игровая матрица DX11/DX12, fullscreen/borderless/windowed, anti-cheat, DPI/multi-monitor и10-минутная непрерывная реальная игра пока не прогнаны.10-минутный dataset проверен автоматически.
- Реально работал NVENC. AMF/QSV/MF software реализованы с runtime probe, но на соответствующем оборудовании не испытаны.
- Overhead записи на игровых сценах не откалиброван. Сравнение ON/OFF требует повторяемого игрового сценария и серии прогонов.
- Future API протестирован mock HTTP, включая обрыв. Production end-to-end JSON+MP4 ждёт реализации сайта по API-SITE-V2-RU.md; клиент не сообщает verified заранее.
- Загрузчик обновлений проверяет HTTPS/version/size/hash, но production update не выполнялся: на сайте опубликована более старая версия1.0.0. Подписанного сертификатом H2HBench установщика нет; поставка portable.

## Проверка исправления 2.0.1

- Все56регрессионных проверок прошли, включая порядок countdown → video start.
- Проверка ETW без прав вернула ETW_ACCESS_DENIED_DETECTED до начала записи.
- Elevated preflight успешно открыл/закрыл ETW session. Повторный10-секундный FPS+video прогон: Completed,1400кадров PresentMon,331кадр MP4,1276952bytes, video complete.
- В первом elevated прогоне FPS также завершился (1365кадров), но MP4 прервался отдельно. Ошибка видео повторно не воспроизвелась; добавлен ограниченный журнал recording-detail для последующей диагностики. Причина конкретного сбоя видео не установлена.
- CS2 к моменту проверки была закрыта; игровой повтор после исправления не выполнялся. Причина исходного отказа FPS установлена по четырём сохранённым CS2 result.json: PresentMon code6/ETW permissions.

## Проверки версии 2.1.0

70 локальных проверок: формулы/жизненный цикл, raw-файл, API-моки, возобновляемая передача кадров при потере ответа, хеши, безопасные ссылки, раздельная модерация, неизвестные поля и сопоставление GPU. Реальные POST на сайт не выполнялись.

Проверен реальный D3D12 workload на NVIDIA GeForce RTX 3050 6GB Laptop GPU: выбранная NVIDIA определена как matched; выбранная Intel UHD при том же процессе — mismatch. Проверка не гарантирует совместимость со всеми играми/античитами; недоступные счётчики дают unknown.

Полная новая компоновка оверлея прошла 60-секундную запись H.264 1280x720 30 FPS с 955 обновлениями и без ошибок доступа. FFprobe подтвердил кодек, длительность и частоту кадров. Проверены WPF-отрисовки локального и серверного режимов. Ранняя промежуточная DLL блокировалась Smart App Control; последняя тестовая сборка запущена без изменения настроек защиты.

Серверный JS-валидатор отдельно проверен на точное сравнение uint64, выбор primary chain, повреждённый SHA256, NaN, неполную запись и несовпадение количества кадров. Серверные endpoints, профильная привязка и приём файла должны быть реализованы владельцем сайта по API-SITE-2.1.md.
