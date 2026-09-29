# Методика h2h-present-interval/1.0

Источник: PresentMon 2.3.1, `--v1_metrics --qpc_time`, колонка `msBetweenPresents` (регистронезависимый разбор). Это интервалы между вызовами Present приложения, не обязательно интервалы физического вывода на экран. Dropped presents не отбрасываются. Frame Generation нельзя публиковать в данном профиле: generated/displayed FPS требует отдельной методики.

Сохраняются положительные конечные интервалы. Первая нулевая запись не участвует. Нулевые/отрицательные/NaN/Infinity не заменяются выдуманными значениями. PID фильтруется и PresentMon, и парсером. Весь набор допустимых кадров хранится с адресом swap chain. Для итоговых метрик выбирается цепочка с наибольшим числом кадров (при равенстве — наименьший адрес). Данные разных swap chains никогда не суммируются. Несколько цепочек ставят FRAME_CAPTURE_UNSTABLE; смена цепочки посреди теста может уменьшить покрытие, такой тест следует повторить.

Время кадра: `(QPCTime − sessionStartQpc) / Stopwatch.Frequency`. Телеметрия использует Stopwatch, запущенный в той же точке перед созданием capture-процесса. Возможная разница старта часов — накладные расходы нескольких вызовов, не время асинхронной доставки ETW. Сэмпл телеметрии помечается временем начала опроса. Интервалы после конца настроенного окна не принимаются. Перед окончательным расчётом PresentMon останавливается и его выход дочитывается.

Для N интервалов в миллисекундах:

- Duration = sum(frameTime) / 1000 — покрытая кадрами длительность, может быть чуть меньше выбранного окна.
- Average FPS = N × 1000 / sum(frameTime).
- Median FPS = 1000 / median(frameTime).
- Minimum FPS = 1000 / max(frameTime); Maximum FPS = 1000 / min(frameTime).
- 1% low = 1000 / mean(самые медленные ceil(N × 0.01) интервалы).
- 0.1% low = 1000 / mean(самые медленные ceil(N × 0.001) интервалы).
- Percentile P: сортировка по возрастанию, индекс `(N − 1) × P`, линейная интерполяция между соседями.

Значения не округляются в сохранённых данных. UI округляет только отображение. Live-панель использует последние 2000 событий (не весь тест); окончательные метрики пересчитываются по полному набору. График истории прореживается для отрисовки, RAW не прореживается.

## RAW format v1

Little-endian, заголовок 8 bytes: ASCII `H2HF`, int32 version=1. Далее записи по 24 bytes: float64 timestampSeconds, float64 frameTimeMs, uint64 swapChainAddress. Максимум 2 млн событий (48 MB плюс managed-память), окно до 600 секунд. При достижении лимита сохраняется частичный тест с ошибкой. Компактный binary выбран без gzip для восстановления после аварийного завершения; неполная последняя запись игнорируется при восстановлении.

Каждые 250 ms writer flush, телеметрия checkpoint каждые 5 секунд. Atomic replace result.json/telemetry.json предотвращает чтение наполовину записанного JSON. Это защита от обычных сбоев процесса, не гарантия сохранности при потере питания или отказе диска.

## Quality flags

- SHORT_TEST: покрытая кадрами длительность меньше 30 секунд.
- BENCHMARK_INTERRUPTED: отмена, закрытие игры, исключение или восстановление после сбоя.
- FRAME_CAPTURE_UNSTABLE: нет кадров, несколько swap chains или диапазон timestamp выбранной цепочки меньше 85% заданного окна.
- GPU_NOT_FULLY_LOADED: средняя доступная NVIDIA GPU usage меньше 80%.
- FRAME_GENERATION_ENABLED: пользователь указал генерацию кадров; upload блокируется, локальное измерение разрешено.

Значения CPU/GPU workload synthetic profile не вычисляются из игрового FPS. Локальный FPS не переводится в H2H Score. Формулы новой методики необходимо согласовать с владельцем серверных рейтингов перед широким сравнением с прежним агентом.

## Дополнительные quality flags

HIGH_BACKGROUND_CPU: хотя бы один доступный sample фоновой CPU-нагрузки >20% (system CPU минус process CPU, нормировано на logical processors). THERMAL_THROTTLING: явно доступный датчик вернул true; неизвестное значение не означает отсутствия throttling. VIDEO_CAPTURE_FAILED: запись/проверка MP4 не удалась. SYNTHETIC_COMPONENT_FAILED: ошибка компонента, остальные измерения сохранены.

## Системная методика h2h-synthetic/2.0.0

Quick: 1 проход, 1.5сек на подизмерение. Full: 3 прохода, 3сек. Инициализация/shader compilation исключены из временного окна. Итог — среднее доступных значений; проходы сохраняются отдельно. Variance — максимальный population coefficient of variation метрик, %. Один проход не даёт variance; ошибки не превращаются в нули.

- CPU: 1 либо Environment.ProcessorCount выделенных потоков с барьером начала. Одна операция =32итерации uint xorshift(13,17,5) и multiply0x9e3779b9. Ops/s. Преимущественно integer workload.
- RAM bandwidth: Buffer.BlockCopy64MiB, bytes/seconds/1e6. Cache/runtime влияют на результат.
- RAM latency: random dependent pointer chase4M int32 (16MiB), seed42, ns/access. Working set может помещаться в LLC; это не чистая DRAM latency.
- SSD: собственный64MiB scratchfile, unbuffered+write-through, aligned buffers, DeleteOnClose. Sequential read1MiB MB/s; random4KiB QD1 read/write IOPS. Малый working set, не sustained SSD workload. Пользовательские файлы и raw device не затрагиваются.
- GPU graphics: D3D12 selected DXGI adapter, offscreen1920×1080 RGBA8 fullscreen triangle, fixed64-iteration pixel shader, batch4draws, fence before completion. Draws/s (fps-equivalent), не игровой FPS.
- GPU compute: 65536threads,128dependent float multiply/add iterations,4dispatch batch с UAV barriers, SM5.1 precise,2operations/iteration. GOPS workload, не peak vendor TFLOPS. Readback проверяет конечный результат.

## Видео

WGC HWND, максимум1920×1080 с сохранением пропорций, H26430fps target8Mbps/max10Mbps, no audio, GOP60. Encoder выбирается пробным encode NVENC/AMF/QSV/MF software. Fragmented MP4 сохраняет завершённые фрагменты при сбое. FPS берётся только из PresentMon. FFmpeg drop/duplicate counters относятся к encoder pipeline, не к dropped game presents.

Overlay компонуется drawtext, обновляется до4раз/сек, содержит названия CPU/GPU, установленный объём ОЗУ, UTC, appversion, settings и ID теста. С версии 2.0.3 строка Code выводится только при наличии серверного nonce; заглушка Code local удалена в экранном и видеооверлее. Без сервера — LOCAL/NOT SERVER VERIFIED. С версии 2.0.1 запись начинается после countdown, elapsed в overlay начинается в FPS-окне. Начало MP4 и RAW различаются. Полный SHA256 MP4/RAW фиксируется после закрытия writers. Metadata клиента не является криптографической аттестацией игры.



## Изменения 2.1.0: данные кадров и паспорт

Покадровый файл frames.dat передаётся целиком через возобновляемый API frames при наличии capability benchmarkFrames=h2h-frames/1. Отправляется исходный файл всех swap chains; primarySwapChain и primaryFrameCount задают цепочку итоговых метрик. Сервер обязан самостоятельно проверить хеш, бинарный формат и выбор цепочки. График строится из реальных записей, не из сводных FPS; прореживание допустимо только для отображения.

GPU игры проверяется по приращениям RunningTime движков Windows GPU Engine типа 3D для её PID, с привязкой к DXGI LUID выбранного адаптера. Несколько активных адаптеров означают ambiguous, отсутствие доступных данных — unknown. Проверка выборочная, а не криптографическое доказательство. Mismatch блокирует отправку; неизвестный адаптер отмечается GPU_ADAPTER_UNVERIFIED. API/V-Sync, введённые пользователем, имеют источник user_game_settings. Runtime DXGI не различает DX11/DX12; SyncInterval не доказывает фактический режим V-Sync. Версия игры — FileVersion исполняемого файла, не гарантированный Steam build ID.

Экранный и видеооверлей используют общую компоновку OverlayDesign: тёмная панель, бирюзовый FPS, отдельные колонки метрик, названия CPU/GPU, RAM/VRAM, настройки, ID, UTC, длительность, версия агента и настоящий nonce при наличии. Видео обновляет отдельные поля через открытые файлы постоянного размера с совместным чтением; они не переименовываются и не обрезаются при чтении FFmpeg. Передача/подтверждение файлов не означает принятия результата модерацией.
