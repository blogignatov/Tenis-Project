# Tenis-Project
# Техническое задание

## Сайт-платформа молодёжного теннисного клуба

**Версия:** 1.0
**Дата:** 19 апреля 2026 г.
**Юрисдикция:** Российская Федерация
**Статус:** Готово к передаче разработчику

---

## 1. Общая информация

- **Название проекта:** Сайт-платформа молодёжного теннисного клуба (рабочее название — уточнить у заказчика).
- **Тип:** коммерческий веб-сайт + закрытая платформа учёта игровой практики с личным кабинетом.
- **Целевая аудитория:** взрослые любители тенниса (преимущественно молодёжь), администраторы клуба.
- **Юрисдикция:** Российская Федерация; все ПДн обрабатываются и хранятся на серверах, физически расположенных в РФ, согласно ч. 5 ст. 18 152-ФЗ (с учётом ужесточения от 01.07.2025).

---

## 2. Технологический стек (обязательный)

| Слой | Технология | Обоснование |
|---|---|---|
| Фронтенд | Next.js 14 (App Router), TypeScript, TailwindCSS, shadcn/ui, Recharts, TanStack Query, Zod | SSR для SEO лендинга, типобезопасность, готовые UI-компоненты |
| Бэкенд | Python 3.12, FastAPI, Pydantic v2, SQLAlchemy 2.0, Alembic | Переиспользование существующего алгоритма рейтинга из Tkinter-прототипа без переписывания на другой язык |
| БД | PostgreSQL 16 | Многопользовательский доступ, транзакции, advisory locks для пересчёта рейтинга |
| Кэш/очередь | Redis 7 | Сессии, rate-limit, фоновые задачи (Celery/RQ) |
| Авторизация | JWT access + refresh (httpOnly cookie), argon2id для паролей, TOTP 2FA для ролей admin/superadmin | — |
| Файлы | Yandex Object Storage (S3-совместимый) | Логотип, фото игроков, экспорты |
| Хостинг | **Yandex Cloud** (Managed PostgreSQL + Compute + Object Storage, регион ru-central1) | Требование 152-ФЗ о локализации ПДн в РФ |
| CI/CD | GitHub Actions → Docker → Yandex Container Registry → Yandex Managed Kubernetes **или** VM с docker-compose | — |
| Мониторинг | Yandex Monitoring, Sentry self-hosted (в РФ), логи в Yandex Cloud Logging | ПДн не должны уходить в зарубежные сервисы |
| Домен | .ru, регистратор RU-Center / Reg.ru | — |

**Запрещено:** Vercel/Netlify для прод-хостинга, AWS/GCP/Azure, Cloudflare для WAF с данными ПДн, Supabase/Firebase, зарубежные почтовые сервисы (SendGrid, Mailgun). Почта — Unisender/SendPulse (РФ) или собственный SMTP.

---

## 3. Роли и права доступа

| Роль | Основные возможности |
|---|---|
| `guest` | Просмотр лендинга, публичного рейтинга (никнейм/уровень/место), публичного расписания турниров, правил |
| `player` | Всё из guest + свой ЛК: полная история матчей, график уровня, запись на турниры, изменение своих данных, загрузка аватара |
| `admin` | Всё из player + CRUD турниров, ввод результатов матчей, управление регистрациями, экспорт. Обязательно 2FA |
| `superadmin` | Всё из admin + управление админами, просмотр audit_log, ручная корректировка рейтинга с обязательным комментарием |

---

## 4. Модель данных (PostgreSQL)

### 4.1. Ключевые таблицы

```sql
-- Пользователи и аутентификация
users (
  id UUID PK,
  email CITEXT UNIQUE NOT NULL,
  phone_e164 VARCHAR(16),
  password_hash TEXT NOT NULL,           -- argon2id
  role VARCHAR(16) NOT NULL,             -- player/admin/superadmin
  totp_secret TEXT,                      -- для admin/superadmin
  is_active BOOLEAN DEFAULT true,
  email_verified_at TIMESTAMPTZ,
  consent_pdn_version INT,               -- версия согласия на обработку ПДн
  consent_pdn_at TIMESTAMPTZ,
  consent_public_rating BOOLEAN,         -- согласие на публикацию в рейтинге
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ
);

-- Профиль игрока
players (
  user_id UUID PK REFERENCES users(id),
  public_nickname VARCHAR(32) UNIQUE NOT NULL,
  first_name VARCHAR(64) NOT NULL,
  last_name VARCHAR(64) NOT NULL,
  birth_date DATE,
  gender VARCHAR(8),
  level_value NUMERIC(3,2) NOT NULL CHECK (level_value BETWEEN 1 AND 7),
  level_updated_at TIMESTAMPTZ,
  rating_points INT DEFAULT 0,
  avatar_url TEXT,
  joined_at DATE
);

-- События
events (
  id UUID PK,
  type VARCHAR(16) NOT NULL,              -- tournament/championship/official_match
  title VARCHAR(128) NOT NULL,
  category VARCHAR(32) NOT NULL,
  min_level NUMERIC(3,2) NOT NULL,
  max_level NUMERIC(3,2) NOT NULL,
  starts_at TIMESTAMPTZ NOT NULL,
  ends_at TIMESTAMPTZ,
  location VARCHAR(128),
  max_participants INT,
  registration_opens_at TIMESTAMPTZ,
  registration_closes_at TIMESTAMPTZ,
  status VARCHAR(16) NOT NULL,            -- draft/published/registration/in_progress/finished/cancelled
  bracket_json JSONB,                     -- турнирная сетка
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Регистрации
registrations (
  id UUID PK,
  event_id UUID REFERENCES events(id),
  player_id UUID REFERENCES players(user_id),
  status VARCHAR(16),                     -- pending/confirmed/waitlist/cancelled/no_show
  registered_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(event_id, player_id)
);

-- Матчи
matches (
  id UUID PK,
  event_id UUID REFERENCES events(id),
  round INT,
  player_a UUID REFERENCES players(user_id),
  player_b UUID REFERENCES players(user_id),
  score_json JSONB NOT NULL,              -- [{set:1,a:6,b:4}, {set:2,a:7,b:5}]
  winner UUID,
  played_at TIMESTAMPTZ,
  entered_by UUID REFERENCES users(id),
  verified_by UUID,
  verified_at TIMESTAMPTZ
);

-- История рейтинга (append-only, для графиков и аудита)
rating_history (
  id BIGSERIAL PK,
  player_id UUID REFERENCES players(user_id),
  event_id UUID REFERENCES events(id),
  match_id UUID REFERENCES matches(id),
  delta_points INT,
  points_before INT,
  points_after INT,
  level_before NUMERIC(3,2),
  level_after NUMERIC(3,2),
  reason VARCHAR(32),                     -- match_result/manual_correction/decay
  comment TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Аудит всех действий админов
audit_log (
  id BIGSERIAL PK,
  actor_user_id UUID,
  action VARCHAR(64),
  entity VARCHAR(32),
  entity_id UUID,
  diff_json JSONB,
  ip_address INET,
  user_agent TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Согласия на обработку ПДн (юридический след)
pdn_consents (
  id BIGSERIAL PK,
  user_id UUID REFERENCES users(id),
  document_version INT,
  accepted_at TIMESTAMPTZ,
  ip_address INET,
  document_hash TEXT                      -- sha256 версии документа
);
```

### 4.2. Индексы

- `players(level_value)`
- `events(starts_at, status)`
- `rating_history(player_id, created_at DESC)`
- `matches(event_id)`
- `audit_log(created_at DESC)`

---

## 5. Атомарный пересчёт рейтинга

При вводе результата матча админом выполняется **одна транзакция с PostgreSQL advisory lock**, чтобы исключить гонки при параллельном вводе нескольких матчей одного игрока:

```python
async def apply_match_result(session, match_id: UUID):
    async with session.begin():
        # Блокировка на пару игроков (детерминированный ключ)
        await session.execute(
            text("SELECT pg_advisory_xact_lock(hashtext(:k))"),
            {"k": f"match:{match_id}"}
        )
        match = await session.get(Match, match_id, with_for_update=True)
        # Пересчёт по регламенту клуба (существующий Python-алгоритм)
        deltas = rating_engine.calculate(match, players_state)
        for d in deltas:
            await session.execute(update_player_level(...))
            await session.execute(insert_rating_history(...))
        await session.execute(insert_audit_log(...))
```

Уровень изоляции транзакции — `READ COMMITTED` (по умолчанию); целостность обеспечивается advisory lock + `FOR UPDATE`.

---

## 6. API-контракт

Базовый префикс: `/api/v1`. Аутентификация: `Authorization: Bearer <access_token>` либо httpOnly cookie. Все изменяющие методы — CSRF-токен.

### 6.1. Публичные эндпоинты

- `GET /events?status=registration&type=tournament&from=&to=` — список событий.
- `GET /events/{id}` — карточка события (без ПДн зарегистрированных; только никнеймы).
- `GET /rating?limit=50&offset=0&category=` — публичный рейтинг (никнейм, уровень, очки).
- `GET /rating/{nickname}/public` — публичная карточка игрока (без ФИО/телефона/email).
- `GET /content/club` — контент лендинга (о клубе, услуги, локации).

### 6.2. Аутентификация

- `POST /auth/register` — регистрация (email, пароль, согласие на ПДн обязательно).
- `POST /auth/login` — логин, возвращает access+refresh.
- `POST /auth/refresh`
- `POST /auth/logout`
- `POST /auth/password/reset-request`
- `POST /auth/password/reset-confirm`
- `POST /auth/2fa/enroll`
- `POST /auth/2fa/verify` — для админов.

### 6.3. Личный кабинет игрока

- `GET /me` — профиль с ПДн.
- `PATCH /me` — обновление телефона, аватара, настроек приватности.
- `GET /me/matches?limit=&offset=` — история матчей.
- `GET /me/rating-history?from=&to=` — данные для графика уровня.
- `POST /events/{id}/register` — запись на турнир (валидация: уровень игрока ∈ [min_level, max_level]).
- `DELETE /events/{id}/register` — отмена регистрации.

### 6.4. Админка

- `POST /admin/events`
- `PATCH /admin/events/{id}`
- `POST /admin/events/{id}/publish`
- `POST /admin/events/{id}/bracket/generate` — генерация сетки.
- `POST /admin/events/{id}/matches` — создание матча.
- `PUT /admin/matches/{id}/score` — ввод/корректировка счёта → триггер пересчёта рейтинга.
- `POST /admin/players` — ручное создание игрока (без self-регистрации).
- `POST /admin/rating/correction` — ручная корректировка (обязателен `comment`, логируется в audit_log).
- `GET /admin/export/event/{id}.xlsx` — экспорт результатов.
- `GET /admin/audit-log?from=&to=&actor=` — только superadmin.

**OpenAPI-спецификация** генерируется автоматически FastAPI и публикуется на `/api/docs` (только для окружения dev/staging; в prod — закрыто).

---

## 7. Фронтенд: карта страниц

### 7.1. Публичные

`/`, `/about`, `/services`, `/locations`, `/camps`, `/tournaments`, `/tournaments/[id]`, `/rating`, `/team`, `/contacts`, `/legal/privacy`, `/legal/offer`, `/legal/terms`.

### 7.2. ЛК игрока

`/cabinet`, `/cabinet/profile`, `/cabinet/matches`, `/cabinet/rating`, `/cabinet/registrations`.

### 7.3. Админка

`/admin`, `/admin/events`, `/admin/events/[id]/bracket`, `/admin/events/[id]/matches`, `/admin/players`, `/admin/rating`, `/admin/audit` (superadmin).

### 7.4. Визуализация регламента

Отдельный раздел `/how-it-works` со схемой «уровень → диапазоны категорий → очки → скидки» (SVG-инфографика).

---

## 8. Разграничение публичного и приватного (152-ФЗ)

| Атрибут | Публично | Только в ЛК |
|---|---|---|
| Никнейм, уровень, место в рейтинге, очки | ✓ (с согласия `consent_public_rating`) | |
| Счёт матча, раунд, событие | ✓ | |
| ФИО полностью | | ✓ |
| Email, телефон, дата рождения | | ✓ |
| Аватар (фото) | Опционально (чекбокс) | ✓ |
| История изменений уровня | Агрегированно по нику | Детально в ЛК |

Все ПДн собираются **только после явного согласия** с политикой обработки ПДн (отдельный чекбокс, не дефолтный); версия и хэш документа фиксируются в `pdn_consents`.

---

## 9. Безопасность (обязательный минимум для прод)

- HTTPS везде, HSTS, TLS 1.2+.
- Хеширование паролей: argon2id (t=3, m=64MB, p=4).
- Rate limiting: 5 попыток логина / 15 мин / IP, 100 req/мин общий для API.
- Защита от CSRF (double-submit cookie), XSS (CSP strict), SQLi (только параметризованные запросы через SQLAlchemy).
- 2FA (TOTP) для всех админов, обязательна к enrollment на первом входе.
- Бэкапы PostgreSQL: ежедневные полные + WAL, хранение 30 дней, тестовое восстановление раз в месяц.
- Логи доступа и действий админов — 1 год минимум.
- Pen-test базовый (OWASP Top-10) перед запуском.
- Dependency scanning (GitHub Dependabot + `pip-audit` + `npm audit`) в CI.

---

## 10. Юридические документы (обязательно до запуска)

1. Политика обработки персональных данных (публичная на сайте).
2. Согласие на обработку ПДн (версионируется).
3. Согласие на публикацию ПДн в открытом доступе (никнейм/фото в рейтинге).
4. Пользовательское соглашение.
5. Публичная оферта (если на сайте есть оплата/услуги).
6. Уведомление о cookies.
7. Уведомление Роскомнадзора о намерении обрабатывать ПДн — подаётся до запуска.

Шаблоны готовит юрист; разработчик встраивает их в роутинг и версионирование.

---

## 11. План работ и приёмка

| Спринт (2 нед) | Состав | Артефакты приёмки |
|---|---|---|
| S1 | Инфраструктура (Yandex Cloud, VPC, PG, Redis, CI/CD), каркас FastAPI+Next.js, аутентификация | Регистрация/логин/2FA работает, деплой на staging |
| S2 | Рефакторинг Python-алгоритма рейтинга в модуль, покрытие pytest (≥ 90% на рейтинг-движок), миграция SQLite→PG | Набор юнит-тестов с референсными результатами от заказчика |
| S3 | Админка: CRUD турниров, игроков, ввод матчей, пересчёт рейтинга с advisory lock | Приёмка на тестовом наборе: 20 матчей → ожидаемые уровни |
| S4 | Публичный рейтинг, лендинг, разделы «о нас/услуги/локации/кэмпы» | Смоук-тест со стороны заказчика |
| S5 | ЛК игрока, запись на турниры, графики | e2e-тесты Playwright |
| S6 | Юр. тексты, согласия, уведомление РКН, бэкапы, мониторинг, нагрузочный тест (k6: 200 RPS), базовый пентест | Чек-лист прод-готовности |

### Definition of Done (проект в целом)

- Все пользовательские сценарии из раздела 7 проходят вручную по чек-листу.
- Алгоритм рейтинга воспроизводит результаты Tkinter-прототипа на идентичных входах бит-в-бит.
- Сканер OWASP ZAP не находит High/Critical.
- RPO ≤ 24 ч, RTO ≤ 4 ч (восстановление из бэкапа подтверждено).
- Страница `/legal/privacy` версионирована, согласие записывается в БД.

---

## 12. Ответственность заказчика

- Регламент игровой практики в текстовом виде + формулы пересчёта (источник правды).
- Тестовые кейсы: 15–30 матчей с ожидаемыми уровнями «до/после».
- Брендбук: логотип в SVG, фирменные HEX, шрифты.
- Контент: тексты разделов, фото локаций, команда.
- Юр. лицо/ИП для регистрации оператора ПДн и домена.
- Финальный список ролей админов и их ФИО для создания учёток.

---

*Конец документа.*
