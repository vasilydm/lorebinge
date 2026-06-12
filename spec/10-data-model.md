# 10 · Модель данных и безопасность — Lorebinge SPEC (RN9)

> Часть разбитого спека. Источник-архив: [`spec-rn9.md`](spec-rn9.md) — не редактировать. Содержит: §6 — схема БД (PostgreSQL/Supabase), §7 — RLS. Это единый контракт данных. Оглавление, граф зависимостей и карта перекрёстных ссылок — в [00-index.md](00-index.md).

-----

## 6. Модель данных (схема БД, PostgreSQL/Supabase)

> **Без изменений по сути.** Бэкенд платформо-независим. Изменения для второй платформы — только в полях, связанных с платежами (`subscriptions.platform`, `payment_events.platform`/`store`). Все таблицы — в схеме `public`, кроме `auth.users` (управляется Supabase Auth). У всех таблиц `created_at timestamptz default now()`. Где есть изменения — `updated_at` с триггером. Все id — `uuid default gen_random_uuid()`, кроме явно указанных.

> **Политика `ON DELETE` для внешних ключей (контракт ссылочной целостности).** Раньше не была задана — закрыто по ревизии БД (B-03/C-03). Действует **каскадная цепочка от аккаунта**:
> - `profiles.id` → `auth.users(id)` **`ON DELETE CASCADE`**. Это ключевое звено: удаление аккаунта делается **одним** вызовом `auth.admin.deleteUser(uid)` (Edge Function `delete-account` с сервисной ролью, §8), а БД каскадом вычищает всё остальное. Прямого `delete from auth.users` из RPC не делаем.
> - **Пользовательские таблицы** (`streaks`, `user_course_progress`, `user_lesson_progress`, `user_task_attempts`, `diamond_transactions`, `subscriptions`) → `profiles(id)` **`ON DELETE CASCADE`**.
> - **Контент** (`lessons` → `courses`, `tasks` → `lessons`, `media_assets` → `tasks`) → **`ON DELETE CASCADE`** от родителя (удаление курса в админке/конвейере чистит уроки→задания→медиа).
> - **Аудит переживает удаление аккаунта:** `payment_events.user_id` и `error_logs.user_id` → `profiles(id)` **`ON DELETE SET NULL`** (финансовую/диагностическую историю не теряем; `user_id` поэтому nullable — см. ниже).

### 6.1 Пользователь и геймификация

**`profiles`** (1:1 с `auth.users`)

|Поле                     |Тип            |Примечание                 |
|-------------------------|---------------|---------------------------|
|`id`                     |uuid PK        |= `auth.users.id`          |
|`username`               |text unique    |                           |
|`display_name`           |text           |                           |
|`knowledge_level`        |int default 1  |+1 за каждые `courses_per_level` (дефолт 5) пройденных курсов — §A4/B6.8 |
|`courses_completed`      |int default 0  |счётчик пройденных курсов (для расчёта уровня)|
|`diamonds`               |int default 300|стартовый баланс из конфига|
|`battery`                |int            |текущий заряд              |
|`battery_max`            |int            |макс. заряд (из конфига)   |
|`last_battery_refill_at` |timestamptz    |для расчёта восстановления |
|`created_at / updated_at`|timestamptz    |                           |

**`streaks`** (1:1 с profiles)

|Поле                |Тип                |Примечание                      |
|--------------------|-------------------|--------------------------------|
|`user_id`           |uuid PK FK→profiles|                                |
|`current_streak`    |int default 0      |                                |
|`longest_streak`    |int default 0      |                                |
|`last_activity_date`|date null          |дата последнего засчитанного дня|

> **Заморозка стрика — автоматическая, за алмазы** (§A5). Магазина заморозок и ручной покупки **нет**. При пропуске дня сервер сам списывает `streak_freeze_cost_per_day` алмазов за каждый пропущенный день (если баланс позволяет) и сохраняет стрик; если алмазов не хватает — стрик сбрасывается. Поэтому отдельного поля-счётчика заморозок в `streaks` нет — «валюта заморозки» это сами алмазы.

**`diamond_transactions`** (леджер; идемпотентность и аудит)

|Поле        |Тип        |Примечание                                                                                                  |
|------------|-----------|------------------------------------------------------------------------------------------------------------|
|`id`        |uuid PK    |                                                                                                            |
|`user_id`   |uuid FK    |                                                                                                            |
|`amount`    |int        |знаковое (+награда / −трата)                                                                                |
|`reason`    |text       |enum-строка: `signup_bonus`, `lesson_completion_reward`, `lesson_replay_reward`, `streak_freeze` (автосписание за пропущенный день, §A5)|
|`ref_id`    |uuid null  |ссылка на связанную сущность                                                                                |
|`created_at`|timestamptz|                                                                                                            |

> **Источники и единственный сток алмазов.** Алмазы **начисляются** за завершение урока (`lesson_completion_reward` — взвешенный рандом, §A6), за перепрохождение (`lesson_replay_reward`) и стартовым бонусом (`signup_bonus`). **Тратятся** алмазы теперь **только** на автоматическую заморозку стрика (`streak_freeze`, §A5) — это единственный сток. Квесты как источник алмазов **убраны** (см. ниже §6.4). Награда в конце урока — **только алмазы** (заряд за урок не начисляется, §A4/§12.6).

### 6.2 Контент

**`categories`**

|Поле        |Тип        |
|------------|-----------|
|`id`        |uuid PK    |
|`name`      |text       |
|`slug`      |text unique|
|`sort_order`|int        |

**`courses`**

|Поле                     |Тип                    |Примечание                                                                                                                                                                                                                   |
|-------------------------|-----------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|`id`                     |uuid PK                |                                                                                                                                                                                                                             |
|`slug`                   |text unique            |человекочитаемый идентификатор курса (напр. `bards-blood`). Заполняется конвейером (Приложение C) и используется для **идемпотентного импорта** и диплинков. Для курсов, созданных в админке вручную, генерируется из `title`|
|`title`                  |text                   |                                                                                                                                                                                                                             |
|`category_id`            |uuid FK→categories null|конвейер генерирует **строку категории**, импорт резолвит её в `id` по таблице `categories` (см. Приложение C, C6)                                                                                                           |
|`description`            |text null              |                                                                                                                                                                                                                             |
|`cover_image_url`        |text null              |обложка курса для карточек **In Progress** / **Popular** / **Category** (§12.3в). Сетка вкладки **Grid** использует обложку первого урока (`lessons.cover_image_url`, §12.3б), а **не** это поле                                |
|`illustration_url`       |text null              |**главный логотип/иллюстрация карточки Lores (блок B)** — крупный по центру вверху                                                                                                                                           |
|`card_gradient_start`    |text null              |цвет градиента карточки Lores сверху (hex)                                                                                                                                                                                   |
|`card_gradient_end`      |text null              |цвет градиента карточки Lores снизу (hex)                                                                                                                                                                                    |
|`path_background_color`  |text null              |цвет фона экрана курса — сетки уроков (hex). *Имя поля историческое (раньше — фон «дороги»); семантика теперь «фон сетки уроков».*                                                                                           |
|`is_published`           |bool default false     |                                                                                                                                                                                                                             |
|`popularity_score`       |int default 0          |для блока Most Popular                                                                                                                                                                                                       |
|`sort_order`             |int                    |порядок в ленте Lores и категории                                                                                                                                                                                            |
|`created_at / updated_at`|timestamptz            |                                                                                                                                                                                                                             |


> **Счётчики на карточке Lores** (`количество уроков`, `количество заданий`) — **не хранятся в `courses`**, а вычисляются: `lessons_count = count(lessons WHERE course_id)`, `tasks_count = count(tasks JOIN lessons WHERE course_id)`. Возвращаются агрегатом в `get_home_feed` (раздел 8.1).

> **Происхождение контента.** Курсы попадают в `courses`/`lessons`/`tasks`/`media_assets` двумя путями: (1) вручную через админку (Приложение B, CMS), (2) автоматически через контент-конвейер (Приложение C). Конвейер хранит рабочее состояние в **отдельной схеме `pipeline`**. Импорт из `pipeline` в `public` идемпотентен по `courses.slug`.

**`lessons`** (карточки уроков в сетке курса)

|Поле                     |Тип            |Примечание                                                                                                                                                                                                                      |
|-------------------------|---------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|`id`                     |uuid PK        |                                                                                                                                                                                                                                |
|`course_id`              |uuid FK→courses|                                                                                                                                                                                                                                |
|`title`                  |text           |                                                                                                                                                                                                                                |
|`cover_image_url`        |text null      |**обложка карточки урока в сетке курса** (§12.4). Генерируется конвейером (B3 `lesson_cover`, Приложение C) по теме урока; для уроков из админки — загружается там же. При отсутствии — placeholder (`expo-image`, A13). **Обложка первого урока курса** (мин. `sort_order`) дополнительно служит ячейкой вкладки **Grid** (§12.3б) — тот же кадр, что в соц-промо|
|`sort_order`             |int            |порядок в сетке уроков (редактируется в админке)                                                                                                                                                                                |
|`created_at / updated_at`|timestamptz    |                                                                                                                                                                                                                                |

**`tasks`** (задания в уроке)

|Поле                     |Тип            |Примечание                                                                                                                                                                                                                               |
|-------------------------|---------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|`id`                     |uuid PK        |                                                                                                                                                                                                                                         |
|`lesson_id`              |uuid FK→lessons|                                                                                                                                                                                                                                         |
|`type`                   |text           |enum: `video`, `reading_text`, `reading_media`, `matching`, `fill_blank`, `image_recognition`, `multiple_choice`, `true_false`, `timeline`. `reading_media` — иллюстрированное чтение: медиа сверху + заголовок + текст (см. 6.5)        |
|`requires_check`         |bool           |`false` → контентное задание, кнопка снизу **«Continue»**. `true` → задание с проверкой: кнопка сначала disabled с подписью-действием, после действия — активная **«Done»**, запускает проверку. Дефолт по типу, хранится явно (см. 6.5а)|
|`sort_order`             |int            |                                                                                                                                                                                                                                         |
|`payload`                |jsonb          |содержимое задания (структура зависит от type, см. 6.5)                                                                                                                                                                                  |
|`created_at / updated_at`|timestamptz    |                                                                                                                                                                                                                                         |

**`media_assets`** (видео и аудио-озвучка)

|Поле                 |Тип               |Примечание                       |
|---------------------|------------------|---------------------------------|
|`id`                 |uuid PK           |                                 |
|`task_id`            |uuid FK→tasks null|привязка к заданию               |
|`kind`               |text              |`video` / `tts_audio`            |
|`cloudflare_video_id`|text null         |для видео                        |
|`hls_url`            |text null         |                                 |
|`poster_url`         |text null         |первый кадр/обложка              |
|`audio_url`          |text null         |для TTS (Supabase Storage)       |
|`text_content`       |text null         |текст для чтения (парный к аудио)|
|`duration_seconds`   |int null          |                                 |


> **HLS на обеих платформах.** `hls_url` (Cloudflare Stream, `.m3u8`) проигрывается нативно: AVPlayer на iOS, ExoPlayer на Android — через `expo-video`. Дополнительной транскодировки под платформу не требуется.

> **Где хранится TTS-аудио чтения (источник истины, D-1).** Аудио озвучки сегментов `reading_text`/`reading_media` хранится **inline в `tasks.payload.segments[].audio_url`** (ссылка на файл в Supabase Storage) — один источник истины, рядом с текстом сегмента, к которому привязана озвучка. **`media_assets` для TTS-чтения НЕ используется:** ветка `kind='tts_audio'` (+ `audio_url`/`text_content`) **зарезервирована** и в MVP не задействована (раньше дублировала inline-аудио). `media_assets` в MVP — **только видео** (`kind='video'`: `cloudflare_video_id`/`hls_url`/`poster_url`). Воспроизведение TTS включается глобальным тумблером `app_config.economy.reading_tts_enabled` (спящая фича, §6.4/§12.5а).

### 6.3 Прогресс пользователя

**`user_course_progress`**

|Поле               |Тип             |Примечание             |
|-------------------|----------------|-----------------------|
|`user_id`          |uuid FK         |PK (user_id, course_id)|
|`course_id`        |uuid FK         |                       |
|`started_at`       |timestamptz     |                       |
|`completed_at`     |timestamptz null|                       |
|`lessons_completed`|int default 0   |для прогресс-бара      |
|`last_lesson_id`   |uuid null       |**зарезервировано** (на будущее). В MVP не используется: карточка **In Progress** ведёт на экран курса (`course_path`), а не сразу в урок (§12.3). В выдачу `get_home_feed.in_progress[]` не входит.|

**`user_lesson_progress`**

|Поле          |Тип             |Примечание                                                                           |
|--------------|----------------|-------------------------------------------------------------------------------------|
|`user_id`           |uuid FK         |PK (user_id, lesson_id)                                                              |
|`lesson_id`         |uuid FK         |                                                                                     |
|`status`            |text            |`locked` / `active` / `completed`                                                    |
|`current_task_index`|int default 0   |**позиция возобновления** внутри урока: индекс (по `sort_order`) следующего задания, которое увидит пользователь. Прогресс урока **сохраняется** — выход в середине не сбрасывает прогон; следующий вход продолжает с этого задания (§12.5в, §A7). Сбрасывается в 0 при перепрохождении уже `completed`-урока.|
|`errors_count`      |int default 0   |число неверных ответов на задания в уроке (для тона маскота, F7; на алмазы не влияет)|
|`completed_at`      |timestamptz null|                                                                                     |

**`user_task_attempts`** (история ответов)

|Поле          |Тип        |
|--------------|-----------|
|`id`          |uuid PK    |
|`user_id`     |uuid FK    |
|`task_id`     |uuid FK    |
|`is_correct`  |bool       |
|`attempted_at`|timestamptz|

### 6.4 Подписки, конфиг

> **Ачивки убраны из MVP.** Таблиц `achievements` / `user_achievements` нет. Решение по ревизии (ВАЖН-16): достижения для бизнеса сейчас не приоритетны и в продукт не входят. При возврате к фиче — заводить отдельной миграцией.

> **Квесты убраны из проекта целиком (глобальное решение).** Таблиц `quests` / `user_quests` **нет**. Квесты не несли бизнес-логики; алмазы зарабатываются за уроки и тратятся только на автозаморозку стрика (§A5). Удалены: таблицы, RPC `get_quests`/`claim_quest_reward`, блок `QUESTS`, вкладка/маршрут квестов, фича F14, фаза Phase 11. При возврате к фиче — заводить отдельной миграцией.

**`subscriptions`**

|Поле                    |Тип               |Примечание                                                                                                                              |
|------------------------|------------------|----------------------------------------------------------------------------------------------------------------------------------------|
|`user_id`               |uuid PK FK        |                                                                                                                                        |
|`status`                |text              |`trial` / `active` / `expired` / `cancelled` / `none`                                                                                   |
|`product_id`            |text              |`super_monthly`, `super_family_monthly`, …                                                                                              |
|`purchase_token`        |text null         |от Google Play / `original_transaction_id` от Apple                                                                                     |
|`platform`              |text              |**`google_play` / `app_store`** — на какой платформе оформлена подписка (RevenueCat сообщает store)                                     |
|`rc_app_user_id`        |text null         |App User ID в RevenueCat (связка с `user_id`)                                                                                           |
|`started_at`            |timestamptz null  |                                                                                                                                        |
|`expires_at`            |timestamptz null  |                                                                                                                                        |
|`validated_at`          |timestamptz null  |последняя серверная валидация                                                                                                           |
|`offer_timer_started_at`|timestamptz null  |когда у юзера стартовал персональный таймер спец-предложения (§A14); ставится сервером при первом срабатывании условия `timer_starts_on`|
|`trial_used`            |bool default false|был ли этим юзером **использован** бесплатный триал. Ставится `revenuecat-webhook` на событии `trial_started`. При `true` → `get_subscription_offer` возвращает `trial_available=false` (§A14). *(Семантика «показан» больше не используется — логику срочности покрывают `offer_timer_started_at` + `timer_expired`.)*|


> **Кросс-платформенная подписка.** Подписка `app_store` и `google_play` ведутся в одной строке на пользователя; источник статуса — RevenueCat (вебхук `revenuecat-webhook`). Если у пользователя есть активный entitlement на любой из платформ — `status` = `active`/`trial`. **Apple-специфика:** бесплатная неделя реализуется как **introductory offer** (free trial) на подписке App Store; `purchase_token` для Apple хранит `original_transaction_id`.

> **Жизненный цикл строки (C-04).** Строка `subscriptions` создаётся **при регистрации** — тем же триггером на `auth.users`, что создаёт `profiles`/`streaks` (Phase 1), со `status='none'`. Так `get_subscription_offer`, `get_me` (§8.1) и HUD всегда читают существующую строку и не разбирают `NULL`. **Защитно** на чтении трактуем «строки нет = `none`», но штатно строка есть у каждого пользователя.

**`payment_events`** (аудит платёжных событий; пишет только `revenuecat-webhook`)

|Поле         |Тип            |Примечание                                                                                          |
|-------------|---------------|----------------------------------------------------------------------------------------------------|
|`id`         |uuid PK        |                                                                                                    |
|`event_id`   |text unique    |**ID события из payload RevenueCat** — ключ идемпотентности вебхука (B-5). Вебхук пишет `on conflict (event_id) do nothing`: повторная доставка (RevenueCat = at-least-once) не плодит дубль и не применяет сайд-эффект повторно|
|`user_id`    |uuid FK→profiles **null**|связь по `rc_app_user_id` → `user_id`. **Nullable** (C-5): если резолв не удался (событие раньше связки `rc_app_user_id`↔`user_id`) — строка аудита всё равно пишется с `user_id=null`, резолв best-effort. `ON DELETE SET NULL` (аудит переживает удаление аккаунта)|
|`event_type` |text           |`initial_purchase` / `renewal` / `trial_started` / `cancellation` / `expiration` / `billing_issue`  |
|`platform`   |text           |`app_store` / `google_play` (store из события RevenueCat)                                            |
|`product_id` |text           |идентификатор продукта (`super_monthly`, …)                                                          |
|`price`      |numeric null   |сумма платежа (если есть в событии)                                                                  |
|`currency`   |text null      |валюта (ISO-4217)                                                                                    |
|`country`    |text null      |страна стора (если есть)                                                                             |
|`raw_payload`|jsonb          |полное тело события (для разбора/реконсиляции)                                                       |
|`created_at` |timestamptz    |                                                                                                    |

> **RLS `payment_events`:** запись и чтение — **только сервисная роль** (вебхук пишет, админка читает). С клиента недоступна. Используется для аналитики выручки в админке и при разборе спорных платежей. Событие `trial_started` дополнительно ставит `subscriptions.trial_used = true` (§A14).

**`app_config`** (key-value для экономики и прочих настроек админки)

|Поле   |Тип    |Примечание     |
|-------|-------|---------------|
|`key`  |text PK|напр. `economy`|
|`value`|jsonb  |см. пример ниже|

Пример `economy` value:

```json
{
  "starting_diamonds": 300,
  "battery_max": 10,
  "battery_cost_per_answer": 1,
  "battery_refill_minutes": 60,
  "battery_refill_amount": 1,
  "lesson_diamond_rewards": [
    { "amount": 10, "weight": 70 },
    { "amount": 15, "weight": 25 },
    { "amount": 20, "weight": 5 }
  ],
  "lesson_replay_diamond_reward": 1,
  "courses_per_level": 5,
  "streak_freeze_cost_per_day": 150,
  "mascot_tone_reaction_enabled": false,
  "reading_tts_enabled": false,
  "trial_offer": {
    "trial_enabled": true,
    "trial_days": 7,
    "timer_enabled": false,
    "timer_duration_hours": 24,
    "timer_starts_on": "first_subscription_view"
  }
}
```

**Пояснения к полям экономики** (полные механики — в Приложении A, не дублируем):

- `lesson_diamond_rewards` — взвешенный рандом алмазов за **первое** завершение урока (§A6).
- `lesson_replay_diamond_reward` (дефолт **1**) — фикс-награда за **перепрохождение** уже завершённого урока (взвешенный рандом не разыгрывается повторно; §A6, §12.6, ВАЖН-15).
- `courses_per_level` (дефолт **5**) — сколько пройденных курсов даёт **+1 `knowledge_level`** (§A4; HUD-уровень — §12.0).
- `streak_freeze_cost_per_day` (дефолт **150**) — сколько алмазов сервер **автоматически списывает** за каждый пропущенный день, чтобы сохранить стрик (магазина заморозок нет; §A5).
- `battery_*` — модель монетизации батареи; полная механика и восстановление — **§A4** (канон). Кратко: старт = `battery_max`; любой тест-ответ −`battery_cost_per_answer`; восстановление по `battery_refill_minutes`; подписчик не тратит. **Бонуса заряда за завершение урока нет** (награда за урок — только алмазы; §A4/§12.6).
- `mascot_tone_reaction_enabled` (дефолт `false`) — тумблер реакции маскота по `errors_count`; полное поведение — **§12.6** (канон), счётчик пишется независимо от флага.
- `reading_tts_enabled` (дефолт **`false`**, спящая фича, D-1) — глобальный тумблер **TTS-озвучки чтения** (`reading_text`/`reading_media`). Когда `false` — клиент аудио не запускает (даже если `audio_url` в payload заполнен); когда `true` — озвучивает сегменты. Включается из админки (B6.8), когда озвучка готова. Управляет **только** TTS чтения; эффект-звуки урока и `Sound Effects` (§12.5г) под него не попадают. Канон поведения — **§12.5а**.
- `trial_offer` — триал и персональный таймер; семантика полей — **§A14** (канон).

**`app_config` ключ `subscription_badge`** (ассет-замена батареи у подписчика)

|Поле        |Тип      |Примечание                                                                                                           |
|------------|---------|---------------------------------------------------------------------------------------------------------------------|
|`asset_url` |text     |URL яркого ассета, который **заменяет иконку батареи в HUD** у активного подписчика (Supabase Storage)               |
|`asset_type`|text     |`rive` / `lottie` / `svg` / `png` — формат ассета (предпочтительно анимированный Rive/Lottie, но `png` тоже допустим)|
|`label`     |text null|опц. подпись рядом (напр. название тарифа «SUPER»)                                                                   |

Пример value (`key='subscription_badge'`):

```json
{ "asset_url": "https://.../badges/super.riv", "asset_type": "rive", "label": "SUPER" }
```

> Ассет рендерится кросс-платформенно: `rive` → `rive-react-native`, `lottie` → `lottie-react-native`, `svg` → `react-native-svg`, `png` → `expo-image`. Подмена «батарея ↔ ассет подписки» в HUD — каноном описана в **§12.0** (и §10); здесь — только схема конфига. Загружается админкой (B6.8).

**`app_config` ключ `min_supported_build`** (принудительное обновление / kill switch, §A15)

```json
{ "min_supported_build": 0 }
```

> Целое число — минимальный поддерживаемый `nativeBuildVersion`. Splash после загрузки конфига сравнивает с номером сборки; если сборка меньше → блокирующий экран «Please update the app» с кнопкой в стор. Дефолт `0` (выключено). Полная логика — §A15.

**`ui_configs`** (server-driven UI; версионируется)

|Поле             |Тип        |Примечание                                  |
|-----------------|-----------|--------------------------------------------|
|`id`             |uuid PK    |                                            |
|`min_app_version`|int        |привязка к build-номеру приложения (см. A11)|
|`config`         |jsonb      |раскладка блоков (см. раздел 11)            |
|`is_active`      |bool       |                                            |
|`created_at`     |timestamptz|                                            |

**`price_tiers`**: `id`, `tier_name`, `base_price_usd numeric`, `family_multiplier numeric`.
**`country_price_map`**: `country_code text PK` (ISO-2), `tier_id uuid FK`.

> Цены в `price_tiers`/`country_price_map` — для **отображения в UI** (тексты пейволла). Реальное списание и локальная цена — из стора (App Store / Google Play) через RevenueCat `Offerings`/`Packages` (см. 1.15а, A10).

### 6.5 Структуры `tasks.payload` (jsonb) по типам

```jsonc
// type = "video"
{ "media_asset_id": "uuid", "autoplay": true }

// type = "reading_text"  (чтение текста с опц. TTS-аудио по разделам)
// requires_check = false (кнопка Continue, проверки нет — можно пропустить)
{ "title": "The Black Harvest",
  "segments": [
    { "text": "A chilling sickness creeps across the land...",
      "audio_url": "https://.../seg1.mp3" },
    { "text": "It starts as a whisper, then transforms into a roar...",
      "audio_url": null }
  ] }
// Если segments.length == 1 — текст единым куском, буллеты-индикатор не показываются.

// type = "reading_media"  (иллюстрированное чтение: одно медиа СВЕРХУ + заголовок + текст)
// requires_check = false (кнопка Continue, как у reading_text)
{ "media": {
    "kind": "image",                        // "image" (MVP) | "video" (позже)
    "image_url": "https://.../blight-map.jpg",
    "media_asset_id": null                  // если kind="video" → ссылка на media_assets
  },
  "title": "A Blight Across the Land",
  "segments": [
    { "text": "The blight does not stay in one place...",
      "audio_url": "https://.../seg1.mp3" },
    { "text": "Word spreads faster than any horse...",
      "audio_url": null }
  ] }

// type = "matching"  (сопоставление)
{ "prompt": "Match the date to the event",
  "pairs": [ {"left": "1066", "right": "Battle of Hastings"} ] }

// type = "fill_blank"
{ "text": "The war lasted ___ years.", "answer": "116",
  "options": ["100","116","120"] }

// type = "image_recognition"  («о чём эта картинка?»)
{ "image_url": "https://...", "prompt": "What does this picture show?",
  "options": ["A","B","C","D"], "answer_index": 1 }

// type = "multiple_choice"  (вопрос + 4 варианта, без картинки)
{ "prompt": "Which battle ended the Norman Conquest?",
  "options": ["Battle of Hastings","Battle of Agincourt","Battle of Bosworth","Battle of Crécy"],
  "answer_index": 0 }

// type = "true_false"  (утверждение, два варианта: True / False)
{ "statement": "Shakespeare was born in Stratford-upon-Avon.",
  "answer": true }

// type = "timeline"  (расставить события в хронологическом порядке, drag & drop)
{ "prompt": "Sort these events in chronological order",
  "items": [
    { "id": "a", "label": "Battle of Hastings",       "year": 1066 },
    { "id": "b", "label": "Magna Carta signed",        "year": 1215 },
    { "id": "c", "label": "Black Death reaches England","year": 1348 },
    { "id": "d", "label": "Battle of Agincourt",       "year": 1415 }
  ] }
// items перемешиваются на клиенте; правильный порядок — по возрастанию year.
// year хранится в payload (нужен для проверки), но НЕ показывается пользователю.
```

> **Валидация payload.** На клиенте каждый `payload` парсится **zod**-схемой по `type` перед рендером; невалидный payload → задание помечается недоступным и пропускается (не краш, см. A13).

### 6.5а Кнопка задания и проверка: `requires_check`

> Автопроверок нет: **все задания с проверкой требуют явного нажатия кнопки** для запуска проверки. У большинства типов это **`Done`**; **исключение — `true_false`**, где сама кнопка-ответ (True/False) и есть подтверждение — отдельной `Done` нет, оценка мгновенная (§12.5к, Gap #7). `requires_check` хранится **явно** в строке `tasks`, настраивается в админке; тип определяет лишь **предзаполненный дефолт**.

> **Где что описано (НЕКР-5).** Здесь, в файле модели данных, — только **семантика данных**: значение `requires_check`, дефолты по типам и таблица подписей. **Визуальное поведение кнопки** (disabled→`Done`, анимация перехода, раскладка и fade-градиент при длинном тексте) — каноном в **§12.5** (экраны).

**Поведение по `requires_check`:**

- **`requires_check = true`** → задание с проверкой. Кнопка: сначала **disabled с подписью-действием**, после действия — **активная `Done`** (исключение — `true_false`: две кнопки True/False, тап сразу запускает проверку, без `Done`). Нажатие запускает проверку. **Любой ответ → −`battery_cost_per_answer`** у не-подписчика (§A4); неверный ответ дополнительно увеличивает `errors_count` (для тона маскота). Бонуса заряда нет — награда за урок только алмазами (§A4/§12.6).
- **`requires_check = false`** → контентное задание (видео/чтение). Кнопка — **`Continue`**, батарею не тратит. Подробности по типам — §6.5б.

**Предзаполненные дефолты при создании задания в админке:**

|`type`             |`requires_check`|Подпись disabled-кнопки (пример)|Обоснование дефолта                             |
|-------------------|----------------|--------------------------------|------------------------------------------------|
|`video`            |`false`         |— (см. §6.5б, кнопка `Continue`)|Просмотр — не проверяется по смыслу             |
|`reading_text`     |`false`         |— (см. §6.5б, кнопка `Continue`)|Чтение — не проверяется                         |
|`reading_media`    |`false`         |— (см. §6.5б, кнопка `Continue`)|Иллюстрированное чтение — не проверяется        |
|`matching`         |`true`          |`Match pairs` → `Done`          |Сопоставление, подтверждение `Done`             |
|`fill_blank`       |`true`          |`Type the word` → `Done`        |Вставка слова, подтверждение `Done`             |
|`image_recognition`|`true`          |`Choose answer` → `Done`        |Выбор ответа по картинке, подтверждение `Done`  |
|`multiple_choice`  |`true`          |`Choose answer` → `Done`        |Вопрос + 4 варианта без картинки                |
|`true_false`       |`true`          |две кнопки **True / False** (без `Done`)|Утверждение; тап по True/False **сразу** запускает проверку — отдельной `Done` нет (§12.5к, Gap #7)|
|`timeline`         |`true`          |`Sort events` → `Done`          |Расставить 4–5 событий по хронологии drag & drop|

### 6.5б Кнопка `Continue` у контентных заданий — *перенесено в §12.5*

> Семантика: `reading_media`/`video` → `Continue` активна сразу; `reading_text` → `Continue` появляется только на последнем сегменте; видео — после ~95% просмотра/ручного `Continue` (§A8). **Визуальное поведение** — каноном в §12.5а/б (экраны).

### 6.5в Кнопка `Continue` не перекрывает текст — *перенесено в §12.5*

> Правило раскладки «кнопка не перекрывает текст / скролл / fade-out градиент / safe-area» — каноном в §12.5 (экраны, раскладка раннера). В файле модели данных это поведение больше не дублируется.

### 6.6 Служебные таблицы (логи)

> Таблицы наблюдаемости. Перенесены сюда из дорожной карты (НЕКР-6), чтобы вся схема жила в едином контракте данных §6. В Phase 14 — только ссылка сюда. Платёжный аудит `payment_events` — в §6.4 (рядом с подписками).

**`error_logs`** (серверные и клиентские ошибки)

|Поле        |Тип                    |Примечание                                  |
|------------|-----------------------|--------------------------------------------|
|`id`        |uuid PK                |                                            |
|`source`    |text not null          |`edge_function` / `rpc` / `client`          |
|`function`  |text not null          |название функции/экрана                     |
|`platform`  |text null              |`ios` / `android` / null (для client)       |
|`error_code`|text null              |                                            |
|`message`   |text not null          |                                            |
|`payload`   |jsonb null             |входные данные вызова (без PII)             |
|`user_id`   |uuid FK→profiles null  |`ON DELETE SET NULL` (аудит переживает удаление аккаунта)|
|`created_at`|timestamptz            |                                            |

> **RLS `error_logs`:** запись/чтение — **только сервисная роль** (Edge/RPC пишут, админка читает; с клиента недоступна).

### 6.7 Индексы и ограничения целостности

> Раньше в §6 не было ни одного индекса (B-1) и идемпотентность держалась на проверке-перед-вставкой без констрейнта (B-2). Закрыто здесь — это часть контракта данных, заводится в Phase 1 вместе со схемой. PK и `unique`-поля индексируются Postgres автоматически (`username`, `courses.slug`, `categories.slug`, `payment_events.event_id`, PK `user_*`-таблиц) — ниже только то, что **нужно завести дополнительно**.

**Ограничение идемпотентности (критично, B-2).** Начисление алмазов за урок защищается от двойного срабатывания **на уровне БД**, а не порядком выполнения:

```sql
-- одна награда на (пользователь, причина, урок); перекрывает гонку двойного complete_lesson
create unique index uq_diamond_reward_once
  on diamond_transactions (user_id, reason, ref_id)
  where reason in ('lesson_completion_reward', 'lesson_replay_reward');
```

> `complete_lesson` начисляет через `insert ... on conflict do nothing` и смотрит на число вставленных строк (§8.1). При гонке вторая транзакция получает 0 строк и **не** повторяет начисление и взвешенный розыгрыш.

**Индексы под горячие запросы RPC (§8):**

|Индекс                                                   |Зачем (RPC)                                                       |
|---------------------------------------------------------|------------------------------------------------------------------|
|`courses (is_published, sort_order)`                     |ленты `get_home_feed`/`get_grid_feed`/`get_category_courses`      |
|`courses (category_id)`                                  |`get_category_courses`, `get_catalog` (агрегат по категории)      |
|`courses (popularity_score desc)`                        |блок Most Popular (`get_home_feed.popular`)                       |
|`lessons (course_id, sort_order)`                        |сетка `get_course_path`; обложка первого урока (min `sort_order`) |
|`tasks (lesson_id, sort_order)`                          |раннер; агрегат `tasks_count`                                     |
|`user_task_attempts (user_id, task_id)`                  |`complete_lesson` (проверка/джойн по заданиям урока)              |
|`user_lesson_progress (user_id, status)`                 |выборки активных/пройденных уроков                                |
|`diamond_transactions (user_id, created_at desc)`        |история/баланс, лента транзакций                                  |
|`subscriptions (rc_app_user_id)`                         |`revenuecat-webhook`/`validate-purchase` — резолв юзера           |
|`payment_events (user_id, created_at desc)`              |аналитика выручки в админке                                       |

> Список — минимум под текущие RPC §8; расширяется по мере появления новых запросов. Точные `create index` — в миграции Phase 1.

-----

## 7. Безопасность (RLS — Row-Level Security)

> **Без изменений** (бэкенд платформо-независим). RLS включается на **всех** таблицах. Принципы:

- **Пользовательские данные** (`profiles`, `streaks`, `user_*`, `diamond_transactions`, `subscriptions`): пользователь читает/пишет **только свои строки** (`auth.uid() = user_id`).
- **Контент** (`categories`, `courses`, `lessons`, `tasks`, `media_assets`): чтение — всем авторизованным (только `is_published = true` для courses); запись — только сервисная роль.
- **Конфиги** (`app_config`, `ui_configs`, `price_tiers`, `country_price_map`): чтение — всем; запись — только сервисная роль.
- **Служебные/аудит** (`payment_events`, `error_logs`): запись и чтение — **только сервисная роль** (вебхук/Edge пишут, админка читает; с клиента недоступны).
- **Критичные мутации** (изменение `diamonds`, `battery`, `knowledge_level`, `status` подписки, `user_lesson_progress.status`/`current_task_index`) — **запрещены напрямую с клиента** даже для своих строк. Только через `SECURITY DEFINER` RPC. Подписки пишутся только серверным путём (вебхук RevenueCat / `validate-purchase`).

**Правило:** клиент может читать свой прогресс и баланс, но изменять их — только через RPC (раздел 8.1).

