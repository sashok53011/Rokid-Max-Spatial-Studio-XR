# Rokid Spatial Studio XR — Полная документация / Vollständige Dokumentation / Full Documentation

> **Языки / Sprachen / Languages:**
> - [Русский (RU)](#русский-ru)
> - [Deutsch (DE)](#deutsch-de)
> - [English (EN)](#english-en)

---

<a id="русский-ru"></a>
## Русский (RU)

### 1. Обзор проекта
**Rokid Spatial Studio XR** — это WebXR/Three.js-приложение и гибридный Android-клиент (через Capacitor) для очков дополненной реальности **Rokid Max (1st Gen)** и совместимых устройств. Приложение объединяет 3D-прототипирование, PBR-материалы, физический движок Rapier3D, 4× 360° пространственные камеры, экспорт/импорт 3D-моделей, модульный Plugin SDK и потоковую передачу IMU-телеметрии на частоте до 400 Гц (через WebHID на ПК и через нативный `UsbManager` на Android).

---

### 2. Полностью реализованные функции

1. **Инициализация WebXR, карта глубины и жесты (`src/index.ts`, `src/SpatialStudioApp.ts`)**
   - Запуск сцены через `xrblocks` (`enableCamera('environment')`, `enableReticles`, `enableHands`, `enableGestures`, `enableXRTransitions`).
   - Распознавание жестов рук: `point` (создаёт физический зонд над прототипом) и `spread` (переключает вкладки 3D-панели управления).
   - Карта глубины среды (`DepthOptions`, переключение видимости depth-меша и якорение прототипа на реальные поверхности по клику/лучу).

2. **Драйвер Rokid Max 1st Gen: WebHID + Android `UsbManager` + Madgwick AHRS (`src/sdk/VertoRokidWebHIDDriver.ts`, `src/sdk/RokidAndroidUsbBridge.ts`, `android-native/RokidUsbPlugin.java`)**
   - Автоматический выбор транспорта: `webhid` в Chrome/Edge на ПК или `android-usb` внутри Android APK.
   - Фильтрация устройств Rokid (`VID 0x04D2`, `PID 0x162F`, `0x162D`, `0x1630`, `0x1631`, `0x1632`, `0x1639`, `0x2002`) и wake-handshake (`SET_REPORT` Feature/Output `0x01`).
   - Декодирование 64-байтных (`0x11`) и 128-байтных (`0x80`) HID-пакетов (гироскоп, акселерометр, температура IMU).
   - Встроенный 6-осевой фильтр **Madgwick AHRS** (`beta = 0.045`), пересчёт в систему координат Three.js, функция `recenter()` и плавный `slerp` камеры в режиме 3DoF (вне WebXR-сессии).
   - Нативный Java-плагин `RokidUsbPlugin.java` для Android: запрос прав USB Host (`PendingIntent`), отслеживание отключения кабеля (`ACTION_USB_DEVICE_DETACHED`), высокоприоритетный поток чтения (`THREAD_PRIORITY_URGENT_AUDIO`), измерение точного аппаратного `dtSec` через `System.nanoTime()` и пакетная передача `imuBatch` (до 4 пакетов / 10 мс в Base64) для 400 Гц без перегрузки WebView.

3. **2D-док подключения и график IMU в реальном времени (`src/ui/RokidConnectOverlay.ts`, `src/ui/RokidImuChart.ts`)**
   - Панель `#rokid-webhid-dock` с кнопками **Connect / Disconnect / Recenter 3DoF** и статусом устройства.
   - Интерактивный линейный график на **Recharts** (`LineChart` + `ResponsiveContainer`, кольцевой буфер 36 точек, ~20 Гц обновление) с переключением каналов: **Euler °** (`Yaw/Pitch/Roll`), **Gyro °/s** (`Gx/Gy/Gz`), **Accel** (`Ax/Ay/Az` м/с²).

4. **Пространственный 3D-интерфейс (`src/ui/SpatialControlDeck.ts`)**
   - Левая карточка (`xb.UICard`) с 4 вкладками: `Proto`, `360°`, `Sync/IO`, `SDK`.
   - Правая карточка с превью камеры (`xb.UIImage`), статусом WebHID/IMU, показателями батареи/температуры/FPS/задержки и кнопками подключения.
   - Нижний HUD (`xb.UIOverlay`) с кнопкой выхода из XR и подсказками.

5. **Фабрика 3D-прототипов и библиотека PBR-материалов (`src/prototypes/PrototypeFactory.ts`, `src/materials/MaterialLibrary.ts`)**
   - 3 многокомпонентных прототипа (`gimbal` — оптический подвес, `turbine` — микротурбина, `canopy` — архитектурный навес) с разнесённым видом (`Exploded View`) и анимацией подвижных узлов.
   - 6 физических материалов (`titanium`, `carbon`, `cobalt`, `sapphire`, `ceramic`, `gold`) с процедурными bump/albedo-текстурами, `RoomEnvironment` PMREM IBL и автодетектом программного WebGL (`detectSoftwareWebGL`) для упрощения рефракции.

6. **Физическая песочница Rapier3D (`src/physics/PhysicsSandbox.ts`)**
   - Интеграция `@dimforge/rapier3d-simd-compat`, пул из 24 динамических сферических зондов с CCD (`triggerStressBurst`, `spawnProbe`), коллайдеры пола и пьедестала, переключение гравитации (`-9.81 м/с²` ↔ `0 м/с²` Zero-G).

7. **Универсальный импорт/экспорт 3D-моделей (`src/io/UniversalModelIO.ts`)**
   - Экспорт текущей сборки в `.glb`, `.obj` и бинарный `.stl` с мгновенным скачиванием.
   - Импорт `.glb`, `.gltf`, `.obj`, `.stl` с авто-масштабированием до `0.36 м`, центрированием основания в `y = 0` и генерацией `explodeVector` для разнесённого вида.

8. **Модульный Plugin SDK (`src/sdk/RokidPluginAPI.ts`)**
   - Публичный глобальный интерфейс `window.RokidSpatialAPI` (`registerPlugin`, `togglePlugin`, `getPlugins`, `getTelemetry`, `connectRokidWebHID`, `recenterRokidWebHID`).
   - 2 рабочих встроенных плагина: `stress-heatmap` (динамическая тепловая/нагрузочная подсветка поверхностей) и `spatial-probe` (вращающееся кольцо габаритов `0.5 м`).

---

### 3. Частично реализованные и запланированные функции

| Функция | Текущий статус | Что сделано vs. чего не хватает |
| :--- | :--- | :--- |
| **Сборка Android APK** | Подготовлен проект и плагин | Написаны `capacitor.config.ts`, `RokidUsbPlugin.java`, `MainActivity.java`, `rokid_usb_device_filter.xml` и скрипт `scripts/setup-android-usb.mjs`. Сам `.apk` не собран, так как в облачном контейнере отсутствуют JDK, Gradle и Android SDK. |
| **3DoF / 6DoF трекинг и стереовывод Rokid** | Частично | Работает 3DoF по 6-осевому Madgwick (без магнитометра и без калибровки смещения гироскопа — возможен медленный дрейф по Yaw). Нет нативного SBS-разделения экрана `3840×1080` для очков вне WebXR и нет слияния ARCore 6DoF с IMU. |
| **Профили энергопотребления (`eco` / `balanced` / `performance`)** | Частично | Реально переключают `renderer.setPixelRatio`, `shadowMap.enabled`, `castShadow` и частоту обновления `depthMesh.colliderUpdateFps`. Поле `targetFps` (60/90/120 Гц) пока только отображается в UI и **не ограничивает** частоту цикла `requestAnimationFrame`. |
| **Оптимизация экспорта/импорта в Android APK** | Частично | В браузере используется `<a download>` и `<input type="file">`. В Android WebView скачивание `blob:`-ссылок без нативных плагинов может не сохранять файлы в «Загрузки». |
| **Плагины SDK сторонних разработчиков** | Частично | Есть runtime API регистрации плагинов, но нет изолированной песочницы, загрузки внешних плагинов по URL/манифесту и официальных биндингов Rokid UXR/CXR `.aar`. |
| **Управление аппаратными функциями Rokid Max** | Запланировано | Не реализованы вендорные USB control-команды переключения яркости OLED, режима 2D/3D (SBS), звука USB-Audio и потока UVC-камеры (если подключена внешняя камера). |

---

### 4. Заглушки в коде и чем их заменить в продакшене

1. **Локальная «облачная» синхронизация и бот-коллаборатор (`src/sync/CloudSyncManager.ts`)**
   - **Где заглушка:**
     - `BroadcastChannel('rokid-spatial-studio-v1')` (строки 43–46, 85–130) работает **только между вкладками одного браузера на одном устройстве**, а не по сети.
     - `toggleSimulatedCollaborator()` (`'sim-peer-01'`, строки 62–79, 266–290) создаёт виртуального пользователя *«Elena · Rokid Max #2»*, который движется по синусоиде `Math.sin`/`Math.cos`.
   - **Чем заменить в продакшене:**
     - Подключить WebSocket-сервер (`wss://`), WebRTC DataChannels, LiveKit или Firestore/Colyseus/Yjs-комнаты; синхронизировать общую систему координат через Cloud Anchors / QR-маркер и удалить `'sim-peer-01'`.

2. **«Облачный гибридный рендеринг» (`src/sdk/RokidPluginAPI.ts`, строки 237–249, 278, 322–323)**
   - **Где заглушка:**
     - `setRenderPipeline('cloud')` локально меняет только `scene.environmentIntensity` (`0.95 → 1.45`) и `renderer.toneMappingExposure` (`1.02 → 1.18`), а также умножает расчётную мощность на `0.68` (`cloudDiscount`). Никакого удалённого GPU-сервера нет.
   - **Чем заменить в продакшене:**
     - Интегрировать реальный WebRTC Pixel Streaming / NVIDIA CloudXR / удалённый WebGPU-рендерер, передающий на сервер позу 400 Гц и принимающий стерео-видеопоток (H.265/AV1).

3. **Телеметрия батареи, ватт и задержки (`src/sdk/RokidPluginAPI.ts`, строки 14–45, 69–70, 277–286, 312–324)**
   - **Где заглушка:**
     - `batteryPercent` стартует с захардкоженных `94.0%` и линейно уменьшается по формуле `effectiveWatts * dt / 180`.
     - `thermalC` без подключённого USB HID `0x80` демпфируется к синтетическому значению `31.5 + effectiveWatts * 1.45`.
     - `estimatedWatts`, `batteryHours` и `latencyMs` (`2.3 / 4.2 / 11.4 ms`) — табличные константы.
   - **Чем заменить в продакшене:**
     - В браузере использовать `navigator.getBattery()`, а в Android APK добавить метод в нативный Capacitor-плагин с чтением `android.os.BatteryManager` (`BATTERY_PROPERTY_CAPACITY`, `BATTERY_PROPERTY_CURRENT_NOW`, `EXTRA_TEMPERATURE`) и реальным замером round-trip frame latency.

4. **Процедурные 360°-панорамы вместо реальных камер (`src/environment/Panorama360Manager.ts`, строки 96–168)**
   - **Где заглушка:**
     - `createEquirectangularTexture()` рисует на `HTMLCanvasElement` (`1024×512`) векторный градиент, сетку и подписи сторон света вместо фото или видеопотока с 4 камер.
   - **Чем заменить в продакшене:**
     - Загружать 8K эквиректангулярные `.hdr`/`.webp` панорамы или подключить 4 живых `THREE.VideoTexture` через WebRTC/RTSP-бридж от 360°-камер (Insta360 / Ricoh Theta / Unity/Unreal).

5. **Упрощённый физический коллайдер прототипа и счётчик столкновений (`src/physics/PhysicsSandbox.ts`, строки 60–68, 126)**
   - **Где заглушка:**
     - Коллайдер для любого 3D-прототипа и импортированной модели задан одним статическим кубом `RAPIER.ColliderDesc.cuboid(0.22, 0.18, 0.22)`.
     - `lastCollisionCount` увеличивается при спавне шара, а не при физическом ударе.
   - **Чем заменить в продакшене:**
     - Строить `RAPIER.ColliderDesc.convexHull(...)` или `trimesh(...)` из геометрии активного меша при его загрузке и считывать события контактов через `RAPIER.EventQueue.drainCollisionEvents`.

6. **Процедурные PBR-текстуры и окружение (`src/materials/MaterialLibrary.ts`, строки 71–172)**
   - **Где заглушка:**
     - Используется `RoomEnvironment` и рисованные в Canvas 2D карты шероховатости/рельефа (`128×128` / `256×256`), без внешних фотореалистичных текстур.
   - **Чем заменить в продакшене:**
     - Подключить `KTX2Loader` + `BasisTextureLoader` для PBR-наборов карт (`map`, `normalMap`, `roughnessMap`, `aoMap`) и `RGBELoader`/`EXRLoader` для студийных HDRI.

7. **Пустая заглушка превью камеры (`src/ui/SpatialControlDeck.ts`, строки 39–40, 667–668)**
   - **Где заглушка:**
     - `EMPTY_IMAGE` (`1×1` GIF) и текст `"Stream: Passthrough Simulated"`, если браузер не нашёл веб-камеру.
   - **Чем заменить в продакшене:**
     - Подавать кадры с реальной USB/UVC-камеры или нативного Android `Camera2` / `CameraX` API.

8. **Синтаксическая/экспортная обвязка для Android (`src/io/UniversalModelIO.ts`, строки 173–182)**
   - **Где заглушка:**
     - `triggerDownload()` использует `<a download>` (веб-реализация).
   - **Чем заменить в продакшене:**
     - При `Capacitor.isNativePlatform()` сохранять файлы через `@capacitor/filesystem` и открывать диалог экспорта через `@capacitor/share`.

---

<a id="deutsch-de"></a>
## Deutsch (DE)

### 1. Projektübersicht
**Rokid Spatial Studio XR** ist eine WebXR-/Three.js-Anwendung und ein hybrider Android-Client (über Capacitor) für **Rokid Max (1. Generation)** AR-Brillen. Sie kombiniert 3D-Prototyping, PBR-Materialien, Rapier3D-Physiksimulationen, 4× 360°-Raumkameras, universellen 3D-Dateiimport/-export, eine modulare Plugin-API sowie Live-IMU-Streaming mit bis zu 400 Hz (WebHID am Desktop bzw. natives Android `UsbManager` in der APK).

---

### 2. Vollständig umgesetzte Funktionen

1. **WebXR-Initialisierung, Tiefenmapping & Gestensteuerung (`src/index.ts`, `src/SpatialStudioApp.ts`)**
   - Konfiguration über `xrblocks` (`enableCamera('environment')`, Reticles, Hand-Tracking, Gestenerkennung `point` & `spread`, AR/VR-Übergänge).
   - Echtzeit-Tiefenkollision (`DepthOptions`, ein-/ausblendbares Depth-Mesh, Platzieren von Prototypen auf realen Oberflächen per Klick/Ray).
   - Automatisches 3DoF-Head-Tracking per Quaternion-`slerp` aus `VertoRokidWebHIDDriver.orientation`.

2. **Rokid Max (1. Gen.) Treiber: WebHID + Android `UsbManager` + Madgwick AHRS (`src/sdk/VertoRokidWebHIDDriver.ts`, `src/sdk/RokidAndroidUsbBridge.ts`, `android-native/RokidUsbPlugin.java`)**
   - Automatische Transportwahl: `webhid` in Desktop-Chrome/Edge oder `android-usb` innerhalb einer Capacitor-Android-APK.
   - Geräteerkennung für Rokid (`VID 0x04D2`, `PID 0x162F`, `0x162D`, `0x1630`, `0x1631`, `0x1632`, `0x1639`, `0x2002`) inkl. VertoXR Wake-Handshake (`SET_REPORT` Feature/Output `0x01`).
   - Dekodierung von 64-Byte- (`0x11`) und 128-Byte-HID-Paketen (`0x80`, inkl. IMU-Chip-Temperatur).
   - 6-Achsen-**Madgwick-AHRS-Filter** (`beta = 0.045`), Koordinaten-Remapping für Three.js, `recenter()` und Paketratenmessung (`imuRateHz`).
   - Natives Android-Plugin `RokidUsbPlugin.java`: USB-Host-Berechtigung (`PendingIntent`), Detach-Erkennung (`ACTION_USB_DEVICE_DETACHED`), Reader-Thread mit `THREAD_PRIORITY_URGENT_AUDIO`, präzises `dtSec` über `System.nanoTime()` sowie `imuBatch`-Bündelung (4 Pakete / 10 ms als Base64) für volle 400 Hz ohne WebView-Überlastung.

3. **2D-Verbindungsdock & Live-IMU-Liniendiagramm (`src/ui/RokidConnectOverlay.ts`, `src/ui/RokidImuChart.ts`)**
   - Overlay `#rokid-webhid-dock` mit **Connect / Disconnect / Recenter 3DoF**-Buttons.
   - Echtzeit-**Recharts** `LineChart` (36-Punkte-Puffer, ~20 Hz Refresh) mit Kanalumschaltung: **Euler °** (`Yaw/Pitch/Roll`), **Gyro °/s** (`Gx/Gy/Gz`), **Accel** (`Ax/Ay/Az` m/s²).

4. **Räumliche 3D-Benutzeroberfläche (`src/ui/SpatialControlDeck.ts`)**
   - Linkes `xb.UICard` mit 4 Tabs (`Proto`, `360°`, `Sync/IO`, `SDK`), rechtes Telemetrie-/Kameravorschau-Panel und unteres `xb.UIOverlay`.

5. **3D-Prototypen & PBR-Materialbibliothek (`src/prototypes/PrototypeFactory.ts`, `src/materials/MaterialLibrary.ts`)**
   - 3 mehrteilige Baugruppen (`gimbal`, `turbine`, `canopy`) mit Explosionsansicht (`Exploded View`) und Teilanimation.
   - 6 PBR-Materialpresets (`titanium`, `carbon`, `cobalt`, `sapphire`, `ceramic`, `gold`) inkl. prozeduraler Bump-/Farbtexturen, `RoomEnvironment` PMREM-IBL und Software-WebGL-Erkennung (`detectSoftwareWebGL`).

6. **Rapier3D-Physik-Sandbox (`src/physics/PhysicsSandbox.ts`)**
   - `@dimforge/rapier3d-simd-compat` mit 24 gepoolten CCD-Kugelsonden (`triggerStressBurst`), Boden-/Podest-Kollidern und Erd- vs. Schwerelosigkeitsmodus (`Zero-G`).

7. **Universeller 3D-Import & -Export (`src/io/UniversalModelIO.ts`)**
   - Export nach `.glb`, `.obj` und binärem `.stl`; Import von `.glb`, `.gltf`, `.obj`, `.stl` mit automatischer Skalierung (`0.36 m`), Bodenzentrierung und Explosionsvektoren.

8. **Modulares Plugin-SDK (`src/sdk/RokidPluginAPI.ts`)**
   - Globale API `window.RokidSpatialAPI` sowie zwei integrierte Plugins: `stress-heatmap` und `spatial-probe`.

---

### 3. Teilweise umgesetzte und geplante Funktionen

| Funktion | Status | Implementiert vs. fehlend |
| :--- | :--- | :--- |
| **APK-Kompilierung** | Vorbereitet | `capacitor.config.ts`, `RokidUsbPlugin.java`, `MainActivity.java`, `rokid_usb_device_filter.xml` und `scripts/setup-android-usb.mjs` sind fertig. Die `.apk`-Datei wurde noch nicht gebaut, da in dieser Cloud-Umgebung kein JDK, Gradle oder Android SDK installiert ist. |
| **3DoF/6DoF-Fusion & Stereo-SBS-Ausgabe** | Teilweise | 6-Achsen-3DoF läuft ohne Magnetometer/Gyro-Bias-Kalibrierung (leichter Yaw-Drift möglich). Es fehlen eine native `3840×1080` Side-by-Side-Stereo-Projektion außerhalb von WebXR sowie ARCore-6DoF-Fusion. |
| **Energieprofile (`eco` / `balanced` / `performance`)** | Teilweise | Steuern real `pixelRatio`, Schatten und `colliderUpdateFps`. `targetFps` (60/90/120 Hz) wird jedoch nur angezeigt und drosselt den Render-Loop nicht aktiv. |
| **Dateidownload in Android APK** | Teilweise | Nutzt im Browser `<a download>`, was in Android WebView für `blob:`-URLs eingeschränkt sein kann. |
| **Erweiterte Rokid-Hardwaresteuerung** | Geplant | Proprietäre USB-Control-Transfers für OLED-Helligkeit, 2D/3D-SBS-Umschaltung, USB-Audio und UVC-Kamerazugriff fehlen noch. |

---

### 4. Verbliebene Platzhalter (Stubs) & Produktions-Ersatz

1. **Lokale „Cloud“-Synchronisation & simulierter Kollege (`src/sync/CloudSyncManager.ts`)**
   - **Stub:** `BroadcastChannel('rokid-spatial-studio-v1')` (Zeilen 43–46) synchronisiert nur zwischen Tabs *desselben* Browsers. `toggleSimulatedCollaborator()` (`'sim-peer-01'`, *„Elena · Rokid Max #2“*, Zeilen 62–79, 266–290) bewegt ein Fake-Headset per Sinuswelle.
   - **Ersatz:** WebSockets (`wss://`), WebRTC DataChannels, LiveKit oder Firestore/Colyseus/Yjs mit gemeinsamer räumlicher Verankerung (Cloud Anchors / QR-Marker); `'sim-peer-01'` entfernen.

2. **„Cloud Hybrid Rendering Pipeline“ (`src/sdk/RokidPluginAPI.ts`, Zeilen 237–249, 278, 322–323)**
   - **Stub:** `setRenderPipeline('cloud')` erhöht lediglich `scene.environmentIntensity` (`0.95 → 1.45`) und `toneMappingExposure` (`1.02 → 1.18`) mit einem fiktiven `0.68`-Leistungsfaktor.
   - **Ersatz:** Echtes Remote-GPU-Streaming über WebRTC Pixel Streaming, NVIDIA CloudXR oder einen Remote-WebGPU-Server.

3. **Simulierte Akku-, Leistungs- und Latenzwerte (`src/sdk/RokidPluginAPI.ts`, Zeilen 14–45, 69–70, 277–286, 312–324)**
   - **Stub:** `batteryPercent` beginnt fest bei `94.0%` und sinkt synthetisch; `thermalC` nähert sich ohne USB-HID-`0x80`-Sensorwert einer Formel; `powerWatts` und `latencyMs` sind statische Schätzwerte.
   - **Ersatz:** `navigator.getBattery()` im Web bzw. Android `BatteryManager` über ein natives Capacitor-Plugin sowie echte Frame-Time-/RTT-Messung.

4. **Prozedurale 360°-Kameratexturen (`src/environment/Panorama360Manager.ts`, Zeilen 96–168)**
   - **Stub:** `createEquirectangularTexture()` zeichnet ein synthetisches `1024×512`-Canvas mit Gitternetz statt echter 360°-Kamerabilder.
   - **Ersatz:** 4× `THREE.VideoTexture` über WebRTC/RTSP-Livestreams (z. B. Insta360 / Ricoh Theta) oder hochauflösende 8K-HDR-Panoramen.

5. **Statischer Würfel-Kollider & Spawn-Zähler (`src/physics/PhysicsSandbox.ts`, Zeilen 60–68, 126)**
   - **Stub:** Der Prototyp-Kollider ist ein fester `cuboid(0.22, 0.18, 0.22)`; `lastCollisionCount` zählt nur gespawnte Kugeln.
   - **Ersatz:** Dynamische `convexHull`- oder `trimesh`-Kollider aus den geladenen Mesh-Vertices sowie `RAPIER.EventQueue.drainCollisionEvents`.

6. **Prozedurale Canvas-PBR-Texturen (`src/materials/MaterialLibrary.ts`, Zeilen 71–172)**
   - **Stub:** `RoomEnvironment` und kleine generierte 2D-Canvas-Maps statt gescannter PBR-Assets.
   - **Ersatz:** `KTX2Loader` (Normal-/Roughness-/AO-Maps) und `RGBELoader`/`EXRLoader` für echte Studio-HDRIs.

7. **Kameravorschau-Platzhalter (`src/ui/SpatialControlDeck.ts`, Zeilen 39–40, 667–668)**
   - **Stub:** `EMPTY_IMAGE` (`1×1` GIF) und Hinweis `"Stream: Passthrough Simulated"`, falls keine Kamera gefunden wird.
   - **Ersatz:** Echter USB-/UVC-Stream bzw. Android `CameraX`/`Camera2` Video-Feed.

8. **Web-Download-Mechanismus in APK (`src/io/UniversalModelIO.ts`, Zeilen 173–182)**
   - **Stub:** `<a download>` über `URL.createObjectURL`.
   - **Ersatz:** `@capacitor/filesystem` + `@capacitor/share` unter Android.

---

<a id="english-en"></a>
## English (EN)

### 1. Project Overview
**Rokid Spatial Studio XR** is a WebXR/Three.js spatial workspace and hybrid Capacitor Android client built for **Rokid Max (1st Gen)** AR glasses. It combines multi-part 3D virtual prototyping, photorealistic PBR materials, Rapier3D real-time physics, four 360° architectural walkthrough cameras, universal 3D model import/export, a runtime Plugin SDK, and 400 Hz USB-C IMU streaming (via WebHID on desktop Chrome/Edge and native Android `UsbManager` inside an APK).

---

### 2. Fully Implemented Features

1. **WebXR Session, Environment Depth & Gestures (`src/index.ts`, `src/SpatialStudioApp.ts`)**
   - Bootstraps `xrblocks` with environment camera streaming, depth-projected reticles, hand tracking, `point`/`spread` gestures, AR↔VR transitions, and toggleable depth mesh visualization (`colliderUpdateFps`).
   - Automatically slerps the Three.js camera quaternion to `VertoRokidWebHIDDriver.orientation` when streaming IMU data outside a WebXR session.

2. **Rokid Max 1st Gen Driver: WebHID + Android `UsbManager` + Madgwick AHRS (`src/sdk/VertoRokidWebHIDDriver.ts`, `src/sdk/RokidAndroidUsbBridge.ts`, `android-native/RokidUsbPlugin.java`)**
   - Automatically selects `webhid` (desktop Chrome/Edge) or `android-usb` (Capacitor Android APK).
   - Matches Rokid `VID 0x04D2` and Rokid Max PIDs (`0x162F`, `0x162D`, `0x1630`, `0x1631`, `0x1632`, `0x1639`, `0x2002`) and sends the VertoXR wake handshake (`SET_REPORT` Feature/Output `0x01`).
   - Parses both 64-byte (`0x11`) and 128-byte (`0x80`) HID IMU reports (gyroscope, accelerometer, and on-chip temperature).
   - Runs a 6-axis **Madgwick AHRS** filter (`beta = 0.045`), optical-to-Three.js frame remapping, `recenter()`, and live packet-rate measurement (`imuRateHz`).
   - Native Android Java plugin (`RokidUsbPlugin.java`): requests USB host permission (`PendingIntent`), listens for `ACTION_USB_DEVICE_DETACHED`, reads packets on an urgent-audio-priority thread, computes nanosecond `dtSec` intervals, and emits Base64 `imuBatch` events (up to 4 packets / 10 ms) to preserve 400 Hz precision without saturating the WebView bridge.

3. **2D Connection Dock & Real-Time Recharts IMU Chart (`src/ui/RokidConnectOverlay.ts`, `src/ui/RokidImuChart.ts`)**
   - Floating `#rokid-webhid-dock` overlay with **Connect / Disconnect / Recenter 3DoF** controls.
   - Live **Recharts** `LineChart` (`ResponsiveContainer`, 36-sample rolling buffer throttled at ~20 Hz) with channel switcher: **Euler °** (`Yaw/Pitch/Roll`), **Gyro °/s** (`Gx/Gy/Gz`), and **Accel** (`Ax/Ay/Az` m/s²).

4. **3D Spatial Control Deck & Telemetry UI (`src/ui/SpatialControlDeck.ts`)**
   - Left `xb.UICard` (`Proto`, `360°`, `Sync/IO`, `SDK` tabs), right sensor/camera telemetry card (`xb.UIImage` + metrics + WebHID buttons), and bottom `xb.UIOverlay`.

5. **3D Prototyping & PBR Material Library (`src/prototypes/PrototypeFactory.ts`, `src/materials/MaterialLibrary.ts`)**
   - 3 composite prototypes (`gimbal`, `turbine`, `canopy`) with animated sub-parts and smooth `Exploded View` separation.
   - 6 procedural `MeshPhysicalMaterial` presets (`titanium`, `carbon`, `cobalt`, `sapphire`, `ceramic`, `gold`), `RoomEnvironment` PMREM studio lighting, and `detectSoftwareWebGL()` fallback.

6. **Rapier3D Physics Sandbox (`src/physics/PhysicsSandbox.ts`)**
   - Pooled 24 dynamic CCD spheres (`triggerStressBurst`, `spawnProbe`), ground & movable pedestal colliders, and Earth (`-9.81 m/s²`) vs. `Zero-G` gravity toggle.

7. **Universal 3D File I/O (`src/io/UniversalModelIO.ts`)**
   - Exports to `.glb`, `.obj`, and binary `.stl`; imports `.glb`, `.gltf`, `.obj`, `.stl` with auto-scaling (`0.36 m`), floor alignment, and generated `explodeVector` metadata.

8. **Extensible Plugin SDK (`src/sdk/RokidPluginAPI.ts`)**
   - Exposes `window.RokidSpatialAPI` (`registerPlugin`, `togglePlugin`, `getPlugins`, `getTelemetry`, `connectRokidWebHID`, `recenterRokidWebHID`) with two built-in plugins (`stress-heatmap`, `spatial-probe`).

---

### 3. Partially Implemented & Planned Features

| Feature | Status | Implemented vs. Missing |
| :--- | :--- | :--- |
| **Standalone APK Build** | Scaffolded | `capacitor.config.ts`, `RokidUsbPlugin.java`, `MainActivity.java`, `rokid_usb_device_filter.xml`, and `scripts/setup-android-usb.mjs` are ready. No `.apk` binary was compiled here because JDK, Gradle, and Android SDK are unavailable in this container. |
| **6DoF Fusion & Stereo SBS Display** | Partial | 6-axis Madgwick provides 3DoF without magnetometer heading lock or static gyro-bias calibration (long-term yaw drift is possible). Missing native `3840×1080` SBS stereo rendering outside WebXR and ARCore 6DoF sensor fusion. |
| **Battery Power Profiles (`eco` / `balanced` / `performance`)** | Partial | Actively scales `pixelRatio`, shadows, and `depthMesh.colliderUpdateFps`. `targetFps` (60/90/120 Hz) is displayed in telemetry only and does not throttle the render loop. |
| **Native File Export in Android APK** | Partial | Uses web `<a download>` for exported `.glb`/`.obj`/`.stl` blobs, which may not save to Android storage cleanly inside WebView. |
| **Rokid MCU Display / Audio / UVC Controls** | Planned | Missing proprietary USB control commands for Rokid Max OLED brightness, 2D↔3D SBS mode toggling, USB audio routing, and UVC camera capture. |

---

### 4. Remaining Stubs & Production Replacements

1. **Local-Only "Cloud" Sync & Simulated Collaborator (`src/sync/CloudSyncManager.ts`)**
   - **Stub:** `BroadcastChannel('rokid-spatial-studio-v1')` (lines 43–46, 85–130) only syncs tabs within the *same* local browser. `toggleSimulatedCollaborator()` (`'sim-peer-01'`, *"Elena · Rokid Max #2"*, lines 62–79, 266–290) animates a synthetic peer using `Math.sin`/`Math.cos`.
   - **Production Replacement:** Replace with WebSockets (`wss://`), WebRTC DataChannels, LiveKit, or Firestore/Colyseus/Yjs room synchronization anchored via Cloud Anchors / QR markers, and remove `'sim-peer-01'`.

2. **Simulated "Cloud Hybrid Rendering Pipeline" (`src/sdk/RokidPluginAPI.ts`, lines 237–249, 278, 322–323)**
   - **Stub:** `setRenderPipeline('cloud')` only boosts `scene.environmentIntensity` (`0.95 → 1.45`) and `toneMappingExposure` (`1.02 → 1.18`) with a synthetic `0.68x` power multiplier—no remote GPU server is connected.
   - **Production Replacement:** Integrate WebRTC Pixel Streaming, NVIDIA CloudXR, or a remote WebGPU render server streaming stereo H.265/AV1 frames from upstream 400 Hz head poses.

3. **Synthetic Battery, Thermal & Latency Telemetry (`src/sdk/RokidPluginAPI.ts`, lines 14–45, 69–70, 277–286, 312–324)**
   - **Stub:** `batteryPercent` starts hardcoded at `94.0%` and decrements synthetically; `thermalC` damps toward `31.5 + effectiveWatts * 1.45` unless USB HID `0x80` provides real IMU temperature; `powerWatts` and `latencyMs` (`2.3 / 4.2 / 11.4 ms`) are static lookup values.
   - **Production Replacement:** Query `navigator.getBattery()` on web and Android `BatteryManager` (`BATTERY_PROPERTY_CAPACITY`, `BATTERY_PROPERTY_CURRENT_NOW`, `EXTRA_TEMPERATURE`) via a Capacitor plugin, plus real frame/network RTT timing.

4. **Procedural 360° Walkthrough Panoramas (`src/environment/Panorama360Manager.ts`, lines 96–168)**
   - **Stub:** `createEquirectangularTexture()` draws `1024×512` 2D canvas gradients and gridlines instead of real spherical photos or 360° camera feeds.
   - **Production Replacement:** Bind 4 live `THREE.VideoTexture` streams from WebRTC/RTSP 360° cameras (Insta360 / Ricoh Theta) or load 8K HDR equirectangular panoramas.

5. **Static Box Prototype Collider & Spawn Counter (`src/physics/PhysicsSandbox.ts`, lines 60–68, 126)**
   - **Stub:** Uses a fixed `RAPIER.ColliderDesc.cuboid(0.22, 0.18, 0.22)` regardless of prototype/imported mesh shape, and increments `lastCollisionCount` on spawn rather than collision.
   - **Production Replacement:** Build `RAPIER.ColliderDesc.convexHull(...)` or `trimesh(...)` from active mesh geometry and drain `RAPIER.EventQueue` collision events.

6. **Procedural Canvas PBR Maps (`src/materials/MaterialLibrary.ts`, lines 71–172)**
   - **Stub:** Uses `RoomEnvironment` and small generated 2D canvas textures (`128×128` / `256×256`).
   - **Production Replacement:** Load compressed KTX2 PBR texture sets via `KTX2Loader` and studio HDRIs via `RGBELoader` / `EXRLoader`.

7. **Camera Preview Fallback (`src/ui/SpatialControlDeck.ts`, lines 39–40, 667–668)**
   - **Stub:** Displays `EMPTY_IMAGE` (`1×1` GIF) and `"Stream: Passthrough Simulated"` when no browser camera is detected.
   - **Production Replacement:** Feed real USB-C UVC / `getUserMedia` or Android `CameraX` frames.

8. **Browser Blob Download in APK (`src/io/UniversalModelIO.ts`, lines 173–182)**
   - **Stub:** Uses `<a download>` for `.glb`/`.obj`/`.stl` exports.
   - **Production Replacement:** Use `@capacitor/filesystem` + `@capacitor/share` when `Capacitor.isNativePlatform()` is true.
