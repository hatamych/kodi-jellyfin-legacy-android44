# Kodi 16 Jarvis + Jellyfin for Legacy Android 4.4 (KitKat) Smart TVs & Boxes
### Сборка Kodi 16 + Jellyfin для старых телевизоров и ТВ-приставок на Android 4.4 KitKat

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Platform](https://img.shields.io/badge/Platform-Android%204.4.2%20%2F%204.4.4%20(KitKat)-green.svg)]()
[![Tested on](https://img.shields.io/badge/Tested%20on-Mystery%20MTV--2430LTA2%20(MStar%20MSD6A628)-blue.svg)]()

[English](#english) | [Русский](#русский)

---

<a name="русский"></a>
## 🇷🇺 Русский

### О проекте
Многие умные телевизоры и ТВ-приставки выпуска 2014–2018 годов (например, на чипах MStar MSD6A628, Amlogic S805/S812, Rockchip RK3188/3288 под управлением **Android 4.4 KitKat**) аппаратной начинкой вполне способны воспроизводить видео 720p/1080p, но программно давно заброшены производителями:
* Современный Jellyfin Android TV требует минимум Android 5.0+ (или Android 7.0+);
* Современные версии Kodi (Kodi 17 Krypton — 21 Omega) требуют Android 5.0+ и современную графическую подсистему;
* Стандартные старые версии плагина `jellyfin-kodi` (v0.1.0 – v0.7.0) намертво зависают или падают с ошибками из-за несовместимости с устаревшим Python 2.7 и C++ API Kodi 16 (Jarvis).

В данном репозитории представлена **полностью адаптированная, проверенная в реальном бою связка**:
1. **`kodi-16.1-jarvis-armeabi-v7a.apk`** — стабильная сборка Kodi 16.1 Jarvis для архитектуры ARMv7;
2. **`plugin.video.jellyfin-0.1.0-kodi16-final.zip`** — модифицированный плагин Jellyfin, из которого вычищены все вызовы несовместимого API Kodi 17/18/19, исправлена работа с современным сервером Jellyfin 10.8 / 10.9+, а также добавлено **автоматическое распознавание аппаратных возможностей ТВ (Smart Transcoding)**.

---

### Какие ошибки были исправлены (Changelog):

1. **Автоматическое транскодирование неподдерживаемых кодеков (Smart Codec Profiling):**
   * *Симптом:* При воспроизведении фильмов в 10-битном кодеке **HEVC (H.265 Main 10)** экран заливало зелёными артефактами, видео мерцало и выскакивала ошибка `CRenderManager::WaitForBuffer - timeout waiting for buffer` (переполнение кэша).
   * *Причина:* Древние чипсеты (MStar, Mali-450) аппаратно поддерживают максимум 8-битный H.264 (AVC) 1080p. Однако стандартный плагин заявлял серверу `DirectPlay: Video [All]`, из-за чего Jellyfin отдавал 10-битный HEVC без перекодирования напрямую в слабый видеочип.
   * *Исправление:* В профиль устройства (`playutils.py`) встроена строгая декларация аппаратных возможностей:
     * `DirectPlay` разрешён только для: `h264, mpeg4, mpeg2video, vc1, vp8, mjpeg` (строго 8-бит).
     * Если файл имеет кодек **HEVC (H.265)** или глубину цвета **10-бит**, плагин возвращает статус `TranscodeReasons: VideoCodecNotSupported`. Сервер Jellyfin **на лету перекодирует фильм в 720p/1080p H.264** через HLS-поток (`master.m3u8`), и телевизор воспроизводит его идеально гладко и без зелёных артефактов!

2. **Ошибка смещения структуры базы данных (Database View Column Shift):**
   * *Симптом:* Фильмы синхронизируются, постеры отображаются, но при нажатии «ОК» фильм не запускается (`PlayMedia could not play media: no such action 'playmedia()'`).
   * *Причина:* Плагин добавлял в таблицу `movie` колонку `premiered`. В Kodi 16 представление `movie_view` собирается через `SELECT movie.*`, из-за чего все последующие колонки сместились на +1. C++ движок Kodi читал путь к файлу из поля просмотров (`playCount`), получал пустую строку и сбрасывал воспроизведение.
   * *Исправление:* Представления `movie_view` и `musicvideo_view` пересозданы со строгим порядком колонок, путь к файлу возвращён на законную позицию.

3. **Несовместимые методы `ListItem` и `InfoTagVideo`:**
   * *Симптом:* Падение скрипта с `AttributeError: 'xbmcgui.ListItem' object has no attribute 'setCast'` и `AttributeError: 'xbmc.InfoTagVideo' object has no attribute 'getDbId'`.
   * *Причина:* В коде воспроизведения вызывались API-функции, появившиеся только в Kodi 17+ Krypton.
   * *Исправление:* Все вызовы защищены безопасными обёртками (`hasattr` / `getattr`), что обеспечило 100% совместимость со старым движком Kodi 16.

4. **Отсутствие таблиц в `MyMusic56.db`:**
   * *Симптом:* Синхронизация музыки падала на исполнителях с ошибкой `no such table: role` и `no such column: idRole`.
   * *Исправление:* Добавлены недостающие таблицы и колонки схемы, необходимые плагину для бесшовной интеграции музыкальной библиотеки.

5. **Быстрый вход и современный Jellyfin API:**
   * Убраны устаревшие эндпоинты авторизации, плагин корректно получает токен сессии от современных серверов Jellyfin.

---

### Инструкция по установке:

1. **Установка Kodi:**
   * Скопируйте файл `kodi-16.1-jarvis-armeabi-v7a.apk` на флешку или передайте по ADB:
     ```bash
     adb install -r kodi-16.1-jarvis-armeabi-v7a.apk
     ```
   * Запустите Kodi на телевизоре один раз, чтобы он создал свою рабочую директорию.

2. **Установка плагина Jellyfin:**
   * Скопируйте архив `plugin.video.jellyfin-0.1.0-kodi16-final.zip` в память телевизора (например, в `/sdcard/Download/`).
   * В Kodi зайдите в: **Система (System) ➔ Настройки (Settings) ➔ Дополнения (Add-ons) ➔ Установить из zip-файла (Install from zip file)**.
   * Выберите скачанный `.zip`.
   * При первом запуске плагин предложит ввести IP-адрес вашего сервера Jellyfin (например, `http://192.168.1.100:8096`) и данные пользователя.
   * Выберите режим синхронизации **Add-on** (или Native) и дождитесь синхронизации медиатеки.

---

<a name="english"></a>
## 🇬🇧 English

### Overview
Many Smart TVs and TV boxes released between 2014 and 2018 (powered by chips such as MStar MSD6A628, Amlogic S805/S812, Rockchip RK3188/3288 running **Android 4.4 KitKat**) have adequate hardware to decode 720p/1080p video, but lack modern software support:
* Modern Jellyfin Android TV app requires Android 5.0+ (or Android 7.0+);
* Official Kodi releases (Kodi 17 Krypton through Kodi 21 Omega) dropped Android 4.4 support;
* Standard legacy versions of `jellyfin-kodi` (v0.1.0 – v0.7.0) crash or fail to play items due to incompatible Python 2.7 / Kodi 16 C++ API calls.

This repository provides a **battle-tested, patched bundle**:
1. **`kodi-16.1-jarvis-armeabi-v7a.apk`** — Stable Kodi 16.1 Jarvis build for ARMv7 devices;
2. **`plugin.video.jellyfin-0.1.0-kodi16-final.zip`** — Modified Jellyfin Kodi add-on with backported fixes for Kodi 16, **hardware profile restriction (auto-transcoding of 10-bit HEVC to H.264)**, and full compatibility with modern Jellyfin 10.8 / 10.9+ servers.

---

### Fixed Issues & Technical Details:

1. **Auto-Transcode for Unsupported HEVC / 10-bit Media:**
   * *Issue:* Playing **HEVC (H.265 Main 10)** movies caused green flickering artifacts and `CRenderManager::WaitForBuffer - timeout waiting for buffer` (cache overflow).
   * *Root Cause:* Legacy MStar / Mali-450 hardware decoders only support 8-bit H.264 (AVC). The upstream add-on falsely reported `DirectPlay: Video [All]`, forcing direct streaming of 10-bit HEVC.
   * *Fix:* Configured hardware profile in `playutils.py` restricting `DirectPlay` strictly to 8-bit `h264, mpeg4, mpeg2video, vc1, vp8`. For HEVC / 10-bit media, Jellyfin now triggers on-the-fly server-side transcoding to H.264 HLS (`master.m3u8`).

2. **Database View Column Shift (Unplayable Items Bug):**
   * *Issue:* Movies synchronized and displayed artwork, but clicking «OK» resulted in `PlayMedia could not play media: no such action 'playmedia()'`.
   * *Root Cause:* Adding `premiered` column into the `movie` table caused `SELECT movie.*` in `movie_view` to shift all subsequent column indices by +1. Kodi's C++ core read the file path from `playCount` (which was an integer/NULL), resulting in an empty path error.
   * *Fix:* Recreated `movie_view` and `musicvideo_view` with explicit column ordering.

3. **Incompatible `ListItem` & `InfoTagVideo` Methods:**
   * *Issue:* Python script crash: `AttributeError: 'xbmcgui.ListItem' object has no attribute 'setCast'` and `AttributeError: 'xbmc.InfoTagVideo' object has no attribute 'getDbId'`.
   * *Fix:* Wrapped newer Kodi 17+ API calls with `hasattr()` / `getattr()` fallback checks.

4. **Missing Tables in `MyMusic56.db`:**
   * *Issue:* Music sync failed on artists with `no such table: role` and `no such column: idRole`.
   * *Fix:* Migrated and added required schema elements (`role`, `idRole`, `iUserrating`).

---

### Installation Guide:

1. **Install Kodi APK:**
   * Transfer `kodi-16.1-jarvis-armeabi-v7a.apk` via USB drive or ADB:
     ```bash
     adb install -r kodi-16.1-jarvis-armeabi-v7a.apk
     ```
   * Launch Kodi once to initialize its data folder.

2. **Install Jellyfin Add-on:**
   * Copy `plugin.video.jellyfin-0.1.0-kodi16-final.zip` to your TV (e.g., `/sdcard/Download/`).
   * In Kodi navigate to: **System ➔ Settings ➔ Add-ons ➔ Install from zip file**.
   * Select the zip file.
   * Connect to your Jellyfin server IP and authenticate.

---

### License
GPL v2 / MIT
