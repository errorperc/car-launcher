# Магнитола BOS-MINI, этап 1: чистка, главный экран, звук. План работ

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Обратимо вычистить систему магнитолы по ADB и поставить своё лёгкое приложение: главный экран и экран «Звук» по макету v7.

**Architecture:** Две части. Первая — Python-скрипты в `tools/adb/`, они работают с магнитолой через `adb` (разведка, снимок, чистка, откат). Вторая — одно Android-приложение `app/` на Kotlin и обычных View, без AppCompat и Compose. Логику, которую можно проверить без устройства (эквалайзер, карточки, парсеры), держим в чистых Kotlin-классах под JUnit. Android-обвязка поверх них проверяется на магнитоле по чек-листу. Аудиоэффект живёт внутри `NotificationListenerService`: система сама держит эту службу, поэтому процесс со звуком не умирает, и отдельная постоянная служба не нужна (ТЗ, раздел 10).

**Tech Stack:** Python 3.12 + pytest (скрипты), JDK 17, Android SDK 34, AGP 8.5.2, Gradle 8.7, Kotlin 1.9.24, JUnit 4.13.2, `org.json` из Android (без сторонних JSON-библиотек), шрифты IBM Plex (OFL) в `res/font`.

**Spec:** `docs/specs/2026-11-08-car-launcher-design.md` (ТЗ). Макет: `docs/mockups/screens-v7.html`.

**Вне этого плана:** голосовые команды (ТЗ, раздел 8). На них будет отдельный план после поездок с этапом 1.

## Global Constraints

- Устройство: Android 12, 2 ГБ ОЗУ, экран 1280×720 landscape. `minSdk 31`, `targetSdk 31`, `compileSdk 34`.
- Только `pm disable-user --user 0` или `pm uninstall -k --user 0`. Системный раздел и root не трогаем.
- Никогда не отключать: приложение подключения телефона и его зависимости (Bluetooth, USB, сеть, звук), вентилятор BOS-MINI, «Настройки», SystemUI, системные службы звука, MCU/CAN, штатное радио, установщик пакетов, одну клавиатуру.
- Масштаб анимаций 0,5 (`window_animation_scale`, `transition_animation_scale`, `animator_duration_scale`).
- Приложение: не больше ~60 МБ памяти в работе, холодный старт главного экрана меньше 1 с, своих постоянных служб нет, опросы не чаще раза в секунду, пока открыт CarPlay, ничего не перерисовываем.
- Цвета: `bg #0b0d14`, `deep #07080d`, `card #11141c`, `raised #1a1e28`, линии `#eef0f5` с альфой 8 %, текст `#eef0f5 / #a3a9b8 / #7a8296`, акцент `#38bdf8`, текст на акценте `#04111f`, «подключено» `#4cc38a`.
- Шрифты IBM Plex Sans (400/500, часы 300) и IBM Plex Mono для цифр. Сетка 4 px, касание ≥ 64 px (пресеты, вкладки, фаза 56 px), текст ≥ 16 px.
- Нельзя: эмодзи, подписи заглавными вразрядку, градиенты, свечения, размытие фона. Анимации 120–180 мс, `PathInterpolator(.22f, 1f, .36f, 1f)`, только цвет, прозрачность и сдвиг.
- Штатный лаунчер не отключаем до проверки нашего в поездках.
- Функцию, которую не удалось подключить к железу, на экране не показываем.

## Review Focus

1. Медиа-сессии нет или её отдаёт приложение подключения без названия: карточка показывает «Ничего не играет» или «Играет в CarPlay», кнопки работают, падений нет (тест в Task 6).
2. Перезагрузка или убитый процесс: эквалайзер и сабвуфер восстанавливаются из сохранённого снимка без открытия экрана «Звук» (тест кодека в Task 9, проверка на устройстве в Task 10).
3. Устройство не даёт создать эффект на сессии 0 или не поддерживает DynamicsProcessing: переходим на Equalizer, если не выходит и он — эквалайзер скрыт с подписью, падений нет (тест фабрики в Task 10).
4. Нет сети или GPS: погода показывает последнее значение или прочерк и Омск, ничего не падает (тест в Task 6).
5. Файл температуры не читается или там мусор: кнопка «Охлаждение» без цифры (тест в Task 6).

---

### Task 0: Инструменты и каркас проекта

**Files:**
- Create: `.gitignore`, `settings.gradle.kts`, `build.gradle.kts`, `gradle.properties`, `gradle/wrapper/*`, `gradlew`, `gradlew.bat`
- Create: `app/build.gradle.kts`, `app/src/main/AndroidManifest.xml`, `app/src/main/java/dev/carhome/HomeActivity.kt` (пустая)
- Create: `tools/adb/__init__.py`, `tests/tools/__init__.py`, `requirements-dev.txt` (`pytest==8.3.3`)

- [ ] **Step 1: Спросить владельца и установить инструменты.** Работа идёт на Mac владельца (ноутбук, который едет в машину). Через Homebrew: `brew install openjdk@17 android-platform-tools` и `brew install --cask android-commandlinetools`, затем `sdkmanager "platforms;android-34" "build-tools;34.0.0"`. Прописать `JAVA_HOME` (`$(brew --prefix openjdk@17)`) и `ANDROID_HOME` (`$(brew --prefix)/share/android-commandlinetools`). Если работа всё-таки на Windows: winget `Microsoft.OpenJDK.17`, `Google.PlatformTools`, SDK в `%LOCALAPPDATA%\Android\Sdk`.
- [ ] **Step 2: Проверить.** `java -version` → 17, `adb version` → печатает версию, `sdkmanager --list_installed` → есть `platforms;android-34`.
- [ ] **Step 3: `git init`, каркас Gradle.** applicationId `dev.carhome`, имя приложения «Главный», `minSdk 31`, `targetSdk 31`. Зависимостей приложения нет, кроме Kotlin stdlib; тесты: `junit:junit:4.13.2`.
- [ ] **Step 4: Собрать.** `./gradlew assembleDebug test` → BUILD SUCCESSFUL. `python -m pytest tests` → `no tests ran`, код выхода 5 считается нормой.
- [ ] **Step 5: Commit** `chore: project skeleton`.

### Task 1: ADB-обёртка и разведка

**Files:**
- Create: `tools/adb/adbtool.py`, `tools/adb/recon.py`
- Test: `tests/tools/test_recon.py`

**Interfaces:**
- Produces: `class Adb(runner: Callable[[list[str]], str] = run_adb, serial: str | None = None)` с методом `shell(cmd: str) -> str`; `recon.collect(adb: Adb, outdir: Path) -> list[Path]`; CLI `python -m tools.adb.recon --out recon/<YYYY-MM-DD>`.

- [ ] **Step 1: Тест.**
```python
def test_collect_writes_one_file_per_command(tmp_path):
    calls = []
    adb = Adb(runner=lambda args: calls.append(args) or "out")
    files = collect(adb, tmp_path)
    names = {f.name for f in files}
    assert {"getprop.txt", "packages.txt", "meminfo.txt", "top.txt",
            "audio_flinger.txt", "audio.txt", "audio_policy.txt",
            "audio_effects.xml", "thermal.txt", "input_devices.txt"} <= names
    assert all(f.read_text() == "out" for f in files)
    assert ["shell", "pm list packages -f"] in calls
```
- [ ] **Step 2: Запустить** `python -m pytest tests/tools/test_recon.py` → FAIL (нет модуля).
- [ ] **Step 3: Реализовать.** Команды: `getprop`, `pm list packages -f`, `dumpsys meminfo`, `top -b -n 1`, `dumpsys media.audio_flinger`, `dumpsys audio`, `dumpsys media.audio_policy`, `cat /vendor/etc/audio_effects.xml` (если нет, то `/system/etc/audio_effects.xml`), `for z in /sys/class/thermal/thermal_zone*; do echo $z $(cat $z/type) $(cat $z/temp); done`, `getevent -pl`. Если команда упала, текст ошибки пишем в файл и идём дальше.
- [ ] **Step 4: Запустить** → PASS.
- [ ] **Step 5: Commit** `feat(tools): adb wrapper and recon`.

### Task 2: Снимок, чистка, откат

**Files:**
- Create: `tools/adb/debloat.py`, `tools/adb/debloat-list.txt`, `tools/adb/protected.txt`
- Test: `tests/tools/test_debloat.py`

**Interfaces:**
- Consumes: `Adb` из Task 1.
- Produces: `snapshot(adb, path: Path) -> None` (JSON: `enabled_packages`, `animations`); `load_groups(path: Path) -> dict[str, list[str]]` (секции `[google]`, `[vendor]`, …); `disable(adb, pkgs: list[str], protected: set[str], state: Path) -> list[str]`; `restore(adb, state: Path, baseline: Path) -> None`; `set_animations(adb, scale: float) -> None`. CLI: `snapshot`, `disable --group NAME`, `restore`, `animations 0.5`.

- [ ] **Step 1: Тесты.**
```python
def test_disable_refuses_protected(tmp_path):
    adb = FakeAdb()
    with pytest.raises(ProtectedPackageError):
        disable(adb, ["com.android.vending", "com.android.settings"],
                protected={"com.android.settings"}, state=tmp_path / "s.json")
    assert adb.calls == []                      # ничего не отключено

def test_disable_records_state_and_restore_reenables(tmp_path):
    adb = FakeAdb()
    disable(adb, ["com.google.android.youtube"], set(), tmp_path / "s.json")
    assert "pm disable-user --user 0 com.google.android.youtube" in adb.calls
    (tmp_path / "b.json").write_text(json.dumps(
        {"enabled_packages": [], "animations": {"window_animation_scale": "1.0",
         "transition_animation_scale": "1.0", "animator_duration_scale": "1.0"}}))
    restore(adb, tmp_path / "s.json", tmp_path / "b.json")
    assert "pm enable com.google.android.youtube" in adb.calls
    assert "settings put global window_animation_scale 1.0" in adb.calls

def test_load_groups_skips_comments(tmp_path):
    p = tmp_path / "l.txt"; p.write_text("# x\n[google]\ncom.a\n\n[vendor]\ncom.b # demo\n")
    assert load_groups(p) == {"google": ["com.a"], "vendor": ["com.b"]}
```
- [ ] **Step 2: Запустить** → FAIL.
- [ ] **Step 3: Реализовать.** `restore` включает и пакеты, удалённые через `uninstall -k`: `cmd package install-existing <pkg>`. В `protected.txt` сразу пишем `com.android.settings`, `com.android.systemui`, `com.android.bluetooth`, `com.android.packageinstaller`, `com.google.android.packageinstaller`, `com.android.providers.settings`. Остальное добавляем после разведки в Task 3.
- [ ] **Step 4: Запустить** → PASS.
- [ ] **Step 5: Commit** `feat(tools): snapshot, debloat, restore`.

### Task 3: Разведка на магнитоле (работа на устройстве)

**Files:**
- Create: `recon/2026-11-XX/*` (вывод), `docs/recon.md`, `app/src/main/java/dev/carhome/DeviceConfig.kt`
- Modify: `tools/adb/protected.txt`, `tools/adb/debloat-list.txt`

**Interfaces:**
- Produces: `object DeviceConfig { const val LINK_PACKAGE: String; const val FAN_PACKAGE: String; const val RADIO_PACKAGE: String; const val FILES_PACKAGE: String; const val MUSIC_FALLBACK_PACKAGE: String; const val VENDOR_SOUND_PACKAGE: String? }`.

- [ ] **Step 1:** Владелец включает режим разработчика и отладку по USB, подключает кабель, подтверждает ключ на экране. `adb devices` → одно устройство `device`.
- [ ] **Step 2:** `python -m tools.adb.recon --out recon/<дата>` и `python -m tools.adb.debloat snapshot recon/<дата>/baseline.json`.
- [ ] **Step 3:** В `docs/recon.md` записать:
  - пакеты: приложение подключения, вентилятор, радио, файлы, плеер, штатное приложение звука;
  - как определить, что телефон подключён (процесс, уведомление, USB-устройство, broadcast), и iPhone это или Android;
  - отдаёт ли приложение подключения медиа-сессию с названием трека;
  - какие эффекты есть в `audio_effects.xml` (DynamicsProcessing, Equalizer);
  - как штатное приложение звука задаёт сабвуфер, баланс и тонкомпенсацию (`dumpsys activity services <pkg>`, `settings list system|global` до и после смены ручки, `logcat` при смене);
  - зона температуры процессора.
- [ ] **Step 4:** Перенести найденные пакеты в `DeviceConfig.kt` и `protected.txt` (подключение, вентилятор, радио, всё, от чего зависит подключение). Составить `debloat-list.txt` по группам: `google`, `google-apps`, `vendor-junk`, `keyboards`.
- [ ] **Step 5: Commit** `docs: device recon`.

### Task 4: Чистка магнитолы (работа на устройстве)

- [ ] **Step 1:** `python -m tools.adb.debloat disable --group <первая группа>`.
- [ ] **Step 2:** Перезагрузить, подключить iPhone: CarPlay открывается, звук идёт, вентилятор открывается. Если нет — `restore` и разобрать группу по одному пакету.
- [ ] **Step 3:** Повторить шаги 1–2 для каждой группы.
- [ ] **Step 4:** `python -m tools.adb.debloat animations 0.5`. Снять `dumpsys meminfo` (свободная память до и после) в `docs/recon.md`.
- [ ] **Step 5: Commit** `chore: debloat applied, state saved`.

### Task 5: Тема, шрифты и неподвижный главный экран

**Files:**
- Create: `app/src/main/res/values/colors.xml`, `dimens.xml`, `styles.xml`, `res/font/plex_sans_*.ttf`, `plex_mono_*.ttf`, `res/layout/activity_home.xml`, `res/drawable/*` (иконки нижней панели, иконка «телефон → экран», фоны карточек)
- Create: `app/src/main/java/dev/carhome/ui/TickScaleView.kt`
- Modify: `HomeActivity.kt`, `AndroidManifest.xml`

**Interfaces:**
- Produces: `TickScaleView` с `var ticks: Int`, `var lit: Int` (режим громкости) и `var progress: Float` 0..1 (режим полосы трека: 41 деление, каждое 10-е длиннее, дорожка 8 px, ручка 24 px). Атрибут `app:mode="volume|progress"`.
- Produces: id в `activity_home.xml`: `cover`, `source`, `title`, `artist`, `album`, `volumeScale`, `volumeValue`, `progress`, `timeNow`, `timeTotal`, `btnPrev`, `btnPlay`, `btnNext`, `linkStatus`, `linkTitle`, `linkSub`, `btnOpenLink`, `clock`, `date`, `temp`, `weather`, `navMusic`, `navRadio`, `navSound`, `navCooling`, `coolingTemp`, `navFiles`, `navApps`.

- [ ] **Step 1:** Разметка по макету v7: сетка `1fr 416px`, отступы 24, карточки 16 радиус, обложка 232, кнопки 72/88/72, нижняя панель 6 × 72 px. Тексты-заглушки как в макете.
- [ ] **Step 2:** Манифест: `HomeActivity` с `MAIN` + `HOME` + `DEFAULT`, `screenOrientation="landscape"`, `launchMode="singleTask"`, тема без заголовка и анимаций окна, фон `bg`.
- [ ] **Step 3:** `./gradlew installDebug`, открыть на магнитоле `adb shell am start -n dev.carhome/.HomeActivity`. Скриншот `adb exec-out screencap -p > docs/shots/home-static.png` совпадает с макетом по раскладке, текст не обрезан.
- [ ] **Step 4: Commit** `feat(app): static home screen`.

### Task 6: Чистая логика главного экрана

**Files:**
- Create: `app/src/main/java/dev/carhome/home/CardState.kt`, `LinkTexts.kt`, `CpuTemp.kt`, `Weather.kt`, `RuDate.kt`
- Test: `app/src/test/java/dev/carhome/home/*Test.kt`

**Interfaces:**
- Produces:
  - `data class MediaMeta(title: String?, artist: String?, album: String?, year: Int?, durationMs: Long, positionMs: Long, playing: Boolean)`
  - `sealed class CardState { object Empty; data class Playing(meta: MediaMeta, source: String); data class ProjectionOnly(text: String) }`
  - `fun cardStateFor(meta: MediaMeta?, controllerPackage: String?, linkPackage: String, phoneOs: PhoneOs): CardState`
  - `enum class PhoneOs { IOS, ANDROID, UNKNOWN }`, `data class PhoneLink(connected: Boolean, os: PhoneOs)`
  - `data class LinkTexts(status: String, title: String, sub: String, button: String, enabled: Boolean)`, `fun linkTexts(link: PhoneLink): LinkTexts`
  - `object CpuTemp { fun parse(raw: String): Int?; fun read(root: File = File("/sys/class/thermal"), zone: String? = null): Int? }`
  - `data class Weather(tempC: Int, code: Int, fetchedAtMs: Long)`, `object OpenMeteo { fun url(lat: Double, lon: Double): String; fun parse(json: String, nowMs: Long): Weather? }`, `object WeatherCodes { fun label(code: Int): String }`, `fun weatherLine(w: Weather?, city: String?): Pair<String, String>` (температура, подпись)
  - `object RuDate { fun time(ms: Long, zone: ZoneId): String; fun date(ms: Long, zone: ZoneId): String }`

- [ ] **Step 1: Тесты.**
```kotlin
@Test fun noSessionIsEmpty() = assertEquals(CardState.Empty, cardStateFor(null, null, "link.pkg", PhoneOs.IOS))
@Test fun linkWithoutTitleIsProjection() = assertEquals(
    CardState.ProjectionOnly("Играет в CarPlay"),
    cardStateFor(MediaMeta(null, null, null, null, 0, 0, true), "link.pkg", "link.pkg", PhoneOs.IOS))
@Test fun androidProjectionText() = assertEquals(
    CardState.ProjectionOnly("Играет в Android Auto"),
    cardStateFor(MediaMeta("", null, null, null, 0, 0, true), "link.pkg", "link.pkg", PhoneOs.ANDROID))
@Test fun bluetoothTrackPlays() { val s = cardStateFor(MediaMeta("Трек", "Исп", null, null, 1, 0, true), "com.android.bluetooth", "link.pkg", PhoneOs.IOS)
    assertTrue(s is CardState.Playing) }

@Test fun iphoneTexts() = assertEquals(LinkTexts("iPhone подключён", "CarPlay", "Карта и музыка с iPhone", "Открыть CarPlay", true), linkTexts(PhoneLink(true, PhoneOs.IOS)))
@Test fun androidTexts() = assertEquals(LinkTexts("Телефон подключён", "Android Auto", "Карта и музыка с телефона", "Открыть Android Auto", true), linkTexts(PhoneLink(true, PhoneOs.ANDROID)))
@Test fun noPhoneTexts() = assertEquals(LinkTexts("Телефон не подключён", "CarPlay и Android Auto", "Подключите телефон кабелем", "Открыть", false), linkTexts(PhoneLink(false, PhoneOs.UNKNOWN)))

@Test fun cpuMilli() = assertEquals(40, CpuTemp.parse("39800\n"))
@Test fun cpuDegrees() = assertEquals(41, CpuTemp.parse("41"))
@Test fun cpuGarbage() { assertNull(CpuTemp.parse("abc")); assertNull(CpuTemp.parse("")); assertNull(CpuTemp.parse("250000")) }
@Test fun cpuMissingDir() = assertNull(CpuTemp.read(File("/nonexistent")))

@Test fun meteoParse() = assertEquals(Weather(-3, 3, 5L), OpenMeteo.parse("""{"current":{"temperature_2m":-2.6,"weather_code":3}}""", 5L))
@Test fun meteoBad() = assertNull(OpenMeteo.parse("<html>", 0))
@Test fun codes() { assertEquals("ясно", WeatherCodes.label(0)); assertEquals("пасмурно", WeatherCodes.label(3)); assertEquals("снег", WeatherCodes.label(73)); assertEquals("гроза", WeatherCodes.label(95)) }
@Test fun noWeatherNoCity() = assertEquals("—" to "Омск", weatherLine(null, null))
@Test fun weatherLine() = assertEquals("−3°" to "Омск, пасмурно", weatherLine(Weather(-3, 3, 0), "Омск"))

@Test fun ruDate() { val z = ZoneId.of("Asia/Omsk"); val ms = 1794148920000L
    assertEquals("20:42", RuDate.time(ms, z)); assertEquals("Воскресенье, 8 ноября", RuDate.date(ms, z)) }
```
- [ ] **Step 2: Запустить** `./gradlew test` → FAIL.
- [ ] **Step 3: Реализовать.** `CpuTemp.parse`: число ≥ 1000 — миллиградусы, делим и округляем; итог вне 0..150 → null. `read`: зона по имени из `DeviceConfig`, иначе первая с `cpu` в `type`, иначе максимум. Коды WMO: 0 ясно; 1, 2 облачно; 3 пасмурно; 45, 48 туман; 51–57 морось; 61–67 дождь; 71–77 снег; 80–82 ливень; 85, 86 снегопад; 95–99 гроза. Минус в температуре — знак `−` (U+2212). `url` запрашивает `current=temperature_2m,weather_code`.
- [ ] **Step 4: Запустить** → PASS.
- [ ] **Step 5: Commit** `feat(app): home screen logic`.

### Task 7: Музыка, громкость, подключение телефона, кнопки

**Files:**
- Create: `app/src/main/java/dev/carhome/media/MediaListenerService.kt`, `media/NowPlayingRepository.kt`, `link/PhoneLinkDetector.kt`, `home/Launch.kt`, `apps/AppsActivity.kt`, `res/layout/activity_apps.xml`
- Modify: `HomeActivity.kt`, `AndroidManifest.xml`

**Interfaces:**
- Consumes: `cardStateFor`, `linkTexts`, `CpuTemp`, `RuDate`, `DeviceConfig`.
- Produces: `class NowPlayingRepository(context: Context) { fun observe(cb: (CardState, MediaController?) -> Unit); fun stop() }`; `class PhoneLinkDetector(context: Context) { fun observe(cb: (PhoneLink) -> Unit); fun stop() }`; `object Launch { fun intentFor(ctx: Context, pkg: String): Intent?; fun lastMusicPackage(ctx: Context): String? }`. `MediaListenerService : NotificationListenerService` (позже держит звук, Task 10).

- [ ] **Step 1:** Подписки на медиа-сессии через `MediaSessionManager.getActiveSessions(ComponentName(MediaListenerService))`. Последний пакет, игравший музыку, пишется в prefs для кнопки «Музыка». Громкость — `AudioManager.STREAM_MUSIC` и broadcast `android.media.VOLUME_CHANGED_ACTION`, шкала 30 делений пропорционально максимуму. Позиция трека обновляется раз в секунду только между `onStart` и `onStop`.
- [ ] **Step 2:** `PhoneLinkDetector` реализует способ из `docs/recon.md`. При переходе из «не подключён» в «подключён», если приложение подключения само не вышло на передний план за 3 с, запускаем его (ТЗ 6.0).
- [ ] **Step 3:** Кнопки: Открыть → `LINK_PACKAGE`; Радио, Файлы, Охлаждение → пакеты из `DeviceConfig`; Музыка → `lastMusicPackage` или `MUSIC_FALLBACK_PACKAGE`; Звук → `SoundActivity` (Task 11, пока заглушка); Приложения → `AppsActivity` (`GridView` по `queryIntentActivities(MAIN/LAUNCHER)`, 6 колонок, без анимаций). Если `Launch.intentFor` вернул null, кнопка неактивна.
- [ ] **Step 4:** Выдать доступ к уведомлениям: `adb shell cmd notification allow_listener dev.carhome/dev.carhome.media.MediaListenerService`.
- [ ] **Step 5: Проверка на магнитоле:** без телефона карточка «Ничего не играет» и «Телефон не подключён»; Bluetooth-трек с iPhone виден с названием и обложкой, кнопки работают; при CarPlay карточка показывает трек или «Играет в CarPlay», кнопки переключают трек; физические кнопки громкости двигают шкалу; каждая кнопка нижней панели открывает своё приложение; «Охлаждение» показывает температуру.
- [ ] **Step 6: Commit** `feat(app): live home screen`.

### Task 8: Погода

**Files:**
- Create: `app/src/main/java/dev/carhome/home/WeatherUpdater.kt`
- Modify: `HomeActivity.kt`, `AndroidManifest.xml` (`INTERNET`, `ACCESS_COARSE_LOCATION`)

**Interfaces:**
- Consumes: `OpenMeteo`, `weatherLine`.
- Produces: `class WeatherUpdater(context: Context) { fun current(): Weather?; fun refreshIfStale(nowMs: Long, cb: (Weather?, String?) -> Unit) }`.

- [ ] **Step 1:** Координаты один раз через `LocationManager.getLastKnownLocation` (GPS, потом network), сохранить в prefs. Без координат — Омск `54.99, 73.37`, город «Омск». Город по координатам через `Geocoder`, если он есть на устройстве, иначе без города.
- [ ] **Step 2:** Запрос `HttpURLConnection`, таймаут 5 с, в фоновом потоке, не чаще раза в 30 минут и только пока экран на переднем плане. Результат в prefs. Ошибка — оставляем прошлое значение.
- [ ] **Step 3: Проверка:** с раздачей интернета видна температура; выключить сеть и перезапустить — видно прошлое значение; очистить данные приложения без сети — «—» и «Омск».
- [ ] **Step 4: Commit** `feat(app): weather`.

### Task 9: Модель эквалайзера и сабвуфера

**Files:**
- Create: `app/src/main/java/dev/carhome/sound/EqPresets.kt`, `EqModel.kt`, `Stepper.kt`, `EqSnapshot.kt`
- Test: `app/src/test/java/dev/carhome/sound/*Test.kt`

**Interfaces:**
- Produces:
  - `object EqPresets { val BANDS_HZ = intArrayOf(31, 62, 125, 250, 500, 1000, 2000, 4000, 8000, 16000); const val MIN_DB = -12; const val MAX_DB = 12; val STOCK: Map<String, IntArray>; val SLOTS = listOf("Мой 1", "Мой 2", "Мой 3") }`. Пресеты (черновые, из макета): Ровный `0×10`; Рок `5,4,2,-1,-2,0,2,4,5,5`; Поп `-1,1,3,4,3,1,0,1,2,2`; Голос `-3,-2,0,2,4,5,4,2,0,-1`. Слоты по умолчанию нулевые.
  - `class EqModel(snapshot: EqSnapshot = EqSnapshot.DEFAULT)` со свойствами `selected: String`, `bands: IntArray`, `dirty: Boolean`, `enabled: Boolean`, `sub: Stepper`, `cut: Stepper`, `phase: Phase`, `loudness: Boolean`, `listening: Listening`, методами `select(name)`, `setBand(i: Int, db: Int)`, `saveTo(slot: String)`, `setEnabled(on: Boolean)`, `hint(): String`, `snapshot(): EqSnapshot` и колбэком `onChange: (() -> Unit)?`.
  - `class Stepper(val min: Int, val max: Int, val step: Int, value: Int)` с `value`, `inc()`, `dec()`, `canInc`, `canDec`.
  - `enum class Phase { DEG0, DEG180 }`, `enum class Listening { DRIVER, CENTER }`.
  - `data class EqSnapshot(...)` + `object EqSnapshotCodec { fun encode(s: EqSnapshot): String; fun decode(s: String?): EqSnapshot }` (битая или пустая строка → `DEFAULT`).

- [ ] **Step 1: Тесты.**
```kotlin
@Test fun stockIsReadOnly() { val m = EqModel(); m.select("Рок"); m.setBand(0, 12)
    assertTrue(m.dirty); assertArrayEquals(intArrayOf(5,4,2,-1,-2,0,2,4,5,5), EqPresets.STOCK["Рок"])
    assertEquals("«Рок» изменён. Удерживайте «Мой 1-3», чтобы сохранить", m.hint()) }
@Test fun saveDirtyToSlot() { val m = EqModel(); m.select("Рок"); m.setBand(0, 12); m.saveTo("Мой 2")
    assertFalse(m.dirty); assertEquals("Мой 2", m.selected); assertEquals(12, m.snapshot().slots["Мой 2"]!![0])
    assertEquals("Сохранено в «Мой 2»", m.hint()) }
@Test fun slotEditsSaveImmediately() { val m = EqModel(); m.select("Мой 1"); m.setBand(3, -4)
    assertFalse(m.dirty); assertEquals(-4, m.snapshot().slots["Мой 1"]!![3])
    assertEquals("Правки сразу сохраняются в «Мой 1»", m.hint()) }
@Test fun stockHint() { val m = EqModel(); m.select("Поп"); assertEquals("Готовый пресет. Подвиньте полосу, чтобы сделать свой", m.hint()) }
@Test fun bypassBlocksEdits() { val m = EqModel(); m.setEnabled(false); val before = m.bands.copyOf(); m.setBand(0, 7)
    assertArrayEquals(before, m.bands); assertEquals("Эквалайзер выключен, звук идёт без обработки", m.hint()) }
@Test fun bandClamped() { val m = EqModel(); m.select("Мой 1"); m.setBand(0, 40); assertEquals(12, m.bands[0]); m.setBand(0, -40); assertEquals(-12, m.bands[0]) }
@Test fun subDefaultsAndLimits() { val m = EqModel(); assertEquals(4, m.sub.value); assertEquals(80, m.cut.value)
    repeat(30) { m.cut.dec() }; assertEquals(10, m.cut.value); assertFalse(m.cut.canDec)
    repeat(30) { m.cut.inc() }; assertEquals(200, m.cut.value); assertFalse(m.cut.canInc)
    repeat(30) { m.sub.inc() }; assertEquals(10, m.sub.value) }
@Test fun codecRoundTrip() { val m = EqModel(); m.select("Мой 3"); m.setBand(9, 6); m.cut.dec(); m.setEnabled(false)
    val back = EqModel(EqSnapshotCodec.decode(EqSnapshotCodec.encode(m.snapshot())))
    assertEquals("Мой 3", back.selected); assertEquals(6, back.bands[9]); assertEquals(70, back.cut.value); assertFalse(back.enabled) }
@Test fun codecGarbage() = assertEquals(EqSnapshot.DEFAULT, EqSnapshotCodec.decode("{oops"))
```
- [ ] **Step 2: Запустить** → FAIL.
- [ ] **Step 3: Реализовать.** По умолчанию выбран «Ровный», эквалайзер включён, сабвуфер `Stepper(-10, 10, 1, 4)`, срез `Stepper(10, 200, 10, 80)`, фаза 0°, тонкомпенсация вкл, баланс «водитель». Подсказка «Сохранено в …» держится, пока не будет следующего изменения. Кодек — `org.json`; в юнит-тестах подключить `testImplementation("org.json:json:20240303")`.
- [ ] **Step 4: Запустить** → PASS.
- [ ] **Step 5: Commit** `feat(app): equalizer model`.

### Task 10: Звуковой движок и проверка, слышен ли эффект

**Files:**
- Create: `app/src/main/java/dev/carhome/sound/AudioEngine.kt`, `DynamicsEngine.kt`, `LegacyEqEngine.kt`, `BandMap.kt`, `VendorAudio.kt`, `SoundHub.kt`, `boot/BootReceiver.kt`
- Modify: `MediaListenerService.kt`, `AndroidManifest.xml` (`MODIFY_AUDIO_SETTINGS`, `RECEIVE_BOOT_COMPLETED`)
- Test: `app/src/test/java/dev/carhome/sound/BandMapTest.kt`, `AudioEngineFactoryTest.kt`

**Interfaces:**
- Consumes: `EqModel`, `EqSnapshotCodec`.
- Produces:
  - `interface AudioEngine { fun apply(bandsDb: IntArray, enabled: Boolean): Boolean; fun release() }`
  - `object AudioEngineFactory { fun create(makers: List<() -> AudioEngine?>): AudioEngine? }` (первый, кто создался и вернул `apply == true` на нулевой кривой)
  - `object BandMap { fun toDevice(ourHz: IntArray, ourDb: IntArray, deviceCenterMilliHz: IntArray): ShortArray }` (миллибелы; каждой полосе устройства — значение нашей ближайшей по логарифму частоты)
  - `enum class VendorFeature { SUB_LEVEL, SUB_CUTOFF, SUB_PHASE, LOUDNESS, BALANCE, HPF, DELAY }`; `interface VendorAudio { val features: Set<VendorFeature>; fun setSub(levelDb: Int, cutHz: Int, phase: Phase); fun setLoudness(on: Boolean); fun setListening(l: Listening) }`; `object NoVendorAudio : VendorAudio` (пустой набор)
  - `object SoundHub { val model: EqModel; var engine: AudioEngine?; var vendor: VendorAudio; val eqAvailable: Boolean; fun init(ctx: Context); fun applyAll() }`, состояние сохраняется в prefs при каждом `onChange`.

- [ ] **Step 1: Тесты.**
```kotlin
@Test fun mapFiveBand() = assertArrayEquals(shortArrayOf(400, 200, 0, 200, 600),
    BandMap.toDevice(EqPresets.BANDS_HZ, intArrayOf(5,4,3,2,1,0,1,2,6,6), intArrayOf(60000, 230000, 910000, 3600000, 14000000)))
@Test fun factoryFallsBack() { val bad = object : AudioEngine { override fun apply(b: IntArray, e: Boolean) = false; override fun release() {} }
    val good = object : AudioEngine { override fun apply(b: IntArray, e: Boolean) = true; override fun release() {} }
    assertSame(good, AudioEngineFactory.create(listOf({ throw RuntimeException() }, { bad }, { good }))) }
@Test fun factoryAllFail() = assertNull(AudioEngineFactory.create(listOf({ null }, { throw UnsupportedOperationException() })))
```
- [ ] **Step 2: Запустить** → FAIL.
- [ ] **Step 3: Реализовать.** `DynamicsEngine`: `DynamicsProcessing(0, 0, config)`, 2 канала, только pre-EQ на 10 полос с частотами отсечки посередине между центрами (по логарифму), MBC и limiter выключены. `LegacyEqEngine`: `Equalizer(0, 0)` + `BandMap`. `SoundHub.init` вызывают `MediaListenerService.onListenerConnected` и `BootReceiver`; порядок `[DynamicsEngine, LegacyEqEngine]`. Если оба не создались, `eqAvailable = false`.
- [ ] **Step 4: Запустить** → PASS.
- [ ] **Step 5: Проверка на магнитоле (временная отладочная кнопка «Рок / Ровный» на главном экране, удалить в конце задачи).** По очереди CarPlay, Bluetooth и радио: на «Рок» слышно больше баса и верха, чем на «Ровный». После перезагрузки, не открывая приложение, «Рок» слышен. Итог по каждому источнику и выбранный движок записать в `docs/recon.md` и применить правило ТЗ 7.4: работает для CarPlay — оставить, не работает для радио — флаг `EQ_SKIPS_RADIO = true` в `DeviceConfig`; не работает вовсе — остановить план и вернуться к владельцу с вариантом «эквалайзер аудиочипа».
- [ ] **Step 6: Commit** `feat(app): audio engine`.

### Task 11: Экран «Звук»

**Files:**
- Create: `app/src/main/java/dev/carhome/sound/SoundActivity.kt`, `ui/EqView.kt`, `res/layout/activity_sound.xml`
- Modify: `HomeActivity.kt` (кнопка «Звук»)

**Interfaces:**
- Consumes: `SoundHub`, `EqModel`, `EqPresets`, `VendorFeature`.
- Produces: `class EqView(context, attrs) : View` с `var bands: IntArray`, `var enabled: Boolean`, `var onBandChange: ((index: Int, db: Int) -> Unit)?`. Рисует ось +12/+6/0/−6/−12 (линия 0 ярче, альфа 35 %), 10 дорожек 8 px, ручки 40 px с обводкой 6 px цвета карточки, значения над дорожками и частоты под ними (Mono 16), кривую Безье 3 px акцентом и заливку акцентом 10 % до линии 0. Дорожка начинается на 48 px ниже верха и кончается на 48 px выше низа. Перетаскивание по вертикали, шаг 1 дБ.

- [ ] **Step 1:** Разметка по макету v7: шапка (заголовок, вкладки, переключатель), слева пресеты + подсказка + `EqView`, справа колонка 392 px: сабвуфер (две строки − значение + и сегмент фазы), тонкомпенсация, баланс.
- [ ] **Step 2:** Поведение: тап по пресету — `select`; долгое нажатие 600 мс на «Мой N» при `dirty` — `saveTo`, кнопка на 1,2 с подсвечивается акцентом. Выключенный эквалайзер: пресеты и `EqView` с альфой 0,4 и без касаний, кривая скрыта. Кнопки степперов неактивны (альфа 0,35) на пределе. Подпись среза: «срез, 10-200 Гц»; при срезе ниже 30 Гц вместо неё «ниже 30 Гц саб почти не слышен» (ТЗ 7.4). Баланс: точка сдвигается анимацией 200 мс, текст «Точка звука: водитель» / «Точка звука: центр салона», кнопка «Центр» / «Водитель».
- [ ] **Step 3:** Видимость: блок «Сабвуфер» и его строки — по `SUB_LEVEL`, `SUB_CUTOFF`, `SUB_PHASE`; «Тонкомпенсация» — `LOUDNESS`; «Баланс» — `BALANCE` или `DELAY`; вкладки «Сабвуфер и фильтры» и «Задержка» — по `HPF`/`DELAY`, без них вкладок нет совсем. Если `eqAvailable == false`, вместо графика текст «Эквалайзер недоступен на этой магнитоле». При `EQ_SKIPS_RADIO` под пресетами строка «Не влияет на радио». Если правая колонка пуста, график на всю ширину.
- [ ] **Step 4: Проверка на магнитоле:** все сценарии подсказок из Task 9 видны на экране; перетаскивание плавное, без рывков; после перезапуска выбранный пресет и кривая на месте; скриншот `docs/shots/sound.png` совпадает с макетом.
- [ ] **Step 5: Commit** `feat(app): sound screen`.

### Task 12: Сабвуфер, тонкомпенсация и баланс через железо

**Files:**
- Create: `app/src/main/java/dev/carhome/sound/<Vendor>Audio.kt` (имя по найденному способу)
- Modify: `SoundHub.kt`

**Interfaces:**
- Consumes: `VendorAudio`, `VendorFeature`, способ из `docs/recon.md`.

- [ ] **Step 1:** Реализовать `VendorAudio` тем способом, который делает штатное приложение звука. В `features` только то, что подтверждено на слух. Баланс: при `DELAY` «водитель» задаётся задержками; при одном `BALANCE` «водитель» — сдвиг на 30 % шкалы влево и вперёд; значение подобрать на слух и записать в `DeviceConfig`. Штатный эквалайзер чипа выставить ровно, если работает наш программный (ТЗ 7.4).
- [ ] **Step 2: Проверка:** уровень, срез, фаза и баланс слышно меняются; после перезагрузки сохраняются. Если способа нет — `SoundHub.vendor = NoVendorAudio`, соответствующие блоки скрыты, запись об этом в `docs/recon.md`.
- [ ] **Step 3: Commit** `feat(app): vendor audio`.

### Task 13: Главный экран по умолчанию, замеры, поездки

- [ ] **Step 1:** `adb shell cmd package set-home-activity dev.carhome/.HomeActivity`. Кнопка «Домой» ведёт на наш экран, штатный лаунчер остаётся включённым.
- [ ] **Step 2: Замеры.** `adb shell am force-stop dev.carhome && adb shell am start -W -n dev.carhome/.HomeActivity` → `TotalTime` < 1000. При открытом CarPlay через 5 минут `adb shell dumpsys meminfo dev.carhome` → `TOTAL PSS` ≤ 60 МБ. `adb shell dumpsys gfxinfo dev.carhome` при открытом CarPlay → новых кадров нет. Цифры записать в `docs/recon.md`.
- [ ] **Step 3: Сборка release** с минификацией R8, подписью своим ключом (ключ не коммитить), установка `adb install -r`.
- [ ] **Step 4: Чек-лист поездок (3 поездки, отмечает владелец):** CarPlay открывается сам после подключения; звук и эквалайзер работают; вентилятор открывается; после холодного старта машины всё на месте; ничего не тормозит. Только после этого, по желанию владельца, можно отключить штатный лаунчер через `debloat.py`.
- [ ] **Step 5: Commit** `chore: release and default home`.
