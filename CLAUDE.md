# Hillside46 ZMK Config

ZMK firmware keymap for Hillside46 split keyboard (nice_nano_v2 controller).

## Key files

- `config/hillside46.keymap` — главный файл: слои, поведения, макросы
- `config/combos.dtsi` — все комбо-клавиши (включается в keymap)
- `config/hillside46.conf` — настройки Kconfig (bluetooth, sleep, RGB и т.д.)
- `config/west.yml` — версия ZMK (сейчас pinned на v0.3.0)
- `build.yaml` — матрица сборки для GitHub Actions (hillside46_left + hillside46_right)

## Слои

Порядок важен: базы (DEF, DEF_MAC) — самые нижние, чтобы overlay-слои стекались
поверх любой из них; ADJ — самый верхний, чтобы перекрывать NAV_MAC при вызове
через NUM+NAV_MAC.

| # | Название        | Описание                                 |
|---|-----------------|------------------------------------------|
| 0 | DEF             | QWERTY база (Win/Lin), homerow mods      |
| 1 | DEF_MAC         | QWERTY база для macOS                     |
| 2 | NUM             | Цифры, символы, F-клавиши (общий)        |
| 3 | NAV             | Навигация, символы, медиа (Win/Lin)      |
| 4 | HK              | Hotkeys (Alt+N комбо, window management) |
| 5 | FZ              | FancyZones, управление окнами (Win)      |
| 6 | NAV_MAC         | Навигация для macOS                      |
| 7 | HK_MAC          | Тайлинг окон + Spaces (macOS)            |
| 8 | ADJ             | Bluetooth, system, переключение ОС       |

ADJ активируется через conditional_layer при одновременном удержании NUM и NAV
(работает и с NAV, и с NAV_MAC — два tri_layer: `<2 3>` и `<2 6>` → 8).

## Целевые ОС (Win/Lin ↔ macOS)

Конфиг держит две параллельные базы:
- **Win/Lin** (по умолчанию при загрузке): слои DEF / NAV / HK / FZ
- **macOS**: слои DEF_MAC / NAV_MAC / HK_MAC

Переключение — клавишами в слое ADJ (верхний ряд левой половины, над BT_SEL 0/1):
- над BT_SEL 0 → `&to DEF` (Win/Lin база)
- над BT_SEL 1 → `&to DEF_MAC` (mac база)

Конвенция модификаторов на mac: `LGUI`=Cmd, `LALT`=Option, `LCTRL`=Ctrl.

### Известные ограничения mac-режима
- При загрузке/смене батареи клавиатура всегда стартует на Win-базе (layer 0) —
  ZMK штатно не запоминает выбор. Нужно нажать переключатель в ADJ.
- Combos гейтятся по самому верхнему активному слою (`combo_active_on_layer` в ZMK),
  поэтому на mac-базе срабатывают только mac-combos, на win-базе — win-combos, конфликта нет.
- Clipboard: Win/Lin — Ctrl (cut/copy/paste), mac — Cmd (LG(X)/LG(C)/LG(V)).
- Переключение языка: Win/Lin — Punto Switcher (Ctrl+Shift+1/2), mac — Ctrl+Space
  (оба combo, позиции 15-16 и 19-20).
- Символьные combos и F-клавиши работают на обеих базах.

### Зависимости на стороне macOS
- **Переключение вкладок** (NAV_MAC, Cmd+Opt+←/→): нативно в Firefox/Chrome/VSCode.
  Для Ghostty добавить в `~/.config/ghostty/config`:
  `keybind = super+alt+right=next_tab` и `keybind = super+alt+left=previous_tab`.
- **Cmd+Tab** (DEF_MAC, на месте AC-Tab) переключает между приложениями (тап = два
  последних приложения). Для переключения между ОКНАМИ (как alt-tab в Windows) — поставить
  приложение AltTab; окна одного приложения — Cmd+` (backtick).
- **Переключение столов** (HK_MAC, правая рука): `H`=`Ctrl+←`, `L`=`Ctrl+→` —
  встроено в macOS, настройки не требует.
- **Тайлинг окон** (HK_MAC, левая рука): нативный тайлинг macOS Sequoia/Tahoe, но
  дефолты завязаны на клавишу Globe (её ZMK слать не умеет). Поэтому слой шлёт
  аккорды Hyper (`Ctrl+Opt+Cmd`+буква), которые надо разово привязать к пунктам меню
  в **System Settings → Keyboard → Keyboard Shortcuts → App Shortcuts → All Applications**
  (`+`, ввести имя пункта меню точь-в-точь, нажать аккорд):

  | Клавиша слоя | Аккорд | Пункт меню (Window → Move & Resize) |
  |---|---|---|
  | Q / W / E | ⌃⌥⌘+Q / W / E | Top Left / Top / Top Right |
  | A / S / D | ⌃⌥⌘+A / S / D | Left / Fill / Right |
  | Z / X / C | ⌃⌥⌘+Z / X / C | Bottom Left / Bottom / Bottom Right |
  | F / R     | ⌃⌥⌘+F / R     | Center / Return to Previous Size |

## Сборка

Через GitHub Actions — push в main триггерит сборку, артефакты (.uf2) скачиваются из workflow.

## Соглашения

- Документация ведётся прямо в комментариях в коде
- Каждый слой должен иметь ASCII-диаграмму раскладки в комментарии

## Git

- `origin` → форк пользователя: `https://github.com/alparo/hillside46.git`
- `upstream` → оригинал: `https://github.com/mmccoyd/zmk-config.git`
- Пушить только в `origin`

## Documentation

- we try to keep the comments in the `config/hillside46.keymap` updated
- when adding keymap diagrams use following letters for modifiers:
  G - Win/Gui/Meta
  A - Alt
  S - Shift
  C - Control
- always exactly in this order. So, for example:
  - for alt+win+h the diagram should have GA-h
  - for shift+ctrl+gui+alt+right it will be GASC-->
  - for alt+5: A-5
