# 34 · Дорожная карта приложения — Lorebinge SPEC (RN9)

> Часть разбитого спека. Источник-архив: [`spec-rn9.md`](spec-rn9.md) — не редактировать. Содержит: §14 — фазы разработки приложения (Phase 0–16). Оглавление, граф зависимостей и карта перекрёстных ссылок — в [00-index.md](00-index.md).

-----

## 14. Дорожная карта разработки (фазы)

> Строгий порядок. Каждая фаза завершается рабочим, проверяемым результатом **на обеих платформах (iOS + Android)**. Не начинать следующую без выполнения AC текущей. **Тестовые задачи в этот документ не входят** — см. отдельный тест-план.

> **Три дорожные карты.** Ниже — фазы приложения (A). Админка (B) — фазы B-Phase 1…7 (Приложение B). Конвейер (C) — фазы C-Phase 1…6 (Приложение C). Общая зависимость для B и C — **Phase 1 (схема БД)**.

### Phase 0 — Каркас проекта (iOS + Android)

**Цель:** пустое приложение запускается на обеих платформах, тема настроена.
**Задачи:** создать монорепо; Expo-проект (TypeScript, New Architecture); подключить зависимости из раздела 2; настроить `app.config.ts` (bundleId/applicationId `com.lorebinge.app`, scheme `lorebinge`, иконки/splash, associated domains/app links заготовки); `eas.json` (профили dev/preview/prod для iOS и Android); prebuild (`ios/`, `android/`); завести Supabase-проект (dev); реализовать дизайн-токены (раздел 5) + `ThemeProvider`/`useTheme`; `App.tsx` с провайдерами (SafeArea, Theme, QueryClient) + RootNavigator со `splash`.
**Deliverables:** собираемый dev-client под **iOS-симулятор и Android-эмулятор** через EAS; тема применяется (light/dark).
**AC:** приложение запускается на iOS-симуляторе **и** Android-эмуляторе с брендовой темой; навигация-каркас работает.

### Phase 1 — База данных и RLS

**Цель:** полная схема БД с защитой.
**Задачи:** SQL-миграции для всех таблиц (раздел 6, поле `subscriptions.platform`+`trial_used`, `payment_events` §6.4, служебные таблицы `error_logs` §6.6, `user_lesson_progress.current_task_index` для возобновления урока §6.3, **`payment_events.event_id` (unique, идемпотентность вебхука) + `user_id` nullable** §6.4); **индексы и ограничения целостности §6.7** (индексы под горячие пути RPC §8; `ON DELETE` для **всех** FK — пользовательские/контент `CASCADE` (вкл. `profiles.id → auth.users ON DELETE CASCADE` для `delete-account`), аудит `payment_events`/`error_logs` `SET NULL`; partial-unique `uq_diamond_reward_once` — идемпотентность алмазов на уровне БД); **без квестов** (`quests`/`user_quests` не заводим — убраны целиком); **без ачивок** (`achievements`/`user_achievements` не заводим — ВАЖН-16); `streaks` **без** `freezes_available` (заморозка автоматическая за алмазы); `profiles` **без** `avatar_url`, с `courses_completed`; **миграции схемы `pipeline`** (Приложение C, C3) с RLS «только сервисная роль»; включить RLS и политики (раздел 7); `seed.sql` (категории, `app_config.economy` с `lesson_replay_diamond_reward`/`courses_per_level`/`reading_tts_enabled:false` (спящая TTS-озвучка, D-1) — **без** `battery_reward_per_lesson`, `app_config.min_supported_build`, дефолтный `ui_configs` с **5 вкладками** (Catalog — `enabled:false`, спящая, §12.3г) — **блок `CATEGORIES` с Home убран**, добавлен блок `CATALOG` в спящую вкладку Catalog; **без блоков SHOP и QUESTS**, `price_tiers`/`country_price_map`); триггеры `updated_at` и создание `profiles`/`streaks`/`subscriptions`(`status='none'`, C-4) при регистрации.
**Deliverables:** применённые миграции в Supabase dev.
**AC:** таблицы существуют; клиент не может изменить чужие/критичные поля; сид-данные на месте.

### Phase 2 — Авторизация (F1)

**Цель:** вход/регистрация и сессии на обеих платформах.
**Задачи:** Supabase Auth; **Google Sign-In** (обе платформы) + **Sign in with Apple** (iOS, `expo-apple-authentication` → `signInWithIdToken`); email-регистрация с кодом; экраны `auth/*`; сохранение сессии в secure-store; авто-создание профиля (300 алмазов); splash-роутинг по сессии.
**AC:** см. F1; вход работает на iOS и Android; splash ведёт корректно.

### Phase 3 — Data-слой и базовые репозитории

**Цель:** доступ к данным.
**Задачи:** supabase-js клиент (+ url-polyfill); zod-DTO + мапперы в доменные модели (раздел 9); репозитории (Profile, Course, Progress, Config, Economy) на TanStack Query; SQLite-кэш для курсов; MMKV для настроек; secure-store для сессии; `AppResult`-обёртка и маппинг ошибок RPC → `ErrorType`.
**AC:** репозитории возвращают доменные модели; конфиг/economy читаются; кэш работает; ошибки маппятся.

### Phase 4 — Home + server-driven UI (F9)

**Цель:** рабочий главный экран с лентой блоков.
**Задачи:** рендерер блоков (раздел 11); встроенный `default_ui_config.json`; блоки Home (Lores, In Progress, Popular — **без Categories**) + **вкладка Grid** (инстаграм-сетка обложек, §12.3б) + **спящая вкладка Catalog** (блок `CATALOG` — алфавитный `SectionList`-индекс категорий, §12.3г; `enabled:false` по умолчанию, включается из админки B6.7) + экран `category` (§12.3в, в стеке Catalog) как независимые компоненты + их хуки; `get_me` (снимок пользователя для холодного старта: профиль+стрик+подписка+бейдж одним вызовом, §8.1/C-6; из него собирается `UserProfile`/HUD — `isSubscriber`/`subscriptionBadge` из `subscriptions`+`subscription_badge`, A-3), `get_home_feed` (без `categories`), `get_grid_feed`, `get_catalog`, `get_category_courses`; нижняя навигация (**5 вкладок: Home · Catalog(спящая) · Grid · Subscription · Profile**; пока Catalog выключена — видны 4) с сохранением состояния (вложенные стеки); **выбор стартовой вкладки по пройденным курсам** (`courses_completed=0` → Grid, иначе Home; Gap #1); **диплинк в стек вкладки Grid, «Назад» → `grid/feed`** (Gap #5, §12.1/§10); **HUD + таб-бар как постоянный каркас** (`course_path`/`category` пушатся внутри стеков вкладок → каркас виден; скрыт только на корневых lesson/reward/paywall; интерактивность HUD — дропдаун level и слот заряда: не-подписчик → подписки, подписчик → «You're already subscribed» + тариф, Gap #6 — и окно level-up — §12.0); **force-update на Splash** (`app_config.min_supported_build` vs `nativeBuildVersion` → блокирующий экран обновления, §A15). Вкладка Profile — только блок Profile (квесты убраны).
**AC:** см. F9; Home (Lores/In Progress/Popular — без Categories) и вкладка Grid отображаются на обеих платформах; **вкладка Catalog по умолчанию скрыта** (`enabled:false`); при включении в конфиге — Catalog показывает алфавитный индекс категорий, тап по категории → её курсы → курс; тап в Grid-ячейке ведёт в курс; навигация сохраняет состояние; HUD и таб-бар видны на всех вкладках и на `course_path`/`category`, скрыты на lesson/reward/paywall.

### Phase 5 — Сетка уроков курса (F5 частично)

**Цель:** экран курса — сетка карточек-уроков с обложками (§12.4).
**Задачи:** `course_path/{courseId}`; `get_course_path` (возвращает `cover_image_url`, `status`, `path_background_color`, `course.is_completed`); рендер **вертикальных ~3:4** карточек (скругление `radiusLarge`) по `sort_order` в сетке на фоне `path_background_color`; обложка урока (`expo-image`, placeholder); три состояния карточки — `completed` (полный цвет, без бейджей), `active` (полный цвет + **статичная акцентная рамка/кольцо `node-active`, без пульса**, локально по контуру — не перекрывает соседей), `locked` (desaturate-overlay + иконка `Lock` прямо на слое, неинтерактивна); токены `node-*` (§5.6); **верхняя плашка «Course completed» при `course.is_completed`** (значок `success`, D); переход в урок с `active`/`completed`.
**AC:** сетка рисуется; статусы корректны; `locked` неинтерактивна и видна с замком; активная карточка единственная имеет статичную акцентную рамку (без пульсации), не залезающую на соседние карточки; пройденный курс показывает верхнюю плашку «Course completed»; обложки грузятся с placeholder без краша.

### Phase 6 — Раннер уроков и задания (F2 частично)

**Цель:** прохождение урока.
**Задачи:** `lesson/{lessonId}` (нижний HUD скрыт, `fullScreenModal`); **`start_lesson` при входе** (возврат `resume_task_index` — открытие урока с сохранённой позиции; ленивая инициализация прогресса; `errors_count` сбрасывается только при свежем проходе/перепрохождении, §A7); **верхняя панель урока** (прогресс-бар; скрытие на полноэкранном видео; крестик `✕` → bottom sheet `Exit lesson?` / `Exit lesson` (**прогресс сохраняется**, не сброс) / `Cancel`, см. 12.5в); реализовать все типы заданий (раздел 6.5): video (`expo-video` — базово), **reading_text** (заголовок + буллеты-индикатор + посегментный текст + свайп-вверх + посегментное TTS-аудио `expo-audio` — **за флагом `reading_tts_enabled`, спящая, дефолт off**, D-1; см. 12.5а), **reading_media** (то же + медиа сверху, см. 12.5б), **matching** (тап-выбор столбцов с цветами success/error, §12.5д), **fill_blank** (кирпичики-слова: drag/тап в пропуск, след-слот, §12.5е), **image_recognition** (картинка + варианты, токен `selected`, §12.5з), **multiple_choice** (вопрос + варианты, токен `selected`, §12.5и), **true_false** (две кнопки True/False, **мгновенная оценка без `Done`**, §12.5к), **timeline** (drag&drop карточек-событий; при ошибке — автоперестановка в правильный порядок + подсветка success/error по местам, §12.5ж); **нижняя кнопка по `requires_check` (Continue vs disabled-действие→Done; исключение — true_false; раскладка не-перекрытия текста — §12.5)**; **вердикт после проверки** (подсветка success/error + баннер + правильный ответ + хаптика/звук, §12.5г); `submit_task_attempt` (продвигает `current_task_index`); переход между заданиями.
**AC:** все типы заданий проходятся на обеих платформах; контентные показывают Continue (reading_text — на последнем сегменте, reading_media — сразу) и не тратят батарею; задания с проверкой показывают disabled-действие → активную Done и вердикт верно/неверно (**true_false — мгновенно по тапу, без Done**; timeline при ошибке автоматически переставляет в верный порядок с цветовой пометкой); текст не перекрывается кнопкой; reading_text переключает разделы свайпом с корректным аудио; **выход из урока сохраняет прогресс — повторный заход продолжает с того же задания**.

### Phase 7 — Видео-плеер с управлением (F6)

**Цель:** «мгновенное» видео + тап-управление.
**Задачи:** `expo-video` + HLS из Cloudflare Stream; кэш/прелоад следующего; постер; адаптивное качество. **Общий компонент плеера** (`features/player`): тап по центру = пауза/play (иконка ▶️ при паузе), перемотка перетаскиванием по прогресс-полосе (Gesture Handler), (опц.) двойной тап ±10 сек. Интегрировать в video-задание раннера (Phase 6).
**AC:** см. F6; одинаковое поведение на iOS (AVPlayer) и Android (ExoPlayer).

### Phase 8 — Экономика на сервере (F2, F3, F4, F5)

**Цель:** батарейка/алмазы/стрик/разблокировка — server-authoritative.
**Задачи:** RPC `start_lesson` (возобновление урока: `resume_task_index` + ленивая инициализация прогресса, §A7), `complete_lesson`, `touch_streak` (внутренняя), `refresh_user_state` (батарея + ленивая переоценка стрика + **возврат батареи при истечении подписки**, C-1; **заменяет `refill_battery`**), `reset_course_progress`; **`buy_streak_freeze`/`apply_streak_freeze` НЕ реализуем — магазина и ручной заморозки нет (КРИТ-8); `get_quests`/`claim_quest_reward` НЕ реализуем — квесты убраны целиком**; **автозаморозка стрика за алмазы** (`streak_freeze_cost_per_day`/пропущенный день) внутри `touch_streak`/`refresh_user_state`; **`submit_task_attempt` тратит `battery_cost_per_answer` за любой ответ и продвигает `current_task_index`; `complete_lesson` завершает по курсору `current_task_index` (дошёл до последнего задания, **не** по наличию attempts — C-2) и начисляет ТОЛЬКО алмазы (первое прохождение — взвешенный рандом, идемпотентно через unique-индекс `uq_diamond_reward_once` + `on conflict do nothing`, B-2; перепрохождение — фикс `lesson_replay_diamond_reward`) — бонуса заряда за урок нет (Gap #6); `courses_completed`/`knowledge_level` обновляются там же; **все мутации `battery`/`diamonds`/`current_task_index` — под `FOR UPDATE` строки юзера (B-4)**; восстановление по `battery_refill_minutes`; **HUD: батарея ↔ ассет подписки** (`app_config.subscription_badge`), возврат батареи при отмене; идемпотентность наград; интеграция в раннер и Home/HUD.
**AC:** см. F2–F5; награды не дублируются; разблокировка только через сервер; стрик пересчитывается при заходе (`refresh_user_state`).

### Phase 9 — Маскот и награды (F7)

**Цель:** эмоциональная петля.
**Задачи:** интеграция `rive-react-native`; экран `reward/{lessonId}`; **нейтральный пул анимаций маскота** (случайный выбор) как базовое поведение; текст похвалы; показ начисленных алмазов; splash-анимация Rive. **Реакция по качеству — за флагом `app_config.economy.mascot_tone_reaction_enabled` (дефолт `false`):** клиент читает флаг на старте; при `true` и наличии анимаций обоих пулов выбирает тон по `errors_count` (celebrate/encourage), иначе — нейтральный пул (мягкий откат, без краша). Сами анимации пулов (10 + 10) добавляются в сборку, когда нарисованы; до этого флаг держим выключенным. `errors_count` уже считается на сервере — бэкенд для включения менять не нужно.
**AC:** см. F7 (анимации играют на обеих платформах).

### Phase 10 — Подписки и пейволл (F8, F10)

**Цель:** монетизация на обеих платформах.
**Задачи:** **RevenueCat** (`react-native-purchases`): конфигурация Offerings/Packages для App Store + Google Play; entitlement `super`; **вкладка Subscription** (Super / Family — самостоятельная, со всеми планами/ценами/текущим планом/триалом, §12.8); **paywall-оверлей** (триггер — **только** `battery=0` во время задания; та же раскладка, что у Subscription; закрытие без покупки → возврат в `course_path` с сохранённым прогрессом урока; покупка → продолжение урока с того же задания; Gap #2/#3/#4, §12.9); Edge `revenuecat-webhook` (обновление `subscriptions` + `payment_events`) и `validate-purchase` (реконсиляция); Restore Purchases; отображение цен из стора (RevenueCat) с фолбэком `price_tiers`. **Предложение подписки (§A14):** RPC `get_subscription_offer`; условный CTA (триал «Try 1 week for $0» — Android free-trial / iOS introductory offer — vs обычная покупка по `trial_available`); **персональный таймер** на Subscription и Paywall (тикает до `timer_ends_at`, исчезает при истечении → триал больше не предлагается); запись `trial_started` в `payment_events`.
**AC:** см. F8, F10; покупка/восстановление работают в обоих сторах; триал и таймер управляются независимо из админки; статус приходит вебхуком и виден в приложении.

### Phase 11 — *удалена (квесты убраны из проекта целиком)*

> Фаза вырезана: квесты (фича F14) убраны полностью — нет таблиц `quests`/`user_quests`, RPC `get_quests`/`claim_quest_reward`, блока `QUESTS`, вкладки/маршрута. Алмазы зарабатываются за уроки и тратятся только на автозаморозку стрика (Phase 8). Номер фазы сохранён, чтобы не сбивать ссылки. При возврате к фиче — заводить отдельной фазой.

### Phase 12 — Профиль и настройки (F11)

**Цель:** ЛК и все настройки.
**Задачи:** Profile (только инфо профиля; **без квестов** — убраны целиком; **без ачивок** — убраны, ВАЖН-16; **без дубля HUD** — level/streak/diamonds/battery даёт общий HUD-каркас, §12.0; интерактивность HUD — дропдаун level и дропдаун слота заряда: не-подписчик → подписки, подписчик → «You're already subscribed» + тариф (Gap #6); окно level-up); Settings и подразделы (12.12): Preferences (MMKV + haptics; Sound Effects не глушит TTS), Profile-edit (Password только для email-аккаунтов), Course-management, **Account Linking через Supabase `linkIdentity`/`unlinkIdentity`** (§A16), Documents (in-app), Support (Restore Subscription, управление подпиской в сторе, LOG OUT); удаление аккаунта через **Edge `delete-account`** (сервисная роль → `auth.admin.deleteUser` + каскад `ON DELETE`, §8.2/§6, C-3).
**AC:** см. F11.

### Phase 13 — Виджеты и локальные уведомления (F12)

**Цель:** напоминания и виджеты на обеих платформах.
**Задачи:** **Android Glance** + **iOS WidgetKit** (≥2 размера каждый) через config-plugin/нативные таргеты; единый мост данных (App Group на iOS, shared storage на Android); локальные ежедневные напоминания (`expo-notifications`, тумблер в Preferences; запрос разрешений на обеих платформах); **deep-link по тапу → Home (`lorebinge://`) в MVP** (НЕКР-16; вариант «последний курс по slug» — пост-MVP).
**AC:** см. F12; виджеты обновляются и открывают приложение (Home) на iOS и Android.

### Phase 14 — Аналитика, краши и логирование (F13)

**Цель:** наблюдаемость продакшена.
**Задачи:**

- `@react-native-firebase/analytics` — ключевые события (обе платформы; `google-services.json` + `GoogleService-Info.plist`).
- `@react-native-firebase/crashlytics` — неперехваченные краши.
- Клиентское логирование ошибок: пойманные исключения (репозитории, хуки, покупки) → `crashlytics().recordError(e)`. Dev-логи — только в `__DEV__`.
- Серверное логирование: таблица **`error_logs` (схема — в §6.6**, перенесена в модель данных, НЕКР-6); каждая Edge Function/RPC при пойманной ошибке пишет запись.

**События аналитики MVP (НЕКР-14)** — покрывают воронку «установка → урок → пейволл → подписка» и экономику:

|Событие                |Ключевые параметры                       |
|-----------------------|-----------------------------------------|
|`sign_up`              |method (google/apple/email)              |
|`lesson_started`       |course_id, lesson_id                     |
|`task_answered`        |task_id, type, is_correct                |
|`lesson_completed`     |course_id, lesson_id, errors_count       |
|`course_completed`     |course_id, new_knowledge_level           |
|`battery_empty`        |course_id, lesson_id                     |
|`paywall_shown`        |source (battery_empty — единственный триггер)|
|`paywall_dismissed`    |source                                   |
|`trial_started`        |product_id, platform                     |
|`subscription_started` |product_id, platform                     |
|`streak_lost`          |previous_streak                          |
|`level_up`             |new_knowledge_level                      |
|`widget_opened`        |size                                     |

> Параметры обязательно без PII. Список — рабочий минимум; расширяется по продуктовым запросам. (`trial_started` также пишется в `payment_events` вебхуком — Phase 10.)

**AC:** см. F13; перечисленные события уходят в Firebase Analytics на обеих платформах.

### Phase 15 — Админ-панель (выделена в отдельный подпроект)

Админка — самостоятельный подпроект со своими фазами, стеком и дизайн-системой. **Полное описание — в Приложении B.** Её фазы (**B-Phase 1–7, Приложение B**) идут после релиза MVP-приложения, но БД-фундамент (аналитические view) закладывается раньше.

### Phase 16 — Подготовка к релизу (App Store + Google Play)

**Цель:** публикация в **обоих сторах**.
**Задачи:** dev/prod окружения (два проекта Supabase); EAS Submit в App Store Connect и Google Play Console; **Privacy Policy + App Privacy (Apple) + Data Safety (Google)**; **Privacy Manifests (iOS): проверить агрегированный `PrivacyInfo.xcprivacy`** (Expo генерирует из манифестов SDK), задекларировать required-reason API, согласовать с App Privacy labels (НЕКР-13); иконки/стор-листинги для обеих платформ; диплинки/Universal Links/App Links (проверка AASA + assetlinks.json); проверка DELETE ACCOUNT (требование обоих сторов); проверка Restore Purchases; тест на реальных устройствах iOS и Android; настройка RevenueCat prod (App Store / Play credentials); App Store review checklist (Sign in with Apple, account deletion, subscription disclosure).
**AC:** release-сборки подключаются к prod; обе платформы проходят ревью сторов; юридические требования Apple и Google выполнены.

> **Вне MVP (после релиза):** FCM/APNs push, Cloudflare R2, расширенный server-side billing, офлайн-режим. **iOS теперь в MVP** (целевая платформа наравне с Android). **Квесты (блок F / Phase 11) убраны из проекта целиком** — не входят в MVP.

