# 30 · Обзор приложения — Lorebinge SPEC (RN9)

> Часть разбитого спека. Источник-архив: [`spec-rn9.md`](spec-rn9.md) — не редактировать. Содержит: §1–§4 — обзор продукта, технологический стек, архитектурные принципы, структура репозитория. Оглавление, граф зависимостей и карта перекрёстных ссылок — в [00-index.md](00-index.md).

-----

## 1. Обзор продукта (кратко)

Обучающее приложение по истории в стиле Duolingo. Маркетинговая воронка: «затягивающие» ИИ-видео в соцсетях → крючок → установка приложения → обучение по курсам из уроков и заданий (внутри курса — сетка карточек-уроков, §12.4). Геймификация: батарейка (лимит тестов), алмазы (валюта), стрик (дни подряд), маскот-награды, ачивки, подписка.

**Полное описание продукта — в `concept.md`.** Этот спек не дублирует продуктовую логику, а формализует её.

**Три компонента разработки.** Проект состоит из трёх частей, которые делят одну базу данных Supabase: **(A) Мобильное приложение** — React Native, **iOS + Android** (этот документ, §1–17 + Приложение A), **(B) Админ-панель** (Приложение B), **(C) Контент-конвейер / Pipeline** (Приложение C). Все три синхронизированы по схеме БД из §6 — схема является контрактом между ними. Конвейер (C) генерирует курсы офлайн и **импортирует их в те же продуктовые таблицы** (`courses` / `lessons` / `tasks` / `media_assets`), которые читает приложение и редактирует админка.

**Что изменилось в этой версии (RN + iOS).** Бэкенд (Supabase: схема §6, RLS §7, RPC/Edge §8), админка (B) и конвейер (C) **платформо-независимы и не меняются по сути**. Меняется клиент (A): нативный Android-стек заменён на React Native; добавлена вторая платформа — iOS — со всеми вытекающими (StoreKit/IAP, Sign in with Apple, Universal Links, WidgetKit, APNs, App Store Connect). Бэкенд получает минимальные дополнения для второй платформы (валидация чеков Apple, поле `platform=app_store` в подписках/событиях).

-----

## 2. Технологический стек

> Полное обоснование подхода — `concept.md`, раздел 1.27 (адаптируется под RN). Принцип: **один код — две платформы**; нативный код пишем только там, где без него нельзя (виджеты).

|Слой              |Решение                                                                             |Версия/примечание                                                                    |
|------------------|------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------|
|Платформа         |**React Native + Expo** (managed + prebuild/CNG), **TypeScript**                    |iOS **15.1+** (SDK 55) / **16+** (SDK 56); Android **minSdk 24**, target — последний |
|Сборка/CI         |**EAS Build** (iOS `.ipa` + Android `.aab`) + EAS Submit                            |один пайплайн на обе платформы; New Architecture обязателен (SDK 55+)                |
|Язык              |TypeScript (strict)                                                                 |идентификаторы — английский                                                          |
|UI                |**React Native Reusables** (shadcn для RN) поверх **NativeWind** + иконки **Lucide**|см. §5; готовый UI-кит, палитра через tweakcn (`global.css`), без хардкода стилей    |
|Архитектура       |Компоненты + хуки; feature-based; «UI → data» через хуки-репозитории                |аналог MVVM: `Screen` (компонент) + `useXxxScreen()` (хук-«VM») + `UiState` (тип)    |
|Навигация         |**React Navigation v7** (native-stack + bottom-tabs)                                |+ deep links (custom scheme) + Universal Links (iOS) / App Links (Android)           |
|Состояние         |**TanStack Query** (server state) + **Zustand** (ui/client state)                   |кэш запросов, инвалидация; нет глобального DI                                        |
|Async             |Promises / async-await                                                              |через TanStack Query                                                                 |
|Сеть/Бэкенд       |**supabase-js** (`@supabase/supabase-js`) + `react-native-url-polyfill`             |RPC через `.rpc()`, Edge через `.functions.invoke()`                                 |
|Валидация данных  |**Zod**                                                                             |рантайм-валидация DTO/`payload` по типам                                             |
|Локальный кэш     |**MMKV** (`react-native-mmkv`) — настройки/флаги; **expo-sqlite** — кэш контента    |MMKV ⟵ DataStore; SQLite ⟵ Room                                                      |
|Сессия/секреты    |**expo-secure-store** (Keychain/Keystore)                                           |токены сессии Supabase хранятся шифрованно                                           |
|Картинки          |**expo-image**                                                                      |кэш, placeholder, blurhash; ⟵ Coil                                                   |
|Анимации/жесты    |**rive-react-native** (маскот, splash) + **Reanimated 3** + **Gesture Handler**     |Rive — кросс-платформенный рантайм; жесты — свайп-чтение, drag timeline, scrub видео |
|Видео             |**expo-video** (HLS)                                                                |AVPlayer (iOS) / ExoPlayer (Android) под капотом; ⟵ Media3                           |
|Виджеты           |**WidgetKit (iOS)** + **Glance (Android)**                                          |нативные таргеты через prebuild/config-plugin; единый JS-мост данных                 |
|Локальные уведомл.|**expo-notifications**                                                              |MVP — локальные; APNs (iOS) + FCM (Android) — позже                                  |
|Тактильность      |**expo-haptics**                                                                    |⟵ Android Vibration; кросс-платформенно                                              |
|Бэкенд            |Supabase (PostgreSQL)                                                               |RLS обязателен (без изменений)                                                       |
|Логика            |Postgres RPC + Edge Functions                                                       |server-authoritative (без изменений)                                                 |
|Файлы             |Supabase Storage → Cloudflare R2 (позже)                                            |без изменений                                                                        |
|Видео-хостинг     |Cloudflare Stream                                                                   |без изменений                                                                        |
|Платежи           |**RevenueCat** (`react-native-purchases`)                                           |абстрагирует **Google Play Billing + Apple StoreKit**; серверная валидация + webhooks|
|Аналитика/краши   |**@react-native-firebase** (Analytics + Crashlytics)                                |обе платформы; `crashlytics().recordError(e)`                                        |
|i18n              |**i18next** + `expo-localization`                                                   |все строки в ресурсах (`en`), готово к доп. языкам; ⟵ `strings.xml`                  |
|Safe area/иконки  |`react-native-safe-area-context`, `expo-splash-screen`                              |нотчи/Dynamic Island, единый splash для обеих платформ                               |
|Админка           |React 19 + Vite + Tailwind + Supabase JS                                            |отдельный подпроект — см. Приложение B (shadcn/ui); включает **раздел апрувера конвейера** (Приложение D)                             |
|Конвейер          |Python (APScheduler + asyncio) — воркер                                             |отдельный Python-воркер на Mac mini — см. Приложение C; с приложением и админкой общается **только через Supabase** |
|Репозиторий       |Монорепо                                                                            |`/app` (RN), `/admin`, `/pipeline`, `/supabase`                                      |


> **Платежи — почему RevenueCat.** Подписки нужны в двух сторах одновременно (Apple + Google) с серверной валидацией, восстановлением, гибким управлением офферами и единым источником статуса. RevenueCat закрывает обе платформы одним SDK + вебхуками и снимает с нас реализацию двух разных серверных валидаций. **Альтернатива без 3rd-party:** `react-native-iap` + собственная двойная серверная валидация (Google Play Developer API **и** App Store Server API) в Edge Function — оставлена как fallback, но не дефолт.

> **Expo Router как альтернатива навигации.** React Navigation выбран для явного контроля над графом (§10) и совместимости с монорепо-папкой `/app`. Expo Router (file-based) — допустимая альтернатива с типизированными маршрутами; при переходе на него учесть конфликт имён каталога роутера и папки `/app`.

-----

## 3. Архитектурные принципы

1. **Server-authoritative.** Вся экономика (батарейка, алмазы, стрик) и логика разблокировки уроков выполняются на сервере (Postgres RPC). Клиент только вызывает и отображает. Клиенту никогда не доверяем подсчёт валюты/прогресса.
1. **Single source of truth.** Состояние пользователя живёт в БД. Клиент кэширует (TanStack Query + MMKV/SQLite) для скорости, но при расхождении прав сервер.
1. **Один код — две платформы.** Пишем кросс-платформенно. Платформенные ветки — только там, где это неизбежно (`Platform.select`, `*.ios.tsx` / `*.android.tsx`, нативные модули виджетов). Любая платформо-зависимая логика изолируется в `src/platform/`.
1. **Слои (аналог MVVM):**

- **UI** (компонент-`Screen` + базовые компоненты): экраны, отображение `UiState`, обработка ввода.
- **«ViewModel»** — хук `useXxxScreen()`: держит состояние экрана (`UiState` — discriminated union), дергает репозитории/RPC, маппит в модель экрана.
- **Data** (репозитории-функции/хуки + источники): supabase-js, SQLite-кэш, MMKV, secure-store.

1. **Каждый блок Home — независимый компонент** (см. раздел 11), не знает о своём расположении.
1. **Все строки — в i18n-ресурсах** (`src/core/i18n/en.json`), даже на одном языке. Никаких хардкод-строк в UI.
1. **Все визуальные параметры — в теме** (дизайн-токены, раздел 5). Никаких хардкод-цветов/размеров в компонентах.
1. **Безопасность ключей.** Серверные секреты (Google Play / App Store service credentials, ключи ИИ) — только на сервере (Edge Functions) / в RevenueCat. В JS-бандл не попадают (даже `EXPO_PUBLIC_*` считаются публичными).
1. **Идемпотентность операций.** Начисление наград/списания защищены от двойного срабатывания (леджер транзакций + проверки на сервере).
1. **New Architecture.** Проект работает на New Architecture RN (обязательно в Expo SDK 55+). Нативные модули и библиотеки выбираются совместимые с ней.

-----

## 4. Структура репозитория и проекта

### Монорепо (корень)

```
/
├── app/                  # Мобильное приложение (React Native + Expo, TypeScript) — iOS + Android
├── admin/                # Веб-админка (React + Vite)
├── pipeline/             # Контент-конвейер (Python) — см. Приложение C
├── supabase/             # Миграции БД, RPC, RLS, Edge Functions
│   ├── migrations/       # SQL-миграции (нумерованные; включая схему pipeline)
│   ├── functions/        # Edge Functions (TypeScript) — incl. validate-purchase, revenuecat-webhook
│   └── seed.sql          # Стартовые данные (категории, конфиг)
├── docs/                 # concept.md, spec-rn.md, тест-стратегия (отдельно), прочее
└── README.md
```

### Мобильный проект (`/app`)

```
app/
├── app.config.ts                # Expo config: iOS + Android, plugins, deep links, scheme, версии
├── eas.json                     # EAS Build/Submit профили (dev/preview/prod), обе платформы
├── package.json
├── tsconfig.json
├── index.ts                     # entry (registerRootComponent)
├── App.tsx                      # провайдеры: SafeArea, Theme, QueryClient, RevenueCat, Navigation
├── assets/                      # шрифты, картинки, rive (.riv), splash, иконки, default_ui_config.json
├── ios/                         # нативный iOS-проект (prebuild/CNG) — таргет WidgetKit живёт здесь
├── android/                     # нативный Android-проект (prebuild/CNG) — Glance-виджет здесь
└── src/
    ├── app/                     # навигация: NavRoutes.ts, RootNavigator.tsx, linking (deep/universal)
    ├── core/
    │   ├── theme/               # токены: colors.ts, typography.ts, spacing.ts, radii.ts; ThemeProvider, useTheme
    │   ├── components/          # AppButton, AppCard, Pill, ProgressBar, ...
    │   ├── i18n/                # en.json (строки), keys, t()
    │   └── common/              # AppResult, общие типы, утилиты, маппинг ошибок
    ├── data/
    │   ├── supabase/            # client.ts, DTO (zod-схемы), rpc-обёртки, edge-клиенты
    │   ├── storage/             # mmkv.ts (настройки), session.ts (secure-store), cache (sqlite)
    │   ├── repository/          # репозитории (Profile, Course, Progress, Config, Economy, ...)
    │   └── model/               # доменные модели (TS-типы)
    ├── features/                # экраны по фичам: Screen + хуки + UiState
    │   ├── splash/
    │   ├── auth/
    │   ├── home/                # Home + блоки A–E
    │   │   └── blocks/          # GridBlock, LoresBlock, InProgressBlock, ...
    │   ├── coursepath/          # сетка уроков курса (карточки с обложками)
    │   ├── lesson/              # урок + раннер заданий
    │   │   └── tasks/           # VideoTask, ReadingTextTask, ReadingMediaTask, MatchingTask, FillBlankTask, ...
    │   ├── reward/              # маскот + награды
    │   ├── quests/              # блок F (задания + встроенный магазин заморозки)
    │   ├── subscription/        # блок G + paywall
    │   ├── profile/             # блок H + settings
    │   │   └── settings/        # preferences, profile, course, linking, docs
    │   └── player/              # общий видео-плеер (пауза/перемотка) для видео-заданий
    ├── hud/                     # HUD-компонент (battery/diamonds/streak ↔ ассет подписки)
    └── platform/                # платформо-зависимое: iap (RevenueCat), widgets-bridge, notifications, haptics, apple-auth
```

> Один проект (не монорепо внутри `/app`). Деление на `features/` — папочное, не модульное. Нативные каталоги `ios/`/`android/` генерируются prebuild (CNG); ручной нативный код (WidgetKit/Glance) добавляется через config-plugins, чтобы переживать `prebuild --clean`.

