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
**Задачи:** SQL-миграции для всех таблиц (раздел 6, включая `quests`/`user_quests`, поле `subscriptions.platform`, `payment_events` с `platform`); **миграции схемы `pipeline`** (Приложение C, C3) с RLS «только сервисная роль»; включить RLS и политики (раздел 7); `seed.sql` (категории, `app_config.economy`, дефолтный `ui_configs` с 4 вкладками, `price_tiers`/`country_price_map`, демо-квесты); триггеры `updated_at` и создание `profiles`/`streaks` при регистрации.
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
**Задачи:** рендерер блоков (раздел 11); встроенный `default_ui_config.json`; блоки A–E (Grid, Lores, In Progress, Popular, Categories) как независимые компоненты + их хуки; `get_home_feed`; нижняя навигация (4 вкладки) с сохранением состояния (вложенные стеки); HUD-компонент. Вкладка Quests — заглушка (наполняется в Phase 11).
**AC:** см. F9; Home отображает блоки на обеих платформах; навигация сохраняет состояние; HUD виден на Home.

### Phase 5 — Сетка уроков курса (F5 частично)

**Цель:** экран курса — сетка карточек-уроков с обложками (§12.4).
**Задачи:** `course_path/{courseId}`; `get_course_path` (возвращает `cover_image_url`, `status`, `path_background_color`); рендер карточек по `sort_order` в сетке на фоне `path_background_color`; обложка урока (`expo-image`, placeholder); три состояния карточки — `completed` (полный цвет, без бейджей), `active` (полный цвет + пульс Reanimated, фолбэк-рамка при reduce-motion), `locked` (desaturate-overlay + иконка `Lock` прямо на слое, неинтерактивна); токены `node-*` (§5.6); переход в урок с `active`/`completed`.
**AC:** сетка рисуется; статусы корректны; `locked` неинтерактивна и видна с замком; активная карточка единственная пульсирует (или имеет статичную рамку при reduce-motion); обложки грузятся с placeholder без краша.

### Phase 6 — Раннер уроков и задания (F2 частично)

**Цель:** прохождение урока.
**Задачи:** `lesson/{lessonId}` (нижний HUD скрыт, `fullScreenModal`); **верхняя панель урока** (прогресс-бар; скрытие на полноэкранном видео; крестик `✕` → bottom sheet `Are you sure?` / `Exit lesson` (сброс) / `Cancel`, см. 12.5в); реализовать все типы заданий (раздел 6.5): video (`expo-video` — базово), **reading_text** (заголовок + буллеты-индикатор + посегментный текст с серением прошлых разделов + свайп-вверх через Gesture Handler/Reanimated + посегментное TTS-аудио со старт/стопом, см. 12.5а), **reading_media** (то же + медиа-картинка сверху, см. 12.5б), matching, fill_blank, image_recognition, multiple_choice, true_false, timeline (drag&drop через Gesture Handler); **нижняя кнопка по `requires_check` (Continue vs disabled-действие→Done, см. 6.5а/6.5б; правило не-перекрытия текста §6.5в)**; `submit_task_attempt`; переход между заданиями.
**AC:** все типы заданий проходятся на обеих платформах; контентные показывают Continue (reading_text — на последнем сегменте, reading_media — сразу) и не тратят батарею; задания с проверкой показывают disabled-действие → активную Done; текст не перекрывается кнопкой; reading_text переключает разделы свайпом с корректным аудио.

### Phase 7 — Видео-плеер с управлением (F6)

**Цель:** «мгновенное» видео + тап-управление.
**Задачи:** `expo-video` + HLS из Cloudflare Stream; кэш/прелоад следующего; постер; адаптивное качество. **Общий компонент плеера** (`features/player`): тап по центру = пауза/play (иконка ▶️ при паузе), перемотка перетаскиванием по прогресс-полосе (Gesture Handler), (опц.) двойной тап ±10 сек. Интегрировать в video-задание раннера (Phase 6).
**AC:** см. F6; одинаковое поведение на iOS (AVPlayer) и Android (ExoPlayer).

### Phase 8 — Экономика на сервере (F2, F3, F4, F5)

**Цель:** батарейка/алмазы/стрик/разблокировка — server-authoritative.
**Задачи:** RPC `complete_lesson`, `touch_streak`, `refill_battery`, `buy_streak_freeze`, `apply_streak_freeze`, `reset_course_progress`; **`submit_task_attempt` тратит `battery_cost_per_answer` за любой ответ; `complete_lesson` начисляет фикс `battery_reward_per_lesson` заряда за урок (cap `battery_max`, не подписчику, идемпотентно)**; восстановление по `battery_refill_minutes`; **HUD: батарея ↔ ассет подписки** (`app_config.subscription_badge`), возврат батареи при отмене; идемпотентность наград; интеграция в раннер и Home/HUD.
**AC:** см. F2–F5; награды не дублируются; разблокировка только через сервер.

### Phase 9 — Маскот и награды (F7)

**Цель:** эмоциональная петля.
**Задачи:** интеграция `rive-react-native`; экран `reward/{lessonId}`; **нейтральный пул анимаций маскота** (случайный выбор) как базовое поведение; текст похвалы; показ начисленных алмазов; splash-анимация Rive. **Реакция по качеству — за флагом `app_config.economy.mascot_tone_reaction_enabled` (дефолт `false`):** клиент читает флаг на старте; при `true` и наличии анимаций обоих пулов выбирает тон по `errors_count` (celebrate/encourage), иначе — нейтральный пул (мягкий откат, без краша). Сами анимации пулов (10 + 10) добавляются в сборку, когда нарисованы; до этого флаг держим выключенным. `errors_count` уже считается на сервере — бэкенд для включения менять не нужно.
**AC:** см. F7 (анимации играют на обеих платформах).

### Phase 10 — Подписки и пейволл (F8, F10)

**Цель:** монетизация на обеих платформах.
**Задачи:** **RevenueCat** (`react-native-purchases`): конфигурация Offerings/Packages для App Store + Google Play; entitlement `super`; экран Subscription (Super / Family); paywall (триггер battery=0); Edge `revenuecat-webhook` (обновление `subscriptions` + `payment_events`) и `validate-purchase` (реконсиляция); Restore Purchases; отображение цен из стора (RevenueCat) с фолбэком `price_tiers`. **Предложение подписки (§A14):** RPC `get_subscription_offer`; условный CTA (триал «Try 1 week for $0» — Android free-trial / iOS introductory offer — vs обычная покупка по `trial_available`); **персональный таймер** на Subscription и Paywall (тикает до `timer_ends_at`, исчезает при истечении → триал больше не предлагается); запись `trial_started` в `payment_events`.
**AC:** см. F8, F10; покупка/восстановление работают в обоих сторах; триал и таймер управляются независимо из админки; статус приходит вебхуком и виден в приложении.

### Phase 11 — Quests + Магазин (блок F) (F14)

**Цель:** вкладка заданий с алмазными наградами и встроенным магазином заморозки.
**Задачи:** таблицы `quests`/`user_quests` в миграциях/сиде; RPC `get_quests`, `claim_quest_reward` (идемпотентно); экран `quests` (`features/quests`) с секциями Daily / Monthly / Exclusive и **секцией-магазином** (`SHOP`-блок) — покупка заморозки через `buy_streak_freeze`; обработка `INSUFFICIENT_DIAMONDS`/`QUEST_NOT_COMPLETED`; отдельного маршрута `shop` нет.
**AC:** см. F14 (+ F3 в части траты алмазов).

### Phase 12 — Профиль и настройки (F11)

**Цель:** ЛК и все настройки.
**Задачи:** Profile (дубль HUD, ачивки); Settings и подразделы (12.12): Preferences (MMKV + haptics), Profile-edit, Course-management, Account Linking (Google + Apple + Email), Documents (in-app), Support (Restore Subscription, управление подпиской в сторе, LOG OUT); `delete_account`.
**AC:** см. F11.

### Phase 13 — Виджеты и локальные уведомления (F12)

**Цель:** напоминания и виджеты на обеих платформах.
**Задачи:** **Android Glance** + **iOS WidgetKit** (≥2 размера каждый) через config-plugin/нативные таргеты; единый мост данных (App Group на iOS, shared storage на Android); локальные ежедневные напоминания (`expo-notifications`, тумблер в Preferences; запрос разрешений на обеих платформах); deep-link открытия по тапу.
**AC:** см. F12; виджеты обновляются и открывают приложение на iOS и Android.

### Phase 14 — Аналитика, краши и логирование (F13)

**Цель:** наблюдаемость продакшена.
**Задачи:**

- `@react-native-firebase/analytics` — ключевые события (обе платформы; `google-services.json` + `GoogleService-Info.plist`).
- `@react-native-firebase/crashlytics` — неперехваченные краши.
- Клиентское логирование ошибок: пойманные исключения (репозитории, хуки, покупки) → `crashlytics().recordError(e)`. Dev-логи — только в `__DEV__`.
- Серверное логирование: таблица `error_logs` (схема ниже); каждая Edge Function/RPC при пойманной ошибке пишет запись.

**Схема таблицы `error_logs`:**

```sql
create table error_logs (
  id          uuid primary key default gen_random_uuid(),
  source      text not null,   -- 'edge_function' | 'rpc' | 'client'
  function    text not null,   -- название функции/экрана
  platform    text,            -- 'ios' | 'android' | null (для client)
  error_code  text,
  message     text not null,
  payload     jsonb,           -- входные данные вызова (без PII)
  user_id     uuid references profiles(id),
  created_at  timestamptz default now()
);
-- RLS: запись/чтение — только сервисная роль (через админку)
```

**AC:** см. F13.

### Phase 15 — Админ-панель (выделена в отдельный подпроект)

Админка — самостоятельный подпроект со своими фазами, стеком и дизайн-системой. **Полное описание — в Приложении B.** Её фазы (15.1–15.6) идут после релиза MVP-приложения, но БД-фундамент (аналитические view) закладывается раньше.

### Phase 16 — Подготовка к релизу (App Store + Google Play)

**Цель:** публикация в **обоих сторах**.
**Задачи:** dev/prod окружения (два проекта Supabase); EAS Submit в App Store Connect и Google Play Console; **Privacy Policy + App Privacy (Apple) + Data Safety (Google)**; иконки/стор-листинги для обеих платформ; диплинки/Universal Links/App Links (проверка AASA + assetlinks.json); проверка DELETE ACCOUNT (требование обоих сторов); проверка Restore Purchases; тест на реальных устройствах iOS и Android; настройка RevenueCat prod (App Store / Play credentials); App Store review checklist (Sign in with Apple, account deletion, subscription disclosure).
**AC:** release-сборки подключаются к prod; обе платформы проходят ревью сторов; юридические требования Apple и Google выполнены.

> **Вне MVP (после релиза):** FCM/APNs push, Cloudflare R2, расширенный server-side billing, офлайн-режим. **iOS теперь в MVP** (целевая платформа наравне с Android). Блок F (Quests) входит в MVP (Phase 11).

