# Анализ и рекомендации по оптимизации `app-0-13-86.py`

> Fun Tennis — Rating (Tkinter + SQLite, однофайловое приложение).
> Объём: ~18 334 строк, ~665 КБ, 3 класса (`ScoreDialog`, `PlayerCardDialog`, `App`), ~50 функций верхнего уровня и ~384 метода.

**Важно:** все рекомендации ниже — *без изменения логики расчётов и алгоритмов*. Правила начисления очков, `duel_table_levels`, `recompute_levels_global`, `recompute_points_for_event`, `round_robin_rank_rows`, `recompute_tournament_results`, сидинг чемпионатов и т. д. сохраняются как есть. Речь идёт о структуре кода, читаемости, надёжности и производительности вокруг этой логики.

---

## 1. Общая структура проекта

### 1.1. Проблема: один файл на ~18 тыс. строк
Сейчас всё (константы, миграции БД, расчёты, Tkinter-UI, экспорт в Excel, диалоги, рендер сеток на Canvas) находится в одном `.py`-файле. Это:
- замедляет навигацию в IDE;
- мешает повторному использованию кода;
- усложняет тестирование отдельных блоков (особенно рейтинговой логики);
- провоцирует дублирование — одни и те же SQL-запросы встречаются в нескольких местах.

### 1.2. Рекомендуемое разделение на модули
Разделить строго по слоям, не меняя поведения функций:

```
fun_tennis/
├── __main__.py              # точка входа, запуск App()
├── config.py                # APP_DIR, DB_PATH, BACKUP_DIR, CATEGORIES,
│                            # CH_CATEGORIES, DEFAULT_LEVEL_RANGES,
│                            # RULES_SWITCH_DATE, OLD_/NEW_POINTS_PER_WIN,
│                            # OLD_/NEW_CH_POINTS, EVENT_STATUS
├── utils/
│   ├── dates.py             # now_iso, iso_to_ddmmyyyy, ddmmyyyy_to_iso, ...
│   ├── backup.py            # backup_db, backup_db_daily_on_start, cleanup_old_backups
│   └── tk_helpers.py        # _dateentry_stable и прочие UI-утилиты
├── db/
│   ├── connection.py        # connect(), контекстный менеджер
│   ├── schema.py            # init_db() — CREATE TABLE/INDEX и миграции
│   └── queries.py           # часто повторяющиеся SELECT/INSERT
├── scoring/
│   ├── rules.py             # event_uses_new_rules, points_per_win, ch_points
│   ├── levels.py            # duel_table_levels, _apply_cap, _cap_for_match,
│   │                        # recompute_levels_global, _ensure_event_level_baseline
│   ├── points.py            # recompute_points_for_event, _apply_manual_points_for_event,
│   │                        # _event_points_split_by_player
│   ├── results.py           # recompute_tournament_results, round_robin_rank_rows
│   └── rating.py            # rating_snapshot, player_rating_breakdown_full,
│                            # _best_points_by_round_for_kind, _recent_event_ids_for_kind
├── ui/
│   ├── app.py               # класс App — каркас + навигация по вкладкам
│   ├── tab_players.py
│   ├── tab_tournaments.py
│   ├── tab_championships.py
│   ├── tab_rating.py
│   ├── bracket_canvas.py    # отрисовка плей-офф и утешительной сетки
│   └── dialogs/
│       ├── score_dialog.py
│       └── player_card.py
└── export/
    └── excel.py             # openpyxl-экспорт
```

Такое разделение не влияет на алгоритмы — оно лишь переносит функции в несколько файлов. Все имена и сигнатуры сохраняются.

---

## 2. Работа с базой данных (SQLite)

### 2.1. Централизовать открытие соединения
Сейчас `connect()` вызывается ~122 раза по всему коду, из них огромное количество — в UI-колбэках (`refresh_*`, `on_*`). Типичный шаблон:

```python
con = connect()
# ... запросы ...
con.close()
```

Это приводит к утечкам соединений при исключениях и мешает пакетной работе в одной транзакции. Решение (без изменения логики) — контекстный менеджер:

```python
from contextlib import contextmanager

@contextmanager
def db():
    con = connect()
    try:
        yield con
        con.commit()
    except Exception:
        con.rollback()
        raise
    finally:
        con.close()
```

И заменять блоки `con = connect(); ...; con.close()` на `with db() as con: ...`. Поведение прежнее, надёжность выше.

### 2.2. Единообразные PRAGMA при подключении
Сейчас в `connect()` включены `foreign_keys = ON` и `busy_timeout = 5000`. Полезно добавить (без изменения алгоритмов):

```python
con.execute("PRAGMA journal_mode = WAL")
con.execute("PRAGMA synchronous = NORMAL")
con.execute("PRAGMA temp_store = MEMORY")
con.execute("PRAGMA cache_size = -20000")  # ~20 МБ page cache
```

Это даёт заметный прирост на массовых пересчётах (`recompute_levels_global`, `recompute_points_for_event`), не меняя ни одной формулы.

### 2.3. Миграции схемы
В `init_db()` для каждой новой колонки повторно читается `PRAGMA table_info(...)`:

```python
cols = [r["name"] for r in con.execute("PRAGMA table_info(players)").fetchall()]
if "initial_level" not in cols: ...
cols = [r["name"] for r in con.execute("PRAGMA table_info(players)").fetchall()]
if "is_active" not in cols: ...
```

Лучше прочитать один раз на таблицу:

```python
def _columns(con, table):
    return {r["name"] for r in con.execute(f"PRAGMA table_info({table})").fetchall()}

players_cols = _columns(con, "players")
if "initial_level" not in players_cols:
    con.execute("ALTER TABLE players ADD COLUMN initial_level REAL")
if "is_active" not in players_cols:
    con.execute("ALTER TABLE players ADD COLUMN is_active INTEGER NOT NULL DEFAULT 1")
```

Логика миграций идентична.

### 2.4. Недостающие/полезные индексы
Уже есть хорошие индексы (`idx_matches_event_stage`, `idx_points_ledger_event` и т. д.). Дополнительно:

- `CREATE INDEX IF NOT EXISTS idx_matches_winner ON matches(winner_id);`
- `CREATE INDEX IF NOT EXISTS idx_match_sets_match_setno ON match_sets(match_id, set_no);`
- `CREATE INDEX IF NOT EXISTS idx_level_ledger_event ON level_ledger(event_id);`
- `CREATE INDEX IF NOT EXISTS idx_tournament_results_event ON tournament_results(event_id);`
- `CREATE INDEX IF NOT EXISTS idx_championship_results_event ON championship_results(event_id);`

Ускоряет UI-выборки, не меняя их SQL.

### 2.5. `recompute_levels_global` — батч-вставки
Функция вставляет записи в `level_ledger` по одной (`con.execute(INSERT ...)`) в длинном цикле. Без изменения алгоритма:

- собрать список кортежей `ledger_rows` и выполнить один `con.executemany(...)` в конце;
- обернуть весь цикл в явный `BEGIN ... COMMIT` (в autocommit-режиме каждый `INSERT` — отдельная транзакция).

```python
ledger_rows = []
# внутри цикла:
ledger_rows.append((event_id, match_id, player_id, old_lvl, new_lvl, delta, reason, now))

con.executemany('''
    INSERT INTO level_ledger(event_id, match_id, player_id,
        old_level, new_level, delta, reason, created_at)
    VALUES(?,?,?,?,?,?,?,?)
''', ledger_rows)
```

То же применимо к финальному `UPDATE players SET level=? WHERE id=?` — `executemany` с тем же набором значений.

---

## 3. Расчётная часть (нетронутая логика, но аккуратнее)

Все формулы остаются как есть:

- `duel_table_levels(level_a, level_b, winner_side, sets_rows, cap)` — правила «борзый/бывалый/сосед», пороги 40% / 33%, сценарий 1:1 + супертайбрейк.
- `_cap_for_match` — cap 0.50 для championship/duel, 0.25 для турнира при разнице >0.25.
- `_is_two_player_group_match` + `_match_has_full_duel_cap` — особые случаи (cap = 0.50).
- `recompute_levels_global` — построение общей отсортированной ленты (manual-корректировки → матчи, стабильная сортировка по дате/стадии/раунду/позиции/slot/id).
- `recompute_points_for_event` — очки по правилам до/после `RULES_SWITCH_DATE = 2026-04-01`, специальные случаи WO/retired в группе из 2 игроков.
- `round_robin_rank_rows` — тай-брейки (H2H для 2, % геймов для 3+).
- `recompute_tournament_results` — места 1, 2, 3, 4, 5+, нижний блок (утешилка).
- `_best_points_by_round_for_kind` — окно 5 последних кругов, лучшее за круг, заглушки «0 очков», фильтр несыгранных категорий.
- Сидинг чемпионатов (`ch_get_seed_state`, `ch_set_seed_state`).

### 3.1. Чисто структурные улучшения
- Вынести таблицы `OLD_POINTS_PER_WIN`, `NEW_POINTS_PER_WIN`, `OLD_CH_POINTS`, `NEW_CH_POINTS` в отдельный модуль `scoring/rules.py` как *immutable* (`types.MappingProxyType`) — предотвратит случайную модификацию во время расчётов.
- Заменить «магические» строки стадий на константы:

```python
STAGE_GROUP = "GROUP"
STAGE_PO = "PO"
STAGE_PO_3P = "PO_3P"
STAGE_PO_C = "PO_C"
CH_STAGES = ("CH_R1", "CH_QF", "CH_SF", "CH_F", "CH_3P")
```

- В `_stage_order` словарь `order = {...}` создаётся заново при каждом вызове на каждом матче — вынести в модульный уровень (константа). Это ускорит `recompute_levels_global` на больших объёмах.

### 3.2. Явные типы и docstrings
Добавить type hints и краткие docstrings к ключевым функциям (`duel_table_levels`, `recompute_levels_global`, `recompute_points_for_event`), чтобы в будущем никто случайно не «поправил» формулы. Сами формулы не трогаем.

### 3.3. Тесты на регрессию (без изменения логики)
- `tests/test_duel_table_levels.py` — табличные тесты всех веток (борзый выиграл; бывалый выиграл, но борзый взял ≥40%; соседи <33%; 1:1 + STB и т. п.).
- `tests/test_points_old_vs_new.py` — проверка точек переключения правил по `RULES_SWITCH_DATE`.
- `tests/test_round_robin.py` — тай-брейки (H2H и % геймов).
- `tests/test_recompute_tournament_results.py` — разные размеры сетки и утешилки.

Для этого удобно, что расчётные функции уже умеют принимать `con=...` — тесты можно гонять на in-memory SQLite.

---

## 4. Производительность горячих мест

### 4.1. `recompute_levels_global`
При большой истории (тысячи матчей) сейчас внутри цикла выполняется отдельный `SELECT * FROM match_sets WHERE match_id=?` — N отдельных запросов. Без изменения алгоритма можно один раз сделать:

```python
sets_by_match = {}
for s in con.execute("SELECT * FROM match_sets ORDER BY match_id, set_no"):
    sets_by_match.setdefault(int(s["match_id"]), []).append(s)
```

и внутри цикла брать `sets_rows = sets_by_match.get(mid, [])`. Результат тот же, обращений к БД — одно вместо N.

Финальные `UPDATE players SET level=?` — тоже выполнять через `executemany`, обернув в транзакцию.

### 4.2. `_best_points_by_round_for_kind`
В режиме «TOTALS» уже используется один SUM-запрос — это хорошо. В режиме «BREAKDOWN» для конкретного игрока запрос тоже один; при большой истории можно дополнительно один раз построить словарь `round_no_by_event` и не вызывать `json.loads(ev["settings_json"])` внутри каждого цикла.

### 4.3. `refresh_*` методы UI
Многие методы обновления (`refresh_playoff`, `refresh_consolation`, `refresh_ev_matches`, `refresh_events`, `refresh_ch_events`) при каждом вызове:
- открывают соединение,
- делают несколько `SELECT`,
- полностью перезаполняют Treeview/Canvas.

Без изменения поведения:
- использовать один `with db()` на весь `refresh_*`;
- вынести повторяющиеся JOIN’ы (`matches JOIN players pa JOIN players pb`) в вспомогательную функцию `list_matches_for_event(con, event_id, stages)`;
- при массовых вставках в Treeview делать `tree.delete(*tree.get_children())` один раз до вставки (уже сделано — оставить).

### 4.4. Отрисовка Canvas (плей-офф и утешилка)
В `refresh_playoff_bracket` / `refresh_consolation_bracket`:
- шрифты `text_font`, `winner_font` лучше создать один раз на уровне класса (`self._bracket_text_font = tkfont.Font(...)`), а не при каждом перерисовывании;
- код `draw_text_box` + вычисление ширины имени (`approx_name_w = max(25, min(140, len(winner_name) * 7))`) — вынести в отдельный метод `_draw_match_line(...)`, чтобы не дублировать между основной и утешительной сетками.

---

## 5. UI-код и класс `App`

### 5.1. Декомпозиция мега-класса
Класс `App` содержит сотни методов. Без изменения логики — выделить Mixin-классы по вкладкам:

```python
class App(tk.Tk,
          PlayersTabMixin,
          TournamentsTabMixin,
          ChampionshipsTabMixin,
          RatingTabMixin):
    ...
```

Общий код (`_ensure_editable`, `_ensure_active_for_play`, `_update_tournament_action_states`, `_auto_close_byes`) — в базовый `AppCoreMixin`.

### 5.2. Единый стиль сообщений
Методы часто используют `messagebox.showinfo/showerror/askyesno` с разными заголовками. Константы:

```python
TITLE_CONFIRM = "Подтверждение"
TITLE_DONE    = "ОК"
TITLE_ERROR   = "Ошибка"
```

Поведение не меняется, UX становится предсказуемее.

### 5.3. `_dateentry_stable`
Функция хрупкая (ловит `FocusOut`, управляет `grab_set/grab_release`). Без изменения логики:
- сузить блоки `try/except` (сейчас много голых `except Exception: pass`, что маскирует реальные баги);
- логировать через `logging` вместо `print(...)`.

### 5.4. Логирование
Сейчас ошибки печатаются через `print("Backup cleanup delete error:", e, path)` и т. п. Лучше:

```python
import logging
log = logging.getLogger("fun_tennis")
logging.basicConfig(
    level=logging.INFO,
    filename=os.path.join(APP_DIR, "app.log"),
    format="%(asctime)s %(levelname)s %(name)s: %(message)s",
)
```

И везде `log.exception(...)` / `log.warning(...)`. Поведение не меняется, но ошибки видны после инцидента.

---

## 6. Надёжность и безопасность данных

### 6.1. Транзакции вокруг «тяжёлых» операций
В коде уже есть `con.execute("BEGIN")` + `con.rollback()/commit()` при создании плей-офф/утешилки — правильно. Тот же шаблон стоит распространить на:
- `recompute_levels_global`,
- `recompute_points_for_event`,
- `recompute_tournament_results`.

Логика расчётов не меняется, но результат становится атомарным (либо всё записалось, либо ничего).

### 6.2. Бэкапы
`backup_db_daily_on_start` и `cleanup_old_backups` — хорошие. Улучшения без смены логики:
- использовать `pathlib.Path` вместо `os.path`;
- добавить «бэкап перед каждым recompute» через уже существующий `backup_db("pre-recompute")` — особенно актуально, потому что `recompute_levels_global` делает `DELETE FROM level_ledger`.

### 6.3. Проверка целостности БД при запуске
Один раз в день (или при запуске) выполнять:

```python
con.execute("PRAGMA integrity_check").fetchone()
```

и, если результат не `ok`, автоматически предлагать восстановление из последнего бэкапа в `backups/`. Не влияет на расчёты, но защищает данные.

---

## 7. Мелкие точечные замечания

- В `_ensure_event_level_baseline` динамический IN-список формируется через `','.join(['?']*len(pids))` — безопасно, оставить; вынести в утилиту `in_placeholders(n)`.
- `short_canvas_name` и `format_match_score` — хорошие кандидаты на unit-тесты, поведение не меняется.
- В `ScoreDialog._next_key` порядок полей захардкожен — вынести в константы класса (`_ONE_ORDER`, `_BO3_ORDER`).
- `_clamp_level` дублирует идею «ограничить 1.0..7.0» — имеет смысл сделать `LEVEL_MIN, LEVEL_MAX = 1.0, 7.0` модульными константами и использовать везде (в т. ч. в UI-валидации), не меняя поведения.
- `now_iso()` вызывается в циклах (`_insert_points`, вставки в `level_ledger`) — можно вычислить один раз в начале операции и передать во все вставки; наблюдаемое поведение не меняется, количество системных вызовов уменьшается.

---

## 8. Предлагаемый порядок рефакторинга

Двигаться маленькими безопасными шагами, каждый из которых не меняет алгоритмы:

1. Ввести `logging` и `with db()`-контекстменеджер; заменить `print` на `log.*`.
2. Добавить PRAGMA `WAL`/`synchronous=NORMAL` и индексы из п. 2.4.
3. Вынести константы (`STAGE_*`, `LEVEL_MIN/MAX`, словарь `_stage_order`) на модульный уровень.
4. Собрать `match_sets` одним запросом в `recompute_levels_global`; заменить одиночные `INSERT` на `executemany`; обернуть в транзакцию.
5. Разбить `App` на Mixin-классы по вкладкам (механический перенос методов).
6. Вынести расчётные функции в пакет `scoring/` и покрыть unit-тестами на in-memory SQLite.
7. Постепенно дробить файл по структуре из п. 1.2.

После каждого шага — прогонять регрессионные тесты и сверять `rating_snapshot()` и `player_rating_breakdown_full()` с эталоном на копии боевой БД.

---

## 9. Итого

- **Логика расчётов и алгоритмов не трогается.**
- Предложенные изменения: модульное разбиение файла, централизация соединений с БД, батч-вставки, PRAGMA/индексы, вынесение констант, корректное логирование, транзакционность тяжёлых операций и тесты на регрессию.
- Такой рефакторинг делает код поддерживаемым, ускоряет `recompute_*`-функции на больших историях матчей и резко снижает риск потери данных при сбоях, сохраняя те же результаты рейтинга, очков и мест, что и сейчас.
