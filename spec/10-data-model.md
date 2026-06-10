# 10 · Модель данных и безопасность — Lorebinge SPEC (RN9)

> Часть разбитого спека. Источник-архив: [`spec-rn9.md`](spec-rn9.md) — не редактировать. Содержит: §6 — схема БД (PostgreSQL/Supabase), §7 — RLS. Это единый контракт данных. Оглавление, граф зависимостей и карта перекрёстных ссылок — в [00-index.md](00-index.md).

-----

## 6. Модель данных (схема БД, PostgreSQL/Supabase)

> **Без изменений по сути.** Бэкенд платформо-независим. Изменения для второй платформы — только в полях, связанных с платежами (`subscriptions.platform`, `payment_events.platform`/`store`). Все таблицы — в схеме `public`, кроме `auth.users` (управляется Supabase Auth). У всех таблиц `created_at timestamptz default now()`. Где есть изменения — `updated_at` с триггером. Все id — `uuid default gen_random_uuid()`, кроме явно указанных.

### 6.1 Пользователь и геймификация

**`profiles`** (1:1 с `auth.users`)

|Поле                     |Тип            |Примечание                 |
|-------------------------|---------------|---------------------------|
|`id`                     |uuid PK        |= `auth.users.id`          |
|`username`               |text unique    |                           |
|`display_name`           |text           |                           |
|`avatar_url`             |text null      |                           |
|`knowledge_level`        |int default 1  |+1 за пройденный курс      |
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
|`freezes_available` |int default 0      |куплённые заморозки             |

**`diamond_transactions`** (леджер; идемпотентность и аудит)

|Поле        |Тип        |Примечание                                                                                                  |
|------------|-----------|------------------------------------------------------------------------------------------------------------|
|`id`        |uuid PK    |                                                                                                            |
|`user_id`   |uuid FK    |                                                                                                            |
|`amount`    |int        |знаковое (+награда / −трата)                                                                                |
|`reason`    |text       |enum-строка: `signup_bonus`, `lesson_completion_reward`, `quest_reward`, `streak_freeze`, `shop_purchase`, …|
|`ref_id`    |uuid null  |ссылка на связанную сущность                                                                                |
|`created_at`|timestamptz|                                                                                                            |

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
|`cover_image_url`        |text null              |обложка-превью курса: ячейка вкладки **Grid** (кадр/постер первого видео курса, §12.3б), карточки **In Progress** и **Category** (§12.3в)                                                                                      |
|`illustration_url`       |text null              |**главный логотип/иллюстрация карточки Lores (блок B)** — крупный по центру вверху                                                                                                                                           |
|`card_decoration_url`    |text null              |декоративная картинка внизу справа карточки Lores                                                                                                                                                                            |
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
|`cover_image_url`        |text null      |**обложка карточки урока в сетке курса** (§12.4). Генерируется конвейером (Leonardo, Приложение C) по теме урока; для уроков, созданных в админке вручную — загружается там же. При отсутствии — placeholder (`expo-image`, A13)|
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

### 6.3 Прогресс пользователя

**`user_course_progress`**

|Поле               |Тип             |Примечание             |
|-------------------|----------------|-----------------------|
|`user_id`          |uuid FK         |PK (user_id, course_id)|
|`course_id`        |uuid FK         |                       |
|`started_at`       |timestamptz     |                       |
|`completed_at`     |timestamptz null|                       |
|`lessons_completed`|int default 0   |для прогресс-бара      |
|`last_lesson_id`   |uuid null       |«продолжить с места»   |

**`user_lesson_progress`**

|Поле          |Тип             |Примечание                                                                           |
|--------------|----------------|-------------------------------------------------------------------------------------|
|`user_id`     |uuid FK         |PK (user_id, lesson_id)                                                              |
|`lesson_id`   |uuid FK         |                                                                                     |
|`status`      |text            |`locked` / `active` / `completed`                                                    |
|`errors_count`|int default 0   |число неверных ответов на задания в уроке (для тона маскота, F7; на алмазы не влияет)|
|`completed_at`|timestamptz null|                                                                                     |

**`user_task_attempts`** (история ответов)

|Поле          |Тип        |
|--------------|-----------|
|`id`          |uuid PK    |
|`user_id`     |uuid FK    |
|`task_id`     |uuid FK    |
|`is_correct`  |bool       |
|`attempted_at`|timestamptz|

### 6.4 Ачивки, подписки, конфиг

**`achievements`**

|Поле         |Тип    |Примечание                   |
|-------------|-------|-----------------------------|
|`id`         |uuid PK|                             |
|`title`      |text   |                             |
|`description`|text   |                             |
|`icon_url`   |text   |                             |
|`type`       |text   |`costume` / `medal` / `badge`|
|`criteria`   |jsonb  |условие выдачи               |

**`user_achievements`**: `(user_id, achievement_id)` PK, `earned_at`.

**`quests`** (задания блока F)

|Поле             |Тип              |Примечание                                             |
|-----------------|-----------------|-------------------------------------------------------|
|`id`             |uuid PK          |                                                       |
|`title`          |text             |                                                       |
|`description`    |text null        |                                                       |
|`period`         |text             |`daily` / `monthly` / `exclusive`                      |
|`criteria`       |jsonb            |условие выполнения (напр. `{ "lessons_completed": 3 }`)|
|`reward_diamonds`|int              |награда за выполнение                                  |
|`is_active`      |bool default true|                                                       |
|`starts_at`      |timestamptz null |окно активности                                        |
|`ends_at`        |timestamptz null |                                                       |

**`user_quests`** (прогресс/получение награды)

|Поле          |Тип             |Примечание                             |
|--------------|----------------|---------------------------------------|
|`user_id`     |uuid FK         |PK (user_id, quest_id)                 |
|`quest_id`    |uuid FK→quests  |                                       |
|`progress`    |jsonb           |текущий прогресс по `criteria`         |
|`completed_at`|timestamptz null|                                       |
|`claimed_at`  |timestamptz null|когда награда забрана (идемпотентность)|

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
|`trial_offered`         |bool default false|был ли этому юзеру когда-либо показан бесплатный триал (для логики «после истечения таймера триал больше не предлагается», §A14)        |


> **Кросс-платформенная подписка.** Подписка `app_store` и `google_play` ведутся в одной строке на пользователя; источник статуса — RevenueCat (вебхук `revenuecat-webhook`). Если у пользователя есть активный entitlement на любой из платформ — `status` = `active`/`trial`. **Apple-специфика:** бесплатная неделя реализуется как **introductory offer** (free trial) на подписке App Store; `purchase_token` для Apple хранит `original_transaction_id`.

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
  "battery_reward_per_lesson": 3,
  "battery_refill_minutes": 60,
  "battery_refill_amount": 1,
  "lesson_diamond_rewards": [
    { "amount": 10, "weight": 70 },
    { "amount": 15, "weight": 25 },
    { "amount": 20, "weight": 5 }
  ],
  "streak_freeze_cost_per_day": 150,
  "mascot_tone_reaction_enabled": false,
  "trial_offer": {
    "trial_enabled": true,
    "trial_days": 7,
    "timer_enabled": false,
    "timer_duration_hours": 24,
    "timer_starts_on": "first_subscription_view"
  }
}
```

> **Триал и таймер — две независимые настройки** (§A14). `trial_enabled` управляет тем, предлагается ли бесплатная неделя вообще. `timer_enabled` управляет тем, ограничено ли предложение персональным окном с обратным отсчётом. Их можно включать/выключать по отдельности.

> **`mascot_tone_reaction_enabled`** (дефолт `false`) — тумблер реакции маскота на качество заданий (пулы celebrate/encourage по `errors_count`, F7/§12.6). Управляется из админки (B6.8), читается клиентом на старте. **Включается только в связке с готовыми ассетами:** анимации обоих пулов — бандл-ассеты приложения, поэтому при `true` реакция появится лишь в той сборке, где эти анимации уже есть; если флаг `true`, а пулы в установленной версии отсутствуют — клиент **мягко откатывается на нейтральный пул** (без краша). При `false` всегда играет нейтральный пул. Счётчик `errors_count` пишется независимо от флага.

> **Батарея (модель монетизации).** Стартовый заряд = `battery_max` = **10** (хватает физически на 10 тест-ответов = один урок). **Каждый ответ на задание-тест** (`requires_check=true`) тратит `battery_cost_per_answer` = 1 заряд — **и верный, и неверный** (неважно какой). **За завершение урока** начисляется фикс `battery_reward_per_lesson` = **+3** заряда (не выше `battery_max`) — отличник получает запас ещё на ~3 тест-задания, но не на целый второй урок. Задания без проверки (`video`, `reading_text`, `reading_media` → Continue) батарею **не тратят**. Восстановление со временем: +`battery_refill_amount` каждые `battery_refill_minutes` (дефолт +1/час → полные 10 за 10 ч, ≈24/сутки ≈ 2 урока в день), не выше `battery_max`. Подписчик батарею не тратит. Все числа настраиваются в админке (§A4, B6.8).

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

> Ассет рендерится кросс-платформенно: `rive` → `rive-react-native`, `lottie` → `lottie-react-native`, `svg` → `react-native-svg`, `png` → `expo-image`. Когда подписка активна — в HUD на месте батареи показывается этот ассет; когда подписка отменена/истекла — батарея возвращается (см. §A4, §12.8). Загружается админкой (B6.8).

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

> Автопроверок нет: **все задания с проверкой требуют явного подтверждения** нажатием кнопки. `requires_check` хранится **явно** в строке `tasks`, настраивается в админке; тип определяет лишь **предзаполненный дефолт**.

**Единая нижняя кнопка задания (овальная, на каждом задании).** Два визуальных состояния:

1. **Неактивная (disabled, приглушённая)** — пока задание **не выполнено**. На кнопке — **краткая подпись-действие в 2–3 слова на английском**: `Connect objects`, `Type the word`, `Choose answer`, `Match pairs` и т.п. Подпись задаётся per-type (через i18n).
1. **Активная (enabled, яркая/выпуклая)** — как только пользователь выполнил действие. Подпись меняется на **`Done`**. Переход состояния заметный (цвет/объём, через Reanimated). Нажатие → проверка → списание батареи (любой ответ, см. §A4) → переход к следующему заданию.

**Поведение по `requires_check`:**

- **`requires_check = true`** → задание с проверкой. Кнопка: сначала **disabled с подписью-действием**, после действия — **активная `Done`**. Нажатие `Done` запускает проверку. **Любой ответ → −`battery_cost_per_answer`** у не-подписчика (§A4); неверный ответ дополнительно увеличивает `errors_count` (для тона маскота). Бонус заряда даётся не здесь, а за завершение урока (§A4).
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
|`true_false`       |`true`          |`True or false?` → `Done`       |Утверждение, два варианта: True / False         |
|`timeline`         |`true`          |`Sort events` → `Done`          |Расставить 4–5 событий по хронологии drag & drop|

### 6.5б Кнопка `Continue` у контентных заданий (`requires_check=false`)

**(1) `reading_media` — текст + медиа (картинка ИЛИ видео сверху).** Картинка с текстом и видео с текстом — **один тип** (`reading_media`, отличается только `media.kind`). Кнопка **`Continue` всегда активна сразу**.

**(2) `reading_text` — чисто текстовый блок, поделённый на сегменты, свайп снизу вверх.** Кнопка **`Continue` появляется НЕ сразу**. Изначально кнопки нет — виден только первый блок (акцентный белый), остальные ниже. Свайп снизу вверх (Gesture Handler + Reanimated) открывает следующий сегмент. **`Continue` появляется внизу только когда долистали до последнего сегмента.** Если сегмент один — `Continue` доступна сразу.

> **Видео-задание (`type=video`)** — отдельный полноэкранный тип; его кнопка `Continue` и условие «просмотрено» описаны в §12.5 / §A8 (≈95% длительности или ручной `Continue`).

### 6.5в Кнопка `Continue` не перекрывает текст (правило раскладки)

- Кнопка `Continue`/`Done` **никогда не перекрывает читаемый текст**. Если текст помещается — он целиком над кнопкой.
- Если текст **длинный** — текстовая область становится **скроллируемой** (`ScrollView`, индикатор справа). Кнопка остаётся зафиксированной снизу (абсолютное позиционирование над safe-area), текст скроллится **под** ней.
- **Градиент-исчезание у кнопки:** по мере приближения к зафиксированной кнопке текст **плавно уходит в прозрачность (fade-out градиент)** — через `expo-linear-gradient` поверх нижней кромки скролла, а не резкий обрыв.
- Автор контента старается держать тексты короткими, но при длинном тексте поведение выше обязательно. Учитывать safe-area (нижний инсет) на обеих платформах.

-----

## 7. Безопасность (RLS — Row-Level Security)

> **Без изменений** (бэкенд платформо-независим). RLS включается на **всех** таблицах. Принципы:

- **Пользовательские данные** (`profiles`, `streaks`, `user_*`, `user_quests`, `diamond_transactions`, `subscriptions`, `user_achievements`): пользователь читает/пишет **только свои строки** (`auth.uid() = user_id`).
- **Контент** (`categories`, `courses`, `lessons`, `tasks`, `media_assets`, `achievements`, `quests`): чтение — всем авторизованным (только `is_published = true` для courses; только активные для `quests`); запись — только сервисная роль.
- **Конфиги** (`app_config`, `ui_configs`, `price_tiers`, `country_price_map`): чтение — всем; запись — только сервисная роль.
- **Критичные мутации** (изменение `diamonds`, `battery`, `status` подписки, `user_lesson_progress.status`, `user_quests.claimed_at`) — **запрещены напрямую с клиента** даже для своих строк. Только через `SECURITY DEFINER` RPC. Подписки пишутся только серверным путём (вебхук RevenueCat / `validate-purchase`).

**Правило:** клиент может читать свой прогресс и баланс, но изменять их — только через RPC (раздел 8.1).

