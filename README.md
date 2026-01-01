# PulseCheck one2one

**Автоматическая система записи, транскрибации и AI-анализа встреч для macOS**

PulseCheck one2one - это профессиональное Electron-приложение для автоматической записи онлайн-встреч, их транскрибации с помощью Whisper.cpp и AI-саммаризации. Система автоматически определяет начало и окончание звонков через прямой мониторинг аудио драйвера, записывает оба аудиопотока (ваш микрофон + системный звук) и интегрируется с корпоративной системой KTalk.

## Ключевые возможности

- **Автоматическая запись** - определение начала/окончания встречи без участия пользователя через прямой мониторинг аудио драйвера
- **Двусторонняя запись** - синхронная запись микрофона и системного звука через собственный драйвер
- **Локальная транскрибация** - Whisper.cpp large-v3-turbo с VAD (без отправки данных на сервера)
- **AI-саммаризация** - автоматическое создание кратких отчетов через OpenRouter API
- **AI-оценка встреч** - настраиваемые чек-листы для оценки качества встреч
- **Интеграция с KTalk** - автоматическая привязка записей к событиям календаря
- **Встроенный UI** - React-интерфейс для управления историей встреч
- **База данных** - SQLite для хранения метаданных, транскриптов и саммари
- **E2E тесты** - Playwright для автоматического тестирования

## Архитектура

### Основные компоненты

```
pulsecheck-one2one/
├── main.js                           # Main process entry point
├── preload.js                        # Preload script для безопасности
├── index.html                        # Main window (системный трей)
├── src/
│   ├── main/                         # Main process
│   │   ├── audio/
│   │   │   ├── audioRecorder.js      # Двусторонняя запись через драйвер
│   │   │   ├── audioTranscriber.js   # Whisper.cpp транскрибация + VAD
│   │   │   ├── chunking.js           # VAD-based чанкирование аудио
│   │   │   ├── chunkMerger.js        # Объединение чанков с overlap
│   │   │   ├── repetitionGuard.js    # Защита от зацикливания модели
│   │   │   └── driverInstaller.js    # Установка PulseCheck Audio драйвера
│   │   ├── ai/
│   │   │   ├── transcriptionSummarizer.js  # AI саммаризация
│   │   │   └── checklists/
│   │   │       └── checklistEvaluator.js   # AI-оценка встреч
│   │   ├── meetings/
│   │   │   └── osHeuristicsDetector.js    # Детекция встреч через мониторинг аудио драйвера
│   │   ├── db/
│   │   │   ├── index.js              # Инициализация БД
│   │   │   ├── migrate.js            # Миграции SQLite
│   │   │   ├── repository.js         # Data access layer
│   │   │   └── bootstrap.js          # Начальные данные
│   │   ├── clients/
│   │   │   └── ktalkClient.js        # Интеграция с KTalk API
│   │   ├── ktalk/
│   │   │   └── ktalkMeetingResolver.js  # Привязка встреч к событиям KTalk
│   │   ├── config/
│   │   │   └── configManager.js      # Управление настройками
│   │   └── tray.js                   # Системный трей
│   ├── renderer/
│   │   ├── chronicle-dist/           # Встроенный React UI (билд)
│   │   ├── ai-settings.html          # Настройки AI саммаризации
│   │   ├── ktalk-settings.html       # Настройки KTalk интеграции
│   │   ├── meetings.html             # Окно истории встреч
│   │   └── folder-select.html        # Диалог выбора папки
│   └── shared/
│       └── ffmpegUtils.js            # Утилиты для обработки аудио
├── call-chronicle-box-main/          # Интерфейс Истории встреч - React UI (исходники) 
│   ├── src/
│   │   ├── components/               # UI компоненты (shadcn/ui)
│   │   ├── pages/                    # Страницы приложения
│   │   ├── hooks/                    # React hooks для IPC
│   │   ├── types/                    # TypeScript типы
│   │   ├── lib/                      # Утилиты и вспомогательные функции
│   │   ├── App.tsx                   # Главный компонент
│   │   └── main.tsx                  # Точка входа
│   └── package.json
├── extra/                            # Swift helpers и утилиты
│   ├── devicewatcher/                # Мониторинг изменений аудио-устройств
│   ├── audiowatcher/                 # Мониторинг clientCount драйвера
│   ├── audiorecorder/                # Низкоуровневая запись (AVAudioEngine)
│   ├── driftcorrection/              # Компенсация clock drift
│   ├── meeting-simulator/            # Инструмент для тестирования встреч
│   └── whisper-cpp/                  # Whisper.cpp модели и бинарники
├── drivers/
│   └── PulseCheckAudio.driver/       # Собственный виртуальный аудиодрайвер
└── build/                            # Скрипты сборки, entitlements, notarization
```

### Workflow

1. **Установка драйвера** - пользователь устанавливает `PulseCheckAudio.driver`
2. **Автозапуск** - приложение запускается в системном трее
3. **Создание Aggregate** - `devicewatcher` создает Aggregate устройство (микрофон + системный звук)
4. **Детекция встречи** - `audiowatcher` отслеживает `clientCount` драйвера → встреча началась
5. **Автозапись** - запись автоматически стартует в `~/Recordings/audio/`
6. **Автотранскрибация** - `audioTranscriber` мониторит папку → Whisper.cpp → TXT + JSON
7. **AI-саммаризация** - `transcriptionSummarizer` → OpenRouter API → сохранение в БД
8. **Интеграция с KTalk** - привязка к событию календаря, определение участников
9. **UI** - пользователь открывает окно встреч → видит историю, аудио, транскрипты, саммари

## Системные требования

### macOS
- **Минимальная версия:** macOS 11.0 (Big Sur) или новее
- **Процессор:** Apple Silicon (M1/M2/M3/M4) или Intel x86_64
- **ОЗУ:** 16 GB рекомендуется (для Whisper.cpp large-v3-turbo)
- **Место на диске:**
  - ~500 MB для приложения
  - ~3 GB для моделей Whisper.cpp (автозагрузка из HuggingFace)
  - Место для записей (зависит от интенсивности использования)

### Обязательные компоненты
- **PulseCheck Audio Driver** - виртуальный аудиодрайвер (входит в комплект)
- **Xcode Command Line Tools** - для сборки нативных компонентов

### Разрешения
Приложению требуются следующие разрешения:
- **Микрофон** - для записи вашего голоса
- **Файлы** - для сохранения записей в Documents/Downloads

## Установка и запуск

После установки драйвера:
1. Откройте `.dmg` файл с приложением
2. Переместите `PulseCheck one2one.app` в `/Applications`
3. Запустите приложение
4. Приложение запросит доступ на установку драйвера PulseCheckAudio, обязательно установите его
5. Предоставьте необходимые разрешения (Микрофон, Файлы, Accessibility)
6. Приложение появится в системном трее

## Использование

### Первый запуск

1. **Настройка KTalk** (опционально):
   - Откройте Settings → KTalk
   - Введите session token для интеграции с календарем
   - Встречи будут автоматически привязываться к событиям календаря

2. **Настройка AI** (опционально):
   - Откройте Settings → AI
   - Введите OpenRouter API key (или URL локального LLM)
   - Выберите модель для саммаризации
   - Настройте промпты (есть шаблоны для 1-on-1 и групповых встреч)

### Автоматическая запись

**Система работает полностью автоматически:**

1. Запустите приложение "PulseCheck one2one"
2. В приложении для звонков (Zoom, Google Meet, KTalk, Slack, etc.) выбирете микрофон и динамики PulseCheck 
3. Начните звонок в любом приложении (Zoom, Google Meet, KTalk, Slack, etc.)
4. `audiowatcher` детектирует активность драйвера → запись начинается
5. Во время записи в трее изменится иконка приложения
6. Завершите звонок → запись автоматически останавливается
7. Транскрибация и саммаризация происходят в фоне

**Ручная запись:**

Если автодетекция не сработала (редко):
1. Клик по иконке трея → "Начать запись"
2. После завершения встречи → "Остановить запись"

### Просмотр истории встреч

1. Клик по иконке трея → "Открыть окно встреч"
2. UI показывает:
   - **Список встреч** с датами, участниками, длительностью
   - **Фильтры** по датам, участникам, источнику (KTalk/Manual)
   - **Аудиоплеер** для прослушивания записи
   - **Транскрипт** с временными метками
   - **AI-саммари** (кратко, действия, обсуждения, решения)
   - **Чек-листы** для оценки встречи по критериям

## Технологический стек

### Core
- **Electron** 31.3.0 (Chromium + Node.js 22)
- **better-sqlite3** - встроенная БД для метаданных
- **chokidar** - мониторинг файловой системы
- **axios** - HTTP клиент для API интеграций
- **electron-log** - ротируемые логи

### Audio/Video
- **ffmpeg-static** - обработка аудио (конвертация, нормализация)
- **Whisper.cpp** (large-v3-turbo) - локальная транскрибация
- **Silero VAD v5.1.2** - детекция голоса для умного чанкирования
- **Собственный Swift audiorecorder** (AVAudioEngine) - низкоуровневая запись
- **PulseCheck Audio Driver** - виртуальный драйвер для перехвата системного звука

### AI
- **OpenRouter API** (или любой OpenAI-совместимый endpoint)
- Поддержка кастомных промптов для разных типов встреч
- Умное чанкирование для больших файлов (35 сек с overlap)
- Защита от зацикливания модели (repetition guard)

### UI
- **React** 18 + **TypeScript**
- **Vite** - сборщик
- **shadcn/ui** - библиотека UI компонентов
- **Tailwind CSS** - стилизация
- **Tanstack Query** - управление состоянием и IPC запросами
- **Lucide React** - иконки

### Testing
- **Playwright** - E2E тестирование Electron приложения
- Глобальная переменная `global.__app` для доступа к API из тестов
- Моки для KTalk клиента

## Компоненты и модули

### Swift Helpers (extra/)

#### devicewatcher
- **Назначение:** Мониторинг изменений дефолтных аудио-устройств
- **Функции:**
  - Автоматическое создание Aggregate устройства (микрофон + системный звук)
  - Детекция изменений дефолтного устройства ввода/вывода
  - Поддержка Multi-Output устройств
- **Протокол:** v2.0 JSON schema (события: `default-input-changed`, `aggregate-created`, etc.)

#### audiowatcher
- **Назначение:** Мониторинг `clientCount` драйвера PulseCheck Audio
- **Функции:**
  - Детекция начала встречи (clientCount 0→1)
  - Детекция окончания встречи (clientCount >0→0 или >1→1)
- **Использование:** Основной сигнал для автоматической записи

#### audiorecorder
- **Назначение:** Низкоуровневая запись через AVAudioEngine
- **Функции:**
  - Запись с заданного устройства в WAV
  - Поддержка стерео/моно
  - Настраиваемый sample rate

#### driftcorrection (driftctl)
- **Назначение:** Компенсация clock drift в Aggregate устройствах
- **Функции:**
  - Мониторинг разницы времени между устройствами
  - Автоматическая коррекция через перезапуск устройства

#### meeting-simulator
- **Назначение:** Инструмент для тестирования и симуляции встреч
- **Функции:**
  - Воспроизведение аудио через PulseCheck драйвер
  - Симуляция встреч для проверки автоматической записи
  - Анализ записанных файлов

### Интеграции

#### KTalk Client (src/main/clients/ktalkClient.js)
- **Session-based авторизация** через KTalk API
- **Получение календаря встреч** с фильтрацией по датам
- **Автоматическая привязка записи** к событию календаря
- **Определение участников:**
  - Organizer (организатор)
  - Required/optional attendees
  - Online users (кто реально присутствовал)
- **Поддержка приватных и повторяющихся встреч**

## База данных

### Схема (SQLite v3)

#### meetings
```sql
CREATE TABLE meetings (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  source TEXT NOT NULL,           -- 'ktalk', 'manual', 'zoom'
  ktalk_event_id TEXT,             -- ID события из KTalk календаря
  title TEXT,
  started_at_utc TEXT,
  ended_at_utc TEXT,
  audio_path TEXT,
  transcript_path TEXT,
  created_at_utc TEXT,
  updated_at_utc TEXT
)
```

#### participants
```sql
CREATE TABLE participants (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  email TEXT UNIQUE NOT NULL,
  name TEXT
)
```

#### meeting_participants
```sql
CREATE TABLE meeting_participants (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  meeting_id INTEGER NOT NULL,
  participant_id INTEGER NOT NULL,
  is_self BOOLEAN DEFAULT 0,       -- true если это вы
  role TEXT,                       -- 'organizer', 'required', 'optional'
  FOREIGN KEY (meeting_id) REFERENCES meetings(id),
  FOREIGN KEY (participant_id) REFERENCES participants(id)
)
```

#### summaries
```sql
CREATE TABLE summaries (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  meeting_id INTEGER NOT NULL,
  model TEXT,                      -- 'gpt-4o', 'claude-3-5-sonnet', etc.
  prompt_version TEXT,
  summary_text TEXT,
  summary_path TEXT,               -- DEPRECATED (теперь только в БД)
  created_at_utc TEXT,
  updated_at_utc TEXT,
  FOREIGN KEY (meeting_id) REFERENCES meetings(id)
)
```

#### checklist_templates
```sql
CREATE TABLE checklist_templates (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  version TEXT,
  items_json TEXT NOT NULL,        -- JSON массив пунктов чек-листа
  created_at_utc TEXT,
  updated_at_utc TEXT
)
```

#### checklist_runs
```sql
CREATE TABLE checklist_runs (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  meeting_id INTEGER NOT NULL,
  template_id INTEGER NOT NULL,
  model TEXT,
  score_overall REAL,              -- 0..1
  verdict TEXT,                    -- общий вердикт от AI
  result_json TEXT,                -- детальные результаты
  created_at_utc TEXT,
  FOREIGN KEY (meeting_id) REFERENCES meetings(id),
  FOREIGN KEY (template_id) REFERENCES checklist_templates(id)
)
```

### Миграции

Миграции запускаются автоматически:
- При первом запуске приложения
- После `npm install` (через `postinstall` скрипт)
- Вручную: `npm run migrate`

## Тестирование

### E2E тесты (Playwright)

**Запуск:**
```bash
npm run test:e2e          # headless режим
npm run test:e2e:ui       # UI режим с отладкой
```

**Архитектура:**
- Playwright запускает реальное Electron приложение
- Глобальная переменная `global.__app` предоставляет доступ к Main Process API
- Внешние API (KTalk, OpenRouter) мокируются для изоляции тестов
- Реальные сетевые вызовы не используются

**Тесты покрывают:**
- Автозапуск и инициализацию компонентов
- Детекцию встреч через `audiowatcher`
- Создание записей и связывание с KTalk событиями
- Транскрибацию и саммаризацию (с моками)
- UI взаимодействия

### Ручное тестирование

#### Проверка драйвера
```bash
# Список аудиоустройств
system_profiler SPAudioDataType | grep -i pulsecheck

# Проверка Aggregate устройства (создается devicewatcher)
system_profiler SPAudioDataType | grep -A 5 "Aggregate"
```

#### Проверка автозаписи
1. Запустите приложение
2. Начните звонок в Zoom/Meet/KTalk
3. Проверьте логи:
   ```bash
   tail -f ~/.pulsecheck/app.log
   ```
4. Ищите строки:
   ```
   [audiowatcher] clientCount changed: 0 → 1
   [audioRecorder] Recording started
   ```
5. Завершите звонок → проверьте, что запись остановилась

#### Проверка транскрибации
1. После остановки записи проверьте папку:
   ```bash
   ls -lh ~/Recordings/audio/
   ```
2. Дождитесь создания `.txt` и `.json` файлов
3. Проверьте логи:
   ```
   [audioTranscriber] Transcription started for: meeting-2024-01-15.wav
   [audioTranscriber] Transcript saved: meeting-2024-01-15.txt
   ```

#### Проверка саммаризации
1. После транскрибации проверьте логи:
   ```
   [transcriptionSummarizer] Summary generated for meeting ID: 123
   ```
2. Откройте UI → найдите встречу → проверьте наличие саммари

## Разработка

### Установка зависимостей

```bash
npm install
```

Это автоматически:
- Установит npm зависимости
- Скомпилирует нативные аддоны через `node-gyp`
- Выполнит миграции БД

### Запуск в режиме разработки

**Вариант 1: Только приложение**
```bash
npm start
```

**Вариант 2: Приложение + UI в dev режиме** (hot reload)
```bash
npm run dev
```

Это запустит:
- Vite dev server на `http://localhost:8080` для UI
- Electron приложение с `CHRONICLE_DEV_URL=http://localhost:8080`

### Сборка UI

```bash
npm run chronicle:build
```

Билдится в `src/renderer/chronicle-dist/`

### Сборка приложения

```bash
# Development build (быстрая сборка без подписи)
npm run pack

# Production build (с подписью и нотаризацией)
npm run dist

# Production build без подписи (для тестирования)
npm run dist:unsigned

# Universal binary (Intel + Apple Silicon)
npm run dist:universal
```

### Управление версиями

```bash
npm run version:patch    # 4.3.2 → 4.3.3
npm run version:minor    # 4.3.2 → 4.4.0
npm run version:major    # 4.3.2 → 5.0.0

# Версия + сборка одной командой
npm run build:patch
npm run build:minor
npm run build:major
```

## Отладка

### Логи

**Местоположение:**
- macOS: `~/.pulsecheck/app.log`

**Просмотр в реальном времени:**
```bash
tail -f ~/.pulsecheck/app.log
```

**Уровни логирования:**
- `error` - критические ошибки
- `warn` - предупреждения
- `info` - информационные сообщения
- `debug` - детальная отладка

### Developer Tools

В режиме разработки (`npm start`):
- **Main window:** `Cmd+Option+I` → DevTools
- **Renderer консоль** доступна в логах Main процесса

### Очистка кэша и состояния

```bash
# Очистка настроек и БД
rm -rf ~/Library/Application\ Support/pulsecheck-one2one/

# Очистка логов
rm -rf ~/Library/Logs/pulsecheck-one2one/

```

### Полезные команды

```bash
# Список всех аудиоустройств
system_profiler SPAudioDataType

# Проверка работы драйвера
system_profiler SPAudioDataType | grep -i pulsecheck

# Список запущенных процессов
ps aux | grep -E "devicewatcher|audiowatcher|audiorecorder"

# Проверка версии приложения
/Applications/PulseCheck\ one2one.app/Contents/MacOS/PulseCheck\ one2one --version
```

## Известные проблемы

### macOS Permissions

При первом запуске macOS запросит разрешения:
- **Файлы** (Documents, Downloads) - для сохранения записей

**Решение:** Откройте System Settings → Privacy & Security и предоставьте разрешения.

### Драйвер не работает после обновления macOS

После обновления системы драйвер может потребовать переустановки:
```bash
sudo rm -rf /Library/Audio/Plug-Ins/HAL/PulseCheckAudio.driver
sudo cp -R drivers/PulseCheckAudio.driver /Library/Audio/Plug-Ins/HAL/
sudo launchctl kickstart -k system/com.apple.audio.coreaudiod
```

### Aggregate устройство не создается

**Симптомы:** `devicewatcher` не создает Aggregate устройство

**Решение:**
1. Проверьте, что драйвер установлен (см. выше)
2. Перезапустите `devicewatcher`:
   ```bash
   pkill -9 devicewatcher
   # Приложение автоматически перезапустит его
   ```

### Транскрибация слишком медленная

**Причина:** Модель `large-v3-turbo` требует много ресурсов

**Решение:**
1. Используйте Apple Silicon Mac (M1+) для hardware acceleration
2. Или смените модель на меньшую в настройках (в будущих версиях)

### Саммаризация не работает

**Причины:**
- Не настроен OpenRouter API key
- Некорректный endpoint
- Модель не поддерживает нужный формат

**Решение:**
1. Проверьте настройки в Settings → AI
2. Проверьте логи на наличие ошибок API
3. Попробуйте другую модель (gpt-4o, claude-3-5-sonnet, etc.)

### Детекция встречи не работает

**Причины:**
- Драйвер PulseCheck Audio не установлен или не работает
- audiowatcher не запущен

**Решение:**
1. Проверьте, что драйвер установлен: `system_profiler SPAudioDataType | grep -i pulsecheck`
2. Проверьте логи приложения на наличие сообщений от audiowatcher
3. Перезапустите приложение

## Лицензия

Проект разрабатывается для внутреннего использования.

## Контакты

**Автор:** YKey <yuri.krivochurov@gmail.com>

**Issues:** GitHub Issues (если репозиторий открыт)
