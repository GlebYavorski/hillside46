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
| 7 | HK_MAC          | Hotkeys для macOS (пока пустой)          |
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
- Combos clipboard (cut/copy/paste) работают **только** на Win/Lin (Ctrl-based);
  на mac — Cmd+C/V вручную или ремап на стороне OS.
- Combo переключения языка: на Win/Lin — Punto Switcher (Ctrl+Shift+1/2), на mac —
  Ctrl+Space (оба combo, позиции 15-16 и 19-20). Combos гейтятся по самому верхнему
  активному слою, поэтому на mac-базе срабатывают только mac-combos, конфликта нет.
- Символьные combos и F-клавиши работают на обеих базах.
- Управление окнами на mac (HK_MAC / Rectangle) пока не реализовано — TODO после
  установки Rectangle.

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
