# Debloat Android TV + свой домашний экран

Бриф для ИИ-агента с доступом к терминалу (Claude Code, Cursor, аналоги).
Скопируйте этот файл целиком в чат и скажите: «сделай по этому документу».

Подходит для **любого Android TV / Google TV**: TCL, Xiaomi, Hisense, Sony,
OnePlus, Realme, Nvidia Shield, Chromecast with Google TV. **Не подходит**
Samsung (Tizen), LG (webOS), Fire TV.

Рут не нужен. Всё обратимо. Конкретные имена пакетов у каждого производителя
свои — поэтому ниже не список, а **способ их найти**.

---

## Что должно получиться

- С телевизора убраны реклама, телеметрия производителя, магазинный демо-режим
  и неиспользуемые предустановки
- Автообновление Play Store выключено, анимации урезаны
- Вместо штатного домашнего экрана — свой: док избранного, сетка приложений,
  обои, часы, погода. Настраивается с пульта
- Есть скрипт отката на всё

---

## Правила работы (агенту)

1. **Спрашивай перед удалением всего, что похоже на функцию, а не на мусор.**
   Пользователь знает, чем пользуется, ты — нет.
2. **Проверяй скриншотами, а не предположениями.** `adb exec-out screencap -p > s.png`
   и посмотри на него. UI-дампы врут о том, как это выглядит.
3. **Выводы делай из логов и `dumpsys`.** На глаз в скриншоте легко увидеть баг,
   которого нет.
4. **Не перезагружай телевизор, пока не проверил изменения.** Пока система на
   ногах, adb по сети жив и всё откатывается.
5. **Партиями по 3–6 пакетов**, с проверкой после каждой. Снёс двадцать разом и
   телевизор встал — не узнаешь, который виноват.

---

## 0. Подключение

На телевизоре: **Настройки → Система → Об устройстве → 7 раз нажать «Сборка»
→ назад → Для разработчиков → включить «Отладка по USB»** (или «Отладка по
сети»). Телевизор и компьютер в одной сети.

```bash
adb connect <IP>:5555
adb devices          # должно быть "device", не "offline"
```

Если `adb` не достучится, а телевизор пингуется: на macOS бинарник иногда
блокируется защитой локальной сети. Обходится питоновским пакетом `adb-shell`.

---

## 1. Разведка

```bash
adb shell getprop > getprop.txt
adb shell 'pm list packages -f' > packages.txt
adb shell 'uptime; free -m; df -h /data'
adb shell pm list users
```

Скажи пользователю четыре вещи:

| Проверка | Почему важно |
|---|---|
| `ro.product.cpu.abilist64` | Если **пусто** — CPU 32-битный, и APK только под `arm64-v8a` не поставить |
| `ro.boot.flash.locked` | `1` = бутлоадер залочен. Рута и кастомов не будет, всё через `pm` на уровне user 0 |
| `free -m` + своп | Своп забит, а CPU простаивает → проблема в памяти, чистка поможет |
| `pm list users` | Лишние профили Google TV (`Type: com.android.tv.profile`) ломают часть функций |

---

## 2. Бэкап — до любых изменений

```bash
mkdir -p backup/apks
for ns in system secure global; do adb shell "settings list $ns" | sort > backup/settings_$ns.txt; done
adb shell 'pm list packages -e' | sed 's/^package://' | sort > backup/enabled.txt
adb shell 'pm list packages -d' | sed 's/^package://' | sort > backup/disabled.txt
adb shell 'pm list packages -u' | sed 's/^package://' | sort > backup/all.txt

# APK сторонних приложений: они в /data/app и при удалении исчезнут физически
for p in $(adb shell 'pm list packages -3' | tr -d '\r' | sed 's/^package://'); do
  adb pull "$(adb shell pm path $p | tr -d '\r' | sed 's/^package://' | head -1)" "backup/apks/$p.apk"
done
```

Напиши скрипт отката: `cmd package install-existing` для всех из `all.txt`,
`pm enable` для `enabled.txt`, `pm disable-user` для `disabled.txt`,
`settings put` для дампов настроек.

> **`adb backup` не трать время.** С Android 12 он исключает всё с
> `targetSdk >= 31` — в архив попадут единицы приложений. Плюс Backup Manager
> может быть выключен в прошивке, а включить его пультом нельзя:
> `android.settings.BACKUP_AND_RESET_SETTINGS` не существует, `PRIVACY_SETTINGS`
> ведёт в заглушку.

---

## 3. Как понять, что удалять — на любой модели

Не копируй чужие списки. Классифицируй пакеты этого телевизора.

### 3.1 Собери кандидатов

```bash
adb shell 'pm list packages -s -f' | tr -d '\r' | sed 's/^package://'
```

Смотри на путь: `/system/priv-app`, `/product/app`, `/vendor/app` — там лежит
вендорское. Выпиши всё, что не похоже на AOSP (`com.android.*`) и не Google.

### 3.2 Узнай, что пакет реально делает

**По живым сервисам** — самый надёжный признак телеметрии:

```bash
adb shell 'dumpsys activity services' | grep -oE 'ServiceRecord\{[^}]*\}' | sort -u
```

Ищи в именах: `BIService`, `TrackData`, `Usagest`, `Identifier`, `Analytics`,
`Report`, `Log`. Пакет с такими сервисами — телеметрия, даже если называется
безобидно.

**По обрабатываемым интентам** — так проверяется, не системная ли это прослойка:

```bash
adb shell "dumpsys package <пакет>" | sed -n '/Activity Resolver Table/,/Service Resolver/p'
```

**Критично**: прежде чем удалять, проверь, не единственный ли это обработчик
важного действия:

```bash
adb shell 'cmd package query-activities --brief -a com.android.tv.action.VIEW_INPUTS'
adb shell 'cmd package query-activities --brief -a android.media.tv.action.SETUP_INPUTS'
adb shell 'cmd package query-activities --brief -a android.intent.action.MAIN -c android.intent.category.HOME'
```

Если пакет — единственный ответ, **не удаляй**, что бы ни значило его имя.

**Проверь эмпирически**, если сомневаешься. Переключись на HDMI и посмотри,
какая активность его рисует:

```bash
adb shell 'dumpsys tv_input | grep inputId'    # узнать id входа
adb shell 'am start -a android.intent.action.VIEW -d "content://android.media.tv/passthrough/<id>"'
adb shell 'dumpsys window | grep -m1 mFocusedApp'
```

### 3.3 Имена врут — реальные примеры

На TCL мы поймали три штуки, где название вводит в заблуждение. На другой
модели будут свои, но принцип тот же:

| Пакет | Ожидание по имени | Что на самом деле |
|---|---|---|
| `com.tcl.tv` | эфирные каналы | **рендерер HDMI-входов**, единственный обработчик `SETUP_INPUTS`. Удалить = HDMI не показывает |
| `com.tcl.suspension` | что-то про сон | **переключатель источников**, единственный обработчик `VIEW_INPUTS` |
| `com.tcl.ttvs` | «TV service» | телеметрия: BI, TrackData, Usagest, Identifier |

### 3.4 Типовые цели (ищи аналоги у своего вендора)

- телеметрия и BI вендора
- рекламные полки на домашнем экране
- магазинный демо-режим, гостиничный режим, сервисный гид
- аккаунт производителя
- `android.autoinstalls.config.google.gtvpai` — Play Auto-Install, молча
  доустанавливает промо-приложения
- `com.google.android.syncadapters.calendar` + `com.android.providers.calendar` —
  синхронизация календаря на телевизоре, у нас занимала ~160 МБ
- `com.android.settings.intelligence` — поиск внутри меню настроек
- предустановленные стриминги, которыми пользователь не пользуется

### 3.5 Не трогать никогда

GMS, `gsf`, `webview`, `vending`, `modulemetadata`, `ext.services`,
`ext.shared`, `systemui`, `packageinstaller`, `permissioncontroller`,
`providers.settings|media|downloads`, `com.google.android.tv.remote.service`
(пульт по Wi-Fi/BT), все `*.overlay.*` и `*.resources*`, вендорские HAL,
`tungsten.setupwraith`, и **обе клавиатуры сразу** — останешься без ввода.

### 3.6 Как удалять

```bash
adb shell pm uninstall --user 0 <пакет>          # убрать, APK остаётся в прошивке
adb shell cmd package install-existing <пакет>   # вернуть
```

Всегда с `--user 0`. Партиями, со скриншотом после каждой.

### 3.7 Прочие настройки

```bash
for k in window_animation_scale transition_animation_scale animator_duration_scale; do
  adb shell settings put global $k 0.5
done
```

**Автообновление Play Store** выключается только в его UI: Play Store → аватар →
Настройки → Автообновление → Отключить. Ключ `settings global auto_update_apps`
современный Finsky игнорирует. Агент может пройти туда через `input keyevent` +
`uiautomator dump`, но учти: строки прокручиваются, и координаты для `input tap`
уезжают — DPAD надёжнее.

**Ambient Mode:** если удаляешь штатную заставку, сначала перенацель её, иначе
останется ссылка на удалённый компонент:

```bash
adb shell settings put secure screensaver_components com.android.dreams.basic/.Colors
```

---

## 4. Свой домашний экран

Агент пишет лончер сам. Спецификация ниже проверена на устройстве.

### Что это за приложение

Обычное Android-приложение с `category.HOME`. Чистая Java, **без AndroidX,
без Gradle, без нативного кода** — тогда 32-битность телевизора безразлична,
а APK получается ~25 МБ PSS против 110 у штатного.

### Манифест

```xml
<uses-sdk android:minSdkVersion="26" android:targetSdkVersion="31" />
<uses-permission android:name="android.permission.INTERNET" />          <!-- погода -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-feature android:name="android.software.leanback" android:required="false" />
<uses-feature android:name="android.hardware.touchscreen" android:required="false" />

<!-- Видимость пакетов: <queries>, а не QUERY_ALL_PACKAGES -->
<queries>
    <intent><action android:name="android.intent.action.MAIN" />
            <category android:name="android.intent.category.LEANBACK_LAUNCHER" /></intent>
    <intent><action android:name="android.intent.action.MAIN" />
            <category android:name="android.intent.category.LAUNCHER" /></intent>
    <package android:name="com.android.systemui" />
    <package android:name="com.android.tv.settings" />
</queries>

<activity android:name=".HomeActivity" android:exported="true"
    android:launchMode="singleTask" android:stateNotNeeded="true"
    android:clearTaskOnLaunch="true" android:excludeFromRecents="true"
    android:screenOrientation="landscape"
    android:configChanges="keyboard|keyboardHidden|navigation|orientation|screenSize|screenLayout|uiMode|density">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.HOME" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

### Функции

- **Док** избранных приложений и **сетка** остальных, навигация пультом
- Список приложений — `queryIntentActivities` по `LEANBACK_LAUNCHER`, затем
  добор по `LAUNCHER` для сайдлоада без TV-активности
- Иконка — `ActivityInfo.loadBanner()`, при отсутствии `loadIcon()` + подпись
- **Настройка с пульта**: добавить/убрать из дока, переставить, скрыть из сетки,
  число колонок, обои вкл/выкл и интервал. Всё в `SharedPreferences`, **не в
  ресурсах** — иначе каждая правка требует пересборки APK
- Часы, дата, приветствие по времени суток
- **Панель быстрых настроек** по экранной кнопке
- Обои с кроссфейдом
- Погода под датой

### Что обязательно учесть

**Плитка 16:9 и `FIT_CENTER`.** Баннеры приложений имеют соотношение 16:9.
Считай высоту как `w * 9 / 16`, иначе `CENTER_CROP` срежет края логотипов.
Добавь внутренний отступ ~9×6 dp: свои поля есть не у всех баннеров, у некоторых
надпись упирается в край. Подложку под баннером не рисуй — получится рамка.

**Фокус запрашивай после вёрстки.** В `onResume` плитки ещё нулевого размера,
`requestFocus` молча не срабатывает, и `ScrollView` отдаёт фокус первому
попавшемуся — экран оказывается прокручен. Оборачивай в `view.post(...)`.

**Отслеживай установку приложений через `getChangedPackages(seq)`**, а не через
`PACKAGE_ADDED`: бродкаст, пришедший когда в фокусе было другое приложение,
просто теряется, а счётчик последовательности переживает это.

**Пустая сетка должна исчезать целиком**, а не оставлять заголовок над пустотой.
И предусмотри «вернуть все скрытые» одним действием: когда скрыто всё, до меню
отдельной плитки не добраться.

**Память.** На телевизоре часто 1.5–2 ГБ, и обои — самая дорогая часть:

- `inPreferredConfig = RGB_565` вместо ARGB_8888. Фото под затемнением,
  прозрачность не нужна: 1920×1080 стоит 4 МБ вместо 8
- `inSampleSize` считай от реального размера вью
- освобождай уходящий битмап по окончании кроссфейда, иначе оба слоя держат
  все показанные фотографии до конца сессии
- декодируй в фоновом потоке

### Установка лончера

```bash
adb install -r tvhome.apk
adb shell cmd package set-home-activity --user 0 <пакет>/.HomeActivity
adb shell pm disable-user --user 0 <штатный лончер>
adb shell pm disable-user --user 0 com.google.android.tungsten.setupwraith
adb shell am force-stop com.android.tv.settings
adb shell input keyevent KEYCODE_HOME
```

Штатный лончер: `com.google.android.apps.tv.launcherx` (Google TV) или
`com.google.android.tvlauncher` (старые Android TV). Проверь:

```bash
adb shell 'cmd package query-activities --brief -a android.intent.action.MAIN -c android.intent.category.HOME'
```

### Три ловушки установки

**`setupwraith` обязан быть выключен.** Его `RecoveryActivity` имеет приоритет
HOME выше любого стороннего лончера. При выключенном штатном она уходит в цикл
~20 перезапусков в секунду и выжигает `system_server` на 100% CPU. Ловится так:
номер задачи растёт на сотни за десяток секунд.

```bash
adb shell 'dumpsys activity activities | grep -m1 -oE "RecoveryActivity t[0-9]+"'   # дважды с паузой
adb shell 'top -b -n 1 -m 3 -o PID,%CPU'
```

**`am force-stop com.android.tv.settings`** в конце установки. Иначе остаётся
устаревшая home-задача от смены лончеров, и Home открывает новый лончер
*позади* текущего приложения.

**Обязательно протестируй Home изнутри запущенного приложения**, а не только
с домашнего экрана:

```bash
adb shell "monkey -p <любое приложение> -c android.intent.category.LEANBACK_LAUNCHER 1"
sleep 8 && adb shell input keyevent KEYCODE_HOME && sleep 5
adb shell 'dumpsys window | grep -m1 mFocusedApp'
```

### Сборка без Gradle

`aapt2 compile` → `aapt2 link` → `javac --release 11` → `d8 --lib android.jar`
→ `zipalign` → `apksigner`.

| Грабля | |
|---|---|
| **build-tools 34.0.0 может не собрать** | В нём dev-сборка R8 (`8.2.2-dev`), падающая с внутренним NPE `Cannot invoke "String.length()"` на классах с синтетическими bridge-методами (ковариантные переопределения, `Comparator<T>`). Бери 36.1.0 |
| `keytool -quiet` | Такого флага нет, нужен `-noprompt` |
| `d8` | Без `--lib android.jar` не сделает desugaring |

---

## 5. Обои как у штатной заставки

Если на телевизоре есть Google Ambient Mode, он складывает скачанные картинки
в кэш на общем хранилище:

```
/sdcard/Android/data/com.google.android.apps.tv.dreamx/files/Pictures/cache/
```

WebP 1920×1080 — ровно то, что телевизор показывает сам. Приложение чужой
`Android/data` с Android 11 читать не может, а `adb` может. Напиши скрипт,
который копирует их в **собственную** папку лончера
(`/sdcard/Android/data/<пакет>/files/wallpapers`) — оттуда он читает без
единого разрешения:

```bash
for f in $(adb shell "ls $SRC" | tr -d '\r' | grep '\.0$'); do
  adb shell "cp $SRC/$f $DST/$f"
done
```

Файлы без расширения (DiskLruCache пишет `<hash>.0`), так что не фильтруй
по расширению при чтении — проверяй результат декодирования.

---

## 6. Погода

Два бесплатных сервиса без ключей и регистрации:

| | |
|---|---|
| геолокация устройства | точнее всего, если доступна |
| `https://ipapi.co/json/` (запасной `https://ipwho.is/`) | когда позиции нет |
| `https://api.open-meteo.com/v1/forecast` | температура и `weather_code` (WMO) |

**`ip-api.com` не подходит**: бесплатный тариф только по HTTP, а с Android 9
открытый трафик запрещён —
`java.io.IOException: Cleartext HTTP traffic to ip-api.com not permitted`.

**Одной включённой геолокации мало.** У телевизора нет GPS, и после включения
служб `last location` у всех провайдеров остаётся `null` — включение само по
себе позицию не создаёт. Нужен активный `requestLocationUpdates` на сетевом
провайдере с ожиданием ~12 секунд; после него GMS считает фикс по окружающим
Wi-Fi-сетям с точностью около 100 м. Нет фикса — уходи на IP, но предупреди
пользователя: IP показывает город оператора, который может быть в другом конце
страны.

---

## 7. Если голосовой ассистент не работает

Сообщение «В этом профиле Ассистент пока недоступен» может быть **не связано**
с чисткой. Проверь:

```bash
adb shell 'pm list users'
adb shell 'dumpsys user'                 # Type: com.android.tv.profile, сколько аккаунтов
adb shell 'getprop | grep -iE "country|region|sku"'
```

Две находки, которые встретились нам: брошенный профиль Google TV с нулём
аккаунтов (`pm remove-user <id>`), и противоречивые региональные свойства
(`ro.product.region`, `ro.odm.*.countrycode`, `ro.ecid.country`,
`persist.sys.country` — пустая). Это build-time свойства в read-only разделах,
без рута не переписать. При битом регионе ассистент может быть не провижинен
в принципе, и смена языка интерфейса не помогает — язык в эту проверку
не входит.

---

## 8. Откат

```bash
adb shell pm enable <штатный лончер>
adb shell pm enable com.google.android.tungsten.setupwraith
adb shell cmd package set-home-activity --user 0 <штатный лончер>/.home.HomeActivity
adb uninstall <свой лончер>
# плюс скрипт из шага 2 — вернуть пакеты и настройки
```

Крайняя мера — **Настройки → Система → Об устройстве → Сброс к заводским**.
При залоченном бутлоадере и read-only прошивке настоящий кирпич через `pm`
невозможен, сброс вытащит всегда.

---

## Честно про цифры

Если будешь мерить память «до и после»: замер после недель uptime и замер сразу
после ребута **несравнимы** — у свежезагруженной системы RSS структурно выше,
ядро ещё ничего не вытеснило. Не рисуй из этих двух чисел красивую дельту.
Корректно так: перечисли процессы, которых больше нет совсем
(`adb shell pgrep -f <пакет>`), и скажи, что реальную картину будет видно через
несколько дней работы.

У нас вышло: пакетов 140 → 101, домашний экран 110 МБ PSS → ~25 МБ,
`/data` освободилось 700 МБ.
