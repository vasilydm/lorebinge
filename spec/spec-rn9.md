# SPEC (React Native) — Обучающее приложение по истории (геймификация) — RN9

> **RN9 (что изменилось vs RN8).** Апрувер конвейера перенесён в **админку** (Приложение B): один веб-проект на **React + shadcn/ui + Tailwind**, адаптивный под телефон. Убраны **Tailscale**, **Jinja2/HTMX** и любой туннель. Mac mini (конвейер) и админка общаются **только через Supabase** (таблицы `pipeline.*`); прямого соединения между ними нет. Единый дизайн на приложение + админку + апрувер — через одну палитру (§5).

> **Назначение документа.** Это технический спек для пошаговой разработки через Claude Code (spec-driven development), **переписанный под React Native**. Концепт продукта — в `concept.md`; этот файл переводит концепт в инженерные требования: схему данных, контракты API, структуру кода, навигацию, дизайн-токены и дорожную карту по фазам.
> 
> **Кросс-платформенность.** В отличие от предыдущей версии (нативный Android/Kotlin), приложение теперь **одно на двух платформах — iOS и Android** — на едином коде React Native. Каждое инженерное решение в этом документе учитывает обе платформы; платформенные различия (платежи, виджеты, sign-in, deep links, пуши, подписание, стор-листинги) проговариваются явно.
> 
> **Тестирование вынесено.** Из этого документа **намеренно удалено всё про тестирование** (unit/UI/SQL/smoke, TDD, Definition of Done в части тестов). Стратегия и требования к тестам описываются в отдельном файле.
> 
> **Как использовать.** Разработка идёт по фазам из раздела 14 строго по порядку. Каждая фаза имеет цель, задачи, артефакты (deliverables) и критерии приёмки (acceptance criteria). Не переходить к следующей фазе, пока критерии текущей не выполнены.
> 
> **Рабочее название:** `Lorebinge`. Android `applicationId` / iOS `bundleIdentifier`: `com.lorebinge.app`.
> 
> **Язык интерфейса:** английский. Язык кода/идентификаторов: английский. Язык этого документа: русский.

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

-----

## 5. Дизайн-система (UI-кит)

> **Принцип.** Свой дизайн не рисуем. Берём готовый UI-кит + готовую палитру + готовый набор иконок. Сгенерированные ранее стартовые токены (фиолетовая тема, числовые значения отступов/радиусов/типографики) **отменены** и использоваться не должны. Цвета и размеры в коде компонентов **не хардкодятся** (см. §3) — всё идёт через токены кита.

### 5.1. UI-кит

- **Выбор: React Native Reusables** (shadcn-подход для RN) поверх **NativeWind** (Tailwind для RN). Совместим с Expo и New Architecture (Fabric/TurboModules).
- **Модель «copy-paste»:** компоненты копируются в репозиторий (в `src/components/ui/`), их исходник принадлежит проекту — агент правит компонент у нас, а не борется с `node_modules`.
- **MCP для Claude Code:** к Claude Code подключается **shadcn MCP-сервер** (`npx`, без ключей для базового режима). Агент через него ищет/ставит/смотрит реальные компоненты, а не выдумывает их API — это резко снижает галлюцинации по пропсам. Установку MCP и базовую инициализацию кита выполнить в начале UI-фазы.
- Альтернатива (fallback, не дефолт): **gluestack-ui v3** — если откажемся от Tailwind-воркфлоу.

### 5.2. Механизм темы (как это работает технически)

- Цвета темы живут **CSS-переменными в одном файле `global.css`**: отдельные значения для light и dark.
- В `tailwind.config` переменные подключены как семантические токены (`background`, `foreground`, `primary`, `card`, `border`, `destructive` и т.д.).
- Компоненты используют классы вида `bg-primary`, `text-foreground`, `border-border` — без хардкод-цветов.
- Переключение light/dark — встроенное в NativeWind по системной теме (`useColorScheme`); ручного управления темой по возможности избегаем (пусть фреймворк делает сам).

```css
/* src/global.css — пример структуры (значения подставляет палитра из 5.3) */
:root        { --background: ...; --foreground: ...; --primary: ...; /* … */ }
.dark:root   { --background: ...; --foreground: ...; --primary: ...; /* … */ }
```

### 5.3. Палитра (цвета) — через tweakcn

Палитра берётся не из головы, а генерируется в **tweakcn** (визуальный редактор тем для shadcn, free, без логина: настраиваешь light+dark, получаешь блок CSS-переменных).

**Цепочка применения:**

1. В **tweakcn** подобрать палитру (light + dark) → Export/Copy → получить блок CSS-переменных.
1. Вставить блок в **`src/global.css`** (заменить значения `:root` и `.dark`). Это единственное место — компоненты не трогаются.
1. Перезапуск с очисткой кэша: **`npx expo start --clear`** (без сброса кэша смена темы может не подхватиться).

**Нюанс формата (обязательно проверить):** имена переменных у tweakcn и RN Reusables совпадают (обе на shadcn), но **формат значения** может различаться — RN Reusables исторически ждёт **HSL-каналы** (`0 0% 100%`), а tweakcn под Tailwind v4 по умолчанию отдаёт **OKLCH**. Решение: либо переключить формат вывода в tweakcn, либо поручить Claude Code привести значения под формат, который ждёт наш `tailwind.config`. NativeWind v5 (под Tailwind v4) поддерживает CSS-переменные и P3 — при использовании v5 формат OKLCH допустим.

> Порядок не важен: можно вставить палитру до сборки (агент сразу пишет в наших цветах) или после (приложение перекрашивается разом, без правок компонентов).

> **Единая палитра на весь проект.** Эта же палитра — единый источник цвета для приложения (RN/NativeWind), админки (Приложение B, на shadcn/ui) — **внутри которой живёт раздел апрувера конвейера** (Приложение D), на том же React + shadcn/ui. tweakcn отдаёт стандартные CSS-переменные shadcn/Tailwind, поэтому и приложение, и админка (вместе с апрувером) потребляют один и тот же экспорт напрямую. Бренд-цвет фиксируется здесь один раз и не дублируется хардкодом в B/D.

### 5.4. Иконки — Lucide

- **Набор: Lucide** (`lucide-react-native`) — канонический набор для shadcn/RN Reusables, 1500+ иконок, SVG через `react-native-svg`, tree-shakable. Иконка = именованный компонент: home → `Home`, profile → `User`, и т.д.
- **Стилизация через тему:** иконки получают цвет теми же NativeWind-классами (`text-primary`, `text-foreground`), поэтому **автоматически следуют палитре из 5.3** — отдельной настройки цвета иконок не требуется.
- **className на иконках:** обернуть через `iconWithClassName` (паттерн RN Reusables) либо использовать drop-in `lucide-nativewind`, чтобы иконки принимали `className`.
- Иконки своими руками не рисуем; нестандартную иконку при необходимости заводим как кастомный SVG-компонент и оборачиваем тем же `iconWithClassName`.

### 5.5. Что остаётся жёстким требованием (независимо от палитры)

- Поддержка **light и dark** на обеих платформах.
- **Тема как контракт:** никакого хардкода цветов/размеров в компонентах — только токены/классы кита (см. §3).
- **Единицы.** RN использует безразмерные density-independent пиксели (аналог `dp`/`pt`); масштабирование под плотность — автоматическое. Размеры шрифта учитывают системный масштаб (Dynamic Type / Font size); критичные элементы при необходимости ограничивают `allowFontScaling`/`maxFontSizeMultiplier`.

### 5.6. Семантические роли токенов

Остальные разделы спека ссылаются на роли (`primary`, `surface`/`card`, `success`, `error`/`destructive`, состояния карточки урока, батарейка, стрик, алмаз и т.д.) **по смыслу**, а не по конкретному цвету. Роли реализуются токенами палитры из 5.3; для ролей, которых нет в дефолтном наборе shadcn (например, `streak`, `diamond`, состояния карточки урока), добавляем собственные CSS-переменные в `global.css` и токены в `tailwind.config` по той же схеме.

> **Токены состояний карточки урока (`node-*`, §12.4).** `node-active` — акцент текущего урока (цвет пульса/рамки-фолбэка); `node-locked` — приглушённый тон для иконки замка и desaturate-overlay будущего урока. Состояние `completed` отдельного токена-акцента **не требует** (обложка в обычном цвете, без бейджей). Имя `node-*` историческое (раньше — ноды дороги); семантика теперь «состояние карточки урока в сетке».

### 5.7. TODO до старта UI-фаз

1. Подключить shadcn MCP к Claude Code; инициализировать React Native Reusables + NativeWind.
1. Сгенерировать палитру в tweakcn (light+dark), вставить в `src/global.css`, привести формат под `tailwind.config`.
1. Поставить `lucide-react-native`, настроить `iconWithClassName`/`lucide-nativewind`.
1. Завести доп-токены для проектных ролей (`streak`, `diamond`, `node-active`, `node-locked`, `battery*`).
1. Зафиксировать шрифт и типографическую шкалу как токены кита (без хардкода).

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
|`cover_image_url`        |text null              |обложка для сетки (блок A Grid) и карточек In Progress                                                                                                                                                                       |
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

-----

## 8. Backend API (контракты)

### 8.1 Postgres RPC-функции (`SECURITY DEFINER`)

> **Без изменений** (вызываются через `supabase.rpc(...)`). Все возвращают JSON, проверяют `auth.uid()`.

|Функция                 |Вход                           |Выход                                                                                                                |Логика (кратко)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
|------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|`submit_task_attempt`   |`task_id uuid, is_correct bool`|`{ battery, was_correct, is_subscriber }`                                                                            |Записывает attempt. Если подписка активна (`trial`/`active`) — батарея не тратится. Иначе для теста (`requires_check=true`): `battery=0` → `BATTERY_EMPTY` (→ paywall); иначе `battery -= battery_cost_per_answer` (за любой ответ). `is_correct=false` → `errors_count += 1`. Контентные задания батарею не трогают. (Бонус заряда за урок начисляет `complete_lesson`, §A4.)                                                                                                                                                                                                            |
|`complete_lesson`       |`lesson_id uuid`               |`{ unlocked_next_lesson_id, diamonds_awarded, battery_awarded, new_diamond_balance, errors_count, course_completed }`|Проверяет, что все задания пройдены. `status=completed`, `errors_count` (число неверных тест-ответов, для маскота), разблокировка следующего урока, обновление `lessons_completed`. **За каждый урок → случайные алмазы** из взвешенного `lesson_diamond_rewards` (идемпотентно по леджеру `ref_id=lesson_id`, §A6) **и фикс `battery_reward_per_lesson` заряда** (`battery = min(battery_max, battery + reward)`; не для подписчика; идемпотентно — повторное прохождение не доначисляет, §A4). Последний урок → курс completed, `knowledge_level += 1`, ачивки. Вызывает `touch_streak`.|
|`touch_streak`          |—                              |`{ current_streak, longest_streak }`                                                                                 |Инкремент стрика по дню; обработка пропуска/заморозки (см. A5).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
|`refill_battery`        |—                              |`{ battery, next_refill_at }`                                                                                        |Ленивое восстановление по `battery_refill_minutes` (см. A4). Вызывается при заходе и периодически.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
|`buy_streak_freeze`     |`days int`                     |`{ diamonds, freezes_available }`                                                                                    |Списывает `days * cost` алмазов (проверка баланса), +заморозки, леджер. `INSUFFICIENT_DIAMONDS` если не хватает.                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
|`apply_streak_freeze`   |—                              |`{ current_streak, freezes_available }`                                                                              |Тратит 1 заморозку, прощает пропущенный день.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
|`reset_course_progress` |`course_id uuid`               |`{ ok }`                                                                                                             |Удаляет прогресс по курсу.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
|`get_subscription_offer`|—                              |`{ trial_available, trial_days, timer_active, timer_ends_at, timer_expired }`                                        |**Server-authoritative логика предложения (§A14).** Читает `app_config.economy.trial_offer` и `subscriptions`. Ставит `offer_timer_started_at` на первом вызове при включённом таймере. Клиент только отображает.                                                                                                                                                                                                                                                                                                                                                                         |
|`get_home_feed`         |—                              |`{ grid:[], lores:[…], in_progress:[], popular:[], categories:[] }`                                                  |Агрегирует все блоки Home одним запросом. `lores[]` с вычисленными `lessons_count`/`tasks_count`. Все опубликованные курсы.                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
|`get_course_path`       |`course_id uuid`               |`{ course:{ id,title,path_background_color }, lessons:[{id,title,cover_image_url,sort_order,status}] }`              |Сетка уроков со статусами (`completed`/`active`/`locked`) для пользователя. Возвращает обложку каждого урока и цвет фона сетки.                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
|`get_quests`            |—                              |`{ daily:[], monthly:[], exclusive:[], shop:{ freeze_cost_per_day } }`                                               |Блок F: активные задания + параметры магазина заморозки.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
|`claim_quest_reward`    |`quest_id uuid`                |`{ diamonds, claimed }`                                                                                              |Начисляет алмазы (идемпотентно по леджеру `quest_reward`/`ref_id`). `QUEST_NOT_COMPLETED` если не выполнено.                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
|`delete_account`        |—                              |`{ ok }`                                                                                                             |Удаляет данные пользователя (каскад) и аккаунт. **Требование и Google Play, и App Store** (Apple 5.1.1(v)).                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |

**Транзакционность.** `complete_lesson` — одна транзакция; начисление алмазов идемпотентно (проверка по леджеру `ref_id`).

### 8.2 Edge Functions (TypeScript)

|Функция             |Метод         |Вход                          |Выход                             |Назначение                                                                                                                                                                                                                                                                                                                                                          |
|--------------------|--------------|------------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|`revenuecat-webhook`|POST (webhook)|RevenueCat event payload      |200                               |**Источник истины по подпискам обеих платформ.** Принимает события RevenueCat (initial_purchase, renewal, trial_started, cancellation, expiration, billing_issue) и обновляет `subscriptions` + пишет `payment_events` (с `platform`/`store`, суммой, валютой, страной). Проверка подписи вебхука. Заменяет прежний `play-rtdn` и покрывает App Store + Google Play.|
|`validate-purchase` |POST          |`{ rc_app_user_id }` (или чек)|`{ status, expires_at, platform }`|Реконсиляция статуса по запросу клиента после покупки/восстановления: сверяется с RevenueCat REST API, обновляет `subscriptions`. (Fallback без RevenueCat: валидация чека Apple через App Store Server API **или** Google Play Developer API по `platform`.) Ключи — в секретах Edge Functions.                                                                    |


> **Поток подписки (RN).** Клиент покупает/восстанавливает через RevenueCat SDK (`react-native-purchases`) → RevenueCat обновляет entitlement → шлёт `revenuecat-webhook` → Edge Function пишет `subscriptions`. Клиент дополнительно может дернуть `validate-purchase` для немедленной реконсиляции. Статус в приложении читается из `subscriptions` (server-authoritative), не из локального кэша SDK.

-----

## 9. Доменные модели (TypeScript, `src/data/model/`)

> Kotlin data-классы заменены TypeScript-типами. Рантайм-валидация DTO — через zod (`src/data/supabase/dto.ts`), маппинг DTO → доменные модели — в репозиториях.

```ts
export interface UserProfile {
  id: string;
  username: string;
  displayName: string;
  avatarUrl: string | null;
  knowledgeLevel: number;
  diamonds: number;
  battery: number;
  batteryMax: number;
  isSubscriber: boolean;              // trial/active → батарея заменена ассетом
  subscriptionBadge: SubscriptionBadge | null;
}

export interface SubscriptionBadge {
  assetUrl: string;
  assetType: 'rive' | 'lottie' | 'svg' | 'png';
  label: string | null;
}

export interface Streak {
  current: number;
  longest: number;
  freezesAvailable: number;
}

export interface Category { id: string; name: string; slug: string; }

export interface Course {
  id: string;
  title: string;
  categoryId: string | null;
  categoryName: string | null;        // пилюля-категория на карточке Lores
  description: string | null;
  coverImageUrl: string;
  illustrationUrl: string | null;     // главный логотип карточки Lores
  cardDecorationUrl: string | null;   // декор внизу справа
  cardGradientStart: string | null;
  cardGradientEnd: string | null;     // градиент карточки Lores
  pathBackgroundColor: string | null;
  totalLessons: number;               // кол-во уроков (на карточке)
  totalTasks: number;                 // кол-во заданий (на карточке)
  popularityScore: number;
}

export interface CourseProgress {
  courseId: string;
  lessonsCompleted: number;
  totalLessons: number;
  lastLessonId: string | null;
}
export const courseFraction = (p: CourseProgress): number =>
  p.totalLessons === 0 ? 0 : p.lessonsCompleted / p.totalLessons;

export type LessonStatus = 'locked' | 'active' | 'completed';

export interface Lesson {
  id: string; courseId: string; title: string;
  sortOrder: number; status: LessonStatus;
}

export type TaskType =
  | 'video' | 'reading_text' | 'reading_media' | 'matching' | 'fill_blank'
  | 'image_recognition' | 'multiple_choice' | 'true_false' | 'timeline';

export interface Task {
  id: string;
  lessonId: string;
  type: TaskType;
  sortOrder: number;
  requiresCheck: boolean;             // false → Continue; true → disabled-действие → Done (см. 6.5а)
  payload: TaskPayload;               // discriminated union по type
}

// Сегмент текста для reading_text/reading_media
export interface ReadingSegment { text: string; audioUrl: string | null; }
export interface ReadingTextPayload { kind: 'reading_text'; title: string; segments: ReadingSegment[]; }

export type ReadingMediaKind = 'image' | 'video';
export interface ReadingMedia { kind: ReadingMediaKind; imageUrl: string | null; mediaAssetId: string | null; }
export interface ReadingMediaPayload { kind: 'reading_media'; media: ReadingMedia; title: string; segments: ReadingSegment[]; }

export interface VideoPayload { kind: 'video'; mediaAssetId: string; autoplay: boolean; }
export interface MatchingPayload { kind: 'matching'; prompt: string; pairs: { left: string; right: string }[]; }
export interface FillBlankPayload { kind: 'fill_blank'; text: string; answer: string; options?: string[]; }
export interface ImageRecognitionPayload { kind: 'image_recognition'; imageUrl: string; prompt: string; options: string[]; answerIndex: number; }
export interface MultipleChoicePayload { kind: 'multiple_choice'; prompt: string; options: string[]; answerIndex: number; }
export interface TrueFalsePayload { kind: 'true_false'; statement: string; answer: boolean; }
export interface TimelinePayload { kind: 'timeline'; prompt: string; items: { id: string; label: string; year: number }[]; }

export type TaskPayload =
  | VideoPayload | ReadingTextPayload | ReadingMediaPayload | MatchingPayload
  | FillBlankPayload | ImageRecognitionPayload | MultipleChoicePayload
  | TrueFalsePayload | TimelinePayload;

export interface MediaAsset {
  id: string; kind: 'video' | 'tts_audio';
  hlsUrl: string | null; posterUrl: string | null;
  audioUrl: string | null; textContent: string | null;
  durationSeconds: number | null;
}

export type SubscriptionStatus = 'none' | 'trial' | 'active' | 'expired' | 'cancelled';
export type StorePlatform = 'app_store' | 'google_play';
export interface Subscription {
  status: SubscriptionStatus;
  productId: string | null;
  platform: StorePlatform | null;
  expiresAt: number | null;           // epoch ms
}

export interface EconomyConfig {
  startingDiamonds: number; batteryMax: number;
  batteryCostPerAnswer: number; batteryRefillMinutes: number; batteryRefillAmount: number;
  lessonsWithoutErrorsForReward: number; diamondRewardAmount: number;
  streakFreezeCostPerDay: number;
  mascotToneReactionEnabled: boolean;
}
```

**Обёртка результата** (`src/core/common/result.ts`):

```ts
export type ErrorType =
  | 'NETWORK' | 'BATTERY_EMPTY' | 'INSUFFICIENT_DIAMONDS'
  | 'QUEST_NOT_COMPLETED' | 'AUTH' | 'SERVER' | 'UNKNOWN';

export type AppResult<T> =
  | { ok: true; data: T }
  | { ok: false; type: ErrorType; message?: string };
```

> **UiState экранов** — discriminated union, напр. `type HomeUiState = { kind: 'loading' } | { kind: 'content'; blocks: HomeBlocksData } | { kind: 'error'; message: string }`. Возвращается хуком `useHomeScreen()` и потребляется компонентом-`Screen`.

-----

## 10. Навигация (граф маршрутов)

> **React Navigation v7.** Root — native-stack; внутри — `BottomTab` с 4 вкладками, **каждая со своим вложенным native-stack** (сохранение back stack на вкладку). Параметры между экранами — **только id**. Тип-безопасность через `RootStackParamList`/`TabParamList`.

```
RootStack (native-stack)
├── splash
├── auth (native-stack)
│   ├── auth/login
│   └── auth/register            (email + код подтверждения)
├── main  (BottomTabs — 4 вкладки, каждая со своим native-stack)
│   ├── home          (вкладка 1)
│   ├── quests        (вкладка 2, блок F — задания + магазин заморозки)
│   ├── subscription  (вкладка 3, блок G)
│   └── profile       (вкладка 4, блок H)
│         └── settings
│               ├── settings/preferences
│               ├── settings/profile
│               ├── settings/course
│               ├── settings/account_linking
│               ├── settings/help_center      (in-app webview/native)
│               ├── settings/send_feedback
│               ├── settings/terms            (in-app)
│               └── settings/privacy          (in-app)
├── course_path/{courseId}        (сетка уроков; HUD виден)    — presentation: card
├── lesson/{lessonId}             (раннер заданий; HUD скрыт) — fullScreenModal
├── reward/{lessonId}             (маскот + награда)          — modal
└── paywall                       (при battery=0 или из подписок) — modal
```

> Магазин заморозки (бывший экран `shop`/блок J) не отдельный маршрут — встроен в экран `quests`.

**Deep links / Universal Links (заложить структуру сразу, обе платформы):**

- Custom scheme: `lorebinge://course/{courseId}` → открывает `course_path/{courseId}`.
- **HTTPS-ссылки для маркетинга:** `https://lorebinge.app/course/{id}` → тот же экран.
  - **iOS Universal Links:** файл `apple-app-site-association` на домене + Associated Domains entitlement (`applinks:lorebinge.app`).
  - **Android App Links:** `assetlinks.json` (Digital Asset Links) на домене + intent-filter (`autoVerify`).
- Конфигурируется в React Navigation `linking` (`prefixes` + `config`); `scheme` и associated domains — в `app.config.ts` (`scheme`, `ios.associatedDomains`, `android.intentFilters`).

**Скрытие HUD:** HUD — общий компонент, показывается на `home`, `course_path`; скрыт на `lesson`, `reward`, `paywall` (полноэкранное видео).

**HUD: батарея ↔ ассет подписки.** Слот заряда отображается по статусу подписки:

- **Не-подписчик** (`subscriptions.status` ∈ `none`/`expired`/`cancelled`) → иконка батареи + счётчик `battery/battery_max`; цвет по токенам `batteryNormal`/`batteryLow` (красный при ≤3).
- **Подписчик** (`trial`/`active`) → вместо батареи яркий **ассет из `app_config.subscription_badge`** (Rive/Lottie/SVG/PNG по `asset_type`, опц. `label`). Заряд скрыт.
- Статус берётся из `subscriptions` (server-authoritative), кэш-фолбэк — последнее известное значение (MMKV).

> **Safe area / системные жесты.** Все экраны учитывают инсеты (`react-native-safe-area-context`): нотч/Dynamic Island (iOS), системную навигацию-жестами (Android). Модальные экраны (`lesson`, `paywall`) — с корректной кнопкой/жестом закрытия на обеих платформах; на iOS свайп-вниз для модалок при необходимости отключается на `lesson` (чтобы не прерывать урок случайно).

-----

## 11. Server-driven UI (схема конфига)

> Раскладка блоков и вкладок задаётся `ui_configs.config` (jsonb). Приложение содержит **встроенный дефолтный конфиг** (`assets/default_ui_config.json`, на случай отсутствия сети) и подтягивает активный при старте. **Смена раскладки привязана к build-номеру приложения** (`min_app_version`), не применяется на лету (см. 1.27, A11).

### Формат `config`

```json
{
  "version": 1,
  "tabs": [
    {
      "id": "home", "icon": "home", "label_key": "nav_home", "enabled": true, "order": 1,
      "blocks": [
        { "id": "grid",        "type": "GRID",        "enabled": true, "order": 1 },
        { "id": "lores",       "type": "LORES",       "enabled": true, "order": 2 },
        { "id": "in_progress", "type": "IN_PROGRESS", "enabled": true, "order": 3 },
        { "id": "popular",     "type": "POPULAR",     "enabled": true, "order": 4 },
        { "id": "categories",  "type": "CATEGORIES",  "enabled": true, "order": 5 }
      ]
    },
    { "id": "quests", "icon": "quests", "label_key": "nav_quests", "enabled": true, "order": 2,
      "blocks": [ { "id": "quests", "type": "QUESTS", "enabled": true, "order": 1 },
                  { "id": "shop",   "type": "SHOP",   "enabled": true, "order": 2 } ] },
    { "id": "subscription", "icon": "crown", "label_key": "nav_sub", "enabled": true, "order": 3,
      "blocks": [ { "id": "subscription", "type": "SUBSCRIPTION", "enabled": true, "order": 1 } ] },
    { "id": "profile", "icon": "profile", "label_key": "nav_profile", "enabled": true, "order": 4,
      "blocks": [ { "id": "profile", "type": "PROFILE", "enabled": true, "order": 1 } ] }
  ]
}
```

> Магазин заморозки (`SHOP`) — блок **внутри вкладки Quests**, а не отдельная вкладка.

### Клиентская реализация (RN)

- **Реестр блоков:** `type BlockType = 'GRID' | 'LORES' | 'IN_PROGRESS' | 'POPULAR' | 'CATEGORIES' | 'QUESTS' | 'SUBSCRIPTION' | 'PROFILE' | 'SHOP'`.
- **Рендерер:** компонент `<RenderBlock type={...} />` мапит `BlockType` → соответствующий компонент через объект-словарь `Record<BlockType, React.ComponentType>`. Блок получает данные из своего хука, не зная о расположении.
- Каждая вкладка рендерит свой список `blocks` (отсортированный, только `enabled`) в **`FlatList`** (ленивый рендер; ⟵ LazyColumn).
- **Неизвестный `type`** (конфиг новее приложения) — пропускается без краха (`if (!registry[type]) return null`).
- **Иконки вкладок** — мапинг `icon`-строки → компонент иконки (напр. из `lucide-react-native`/`@expo/vector-icons`); неизвестная иконка → дефолтная.

-----

## 12. Спецификации экранов

> Формат: назначение · состояния UiState · ключевые действия. Каждый экран = `Screen` (компонент) + `useXxxScreen()` (хук) + `UiState` (тип). Списки — `FlatList`; жесты — Gesture Handler + Reanimated; картинки — `expo-image`; видео — `expo-video`; Rive — `rive-react-native`.

### 12.1 Splash

- **Назначение:** Rive-анимация на цветном фоне; проверка сессии; загрузка конфига и economy. Нативный splash (`expo-splash-screen`) удерживается до готовности JS, затем — Rive-сплэш.
- **Переход:** сессия есть → `home`; нет → `auth/login`. Сервер недоступен → заглушка с текстом «Service is temporarily unavailable. Please try again later.» (англ.) + кнопка Retry.

### 12.2 Auth (login / register)

- **Login:** кнопка «Continue with Google»; **на iOS обязательна** кнопка **«Sign in with Apple»** (Apple Guideline 4.8 — при наличии стороннего соц-входа); (опц.) email+password.
  - Google — `@react-native-google-signin/google-signin` (или Supabase OAuth); Apple — `expo-apple-authentication` → передача identity-token в `supabase.auth.signInWithIdToken`.
- **Register (email-путь):** ввод email → код на почту → подтверждение. Соц-вход (Google/Apple) кода не требует.
- **UiState:** `idle | loading | error(message)`.
- **Действие:** успешный вход → создаётся/находится `profiles` (стартовые алмазы) → `home`. Сессия Supabase сохраняется в `expo-secure-store` (Keychain/Keystore).

### 12.3 Home (вкладка 1)

- **Назначение:** HUD сверху + лента блоков из конфига (A–E).
- **UiState:** `loading | content(blocksData) | error`.
- **Данные:** один вызов `get_home_feed` (через TanStack Query).
- **Действия:** тап обложки (Grid) или карточки (Lores/InProgress/Popular) → `course_path/{id}`; тап категории → список курсов категории.

#### 12.3а Карточка Lores (Block B / `LoresBlock`) — детально

- **Контейнер блока:** заголовок секции **«Lores»** + подзаголовок; ниже — горизонтальный список (`FlatList horizontal`, `snapToInterval`/`pagingEnabled` по желанию). Карточек = числу опубликованных курсов; скролл бесконечный «до упора». Данные — `lores[]` из `get_home_feed`.
- **Одна карточка (`LoreCard`)** — вертикальная, фон = **линейный градиент** `card_gradient_start → card_gradient_end` (`expo-linear-gradient`, сверху вниз), скругление `radiusLarge`. Раскладка сверху вниз:

1. **Логотип** — `illustration_url`, крупный, по центру в верхней трети (`expo-image`).
1. **Пилюля категории** — `category_name` в `Pill` (светлый фон, текст = `primary`), UPPERCASE.
1. **Название** — `title`, `headlineMedium`/extra-bold, 1–2 строки.
1. **Счётчики** — две строки: `lessons_count` + label `lessons`, `tasks_count` + label `tasks` (число акцентно, label приглушён). Строки — из i18n (`lore_card_lessons`, `lore_card_tasks`).
1. **Кнопка «Get the lore»** — внизу слева (`labelLarge`), тап → `course_path/{course_id}`.
1. **Декор** — `card_decoration_url`, внизу справа, поверх градиента (не интерактивный).

- **Действие:** тап по карточке или кнопке «Get the lore» → `course_path/{course_id}`.
- **Состояния:** loading — skeleton-карточки; empty — «No lores yet»; ошибки изображений — placeholder (`expo-image`), не краш (см. A13).
- Все надписи — английский, через i18n.

### 12.4 Course Path (сетка уроков курса)

- **Назначение:** экран курса — **сетка карточек-уроков** (grid), каждая карточка = один `lesson` с обложкой (`cover_image_url`). Заменяет прежнюю «дорогу с нодами»: тот же экран, тот же роут `course_path/{courseId}`, тот же HUD сверху — меняется только раскладка (сетка вместо дороги). Фон экрана — цвет из `path_background_color`.
- **UiState:** `loading | content(course, lessons) | error`.
- **Данные:** `get_course_path(courseId)` → `lessons[]` со `status`, `cover_image_url`, `sort_order`.
- **Раскладка:** вертикальный скролл, сетка из карточек по `sort_order` (число колонок — фиксированное, например 2; точное значение — деталь UI-фазы). Карточка прямоугольная со скруглением `radiusLarge`, фон = обложка урока (`expo-image`, кэш/blurhash/placeholder; ошибка картинки → placeholder, не краш, A13). Подпись урока (`title`) — поверх/под обложкой по дизайну кита.

**Состояния карточки (через токены `node-*`, §5.6; без хардкода цвета):**

1. **`completed` (пройден).** Обложка в **обычном (полном) цвете**, без бейджей и значков — «прошли и прошли». Тап → можно перепройти урок (`lesson/{id}`).
1. **`active` (текущий).** Обложка в полном цвете + **пульсация** (Reanimated 3 — `withRepeat(withTiming(...))` по `scale`/свечению; токен `node-active`). Пульсирует **только одна** активная карточка. **A11y-фолбэк:** при включённом «уменьшении движения» (`useReducedMotion()`) пульс не запускается — вместо него статичная рамка/кольцо `node-active`, чтобы текущий урок отличался от пройденного и на статичном экране. Тап → `lesson/{id}`.
1. **`locked` (будущий).** Поверх обложки — **нежёсткий desaturate-overlay** (обложку по-прежнему видно; `opacity` карточки **не меняем**). Внизу справа — иконка **`Lock`** (Lucide), наложена **прямо на слой обложки** (без кружка, без обводки, без подложки; цвет — токен `node-locked`/`muted`). **Неинтерактивна:** тап ничего не делает (не ведёт в урок).

- **Действия:** тап `active` → `lesson/{id}`; тап `completed` → `lesson/{id}` (перепрохождение); тап `locked` → нет реакции.
- **Состояния экрана:** loading — skeleton-сетка; empty — «No lessons yet»; error — ретрай. Ошибки изображений — placeholder (`expo-image`), не краш (A13).
- Все надписи — английский, через i18n.

### 12.5 Lesson (раннер заданий)

- **Назначение:** последовательное прохождение заданий; нижний HUD скрыт; вверху — **панель урока** (прогресс-бар + кнопка выхода), см. 12.5в. Экран открыт как `fullScreenModal`.
- **UiState:** `loading | running(task, index, total) | finished | exitConfirm`.
- **Нижняя кнопка задания** (овальная, см. 6.5а): `requires_check=true` → сначала **disabled с подписью-действием**, после действия — активная **«Done»** (нажатие запускает проверку). `requires_check=false` → **«Continue»** (для `reading_text` — только на последнем сегменте, для `reading_media`/`video` — сразу; см. 6.5б). Автопроверки нет. Кнопка не перекрывает текст (§6.5в).
- **Логика батарейки:** видео и чтение — бесплатно; тест (`requires_check=true`) — перед началом проверка `battery>0` (у не-подписчика), иначе → `paywall`. **Любой ответ** → `submit_task_attempt` → `battery -= 1` (сервер) → следующее задание. Серия верных подряд в курсе на порогах (5→+2, 10→+5 по дефолту) добавляет заряд; неверный ответ обнуляет серию. У подписчика батарея не тратится.
- **Видео-задание:** воспроизводится через **общий плеер** (`features/player`, `expo-video`) с тап-управлением — тап по центру = пауза/play, перемотка перетаскиванием по прогресс-полосе (Gesture Handler; см. F6). «Просмотрено» при ~95% длительности или ручном Continue → `submit_task_attempt(is_correct=true)`.
- **Завершение:** прогресс-бар дошёл до конца → `complete_lesson` → `reward/{lessonId}` с результатом.

#### 12.5в Верхняя панель урока: прогресс-бар и выход

> Три элемента поверх раннера. Все надписи — английский, через i18n.

**1. Прогресс-бар урока (сверху).** Горизонтальная полоса, заполняется **по мере прохождения заданий**: `доля = пройдено / всего`. Каждое завершённое задание (Continue/Done) сдвигает полосу. Полоса считает **все** задания, включая видео.

- **Поведение на полноэкранном видео.** Панель (прогресс-бар + крестик) **скрыта во время воспроизведения** (иммерсивный режим). При **тапе по видео** (пауза) — панель появляется; при возобновлении — скрывается. Прогресс-счётчик не останавливается.

**2. Кнопка выхода (крестик, справа вверху).** Иконка `✕`. Тап → **немедленно останавливает любой звук** (TTS текущего сегмента или аудио видео) → открывается подтверждающая плашка (`exitConfirm`). Урок не прерывается — только звук на паузе.

**3. Подтверждающая плашка выхода (bottom sheet).** Выезжает снизу (`@gorhom/bottom-sheet`), контент затемняется. Звук на паузе, пока плашка открыта:

- **Заголовок:** `Are you sure?`
- **Текст:** `If you exit the lesson now, your progress will reset.`
- **Кнопки (сверху вниз):**
  - **`Exit lesson`** — деструктивная (красная, токен `error`, пилюля): выход подтверждён, звук прекращается. **Прогресс по уроку сбрасывается** — текущий прогон отбрасывается, при следующем заходе урок с **первого задания**. Навигация → назад на `course_path`.
  - **`Cancel`** — текстовая: закрывает плашку, **звук возобновляется**, возврат к текущему заданию.

> **Что значит «сброс прогресса».** Завершение урока атомарно (`complete_lesson` срабатывает только когда пройдены все задания, см. A7). Выход в середине: `user_lesson_progress` остаётся `active`, клиент отбрасывает прогон, следующий вход — с задания №1. Серверная мутация не нужна. **Важно:** батарея, списанная за ответы в этом проходе, **не возвращается** (списание уже зафиксировано на сервере).

#### 12.5а Задание `reading_text` (чтение текста) — детально

- **Назначение:** показ текста (без медиа) с опц. TTS-озвучкой; **без проверки** (`requires_check=false`), кнопка **«Continue»**.
- **Раскладка (сверху вниз):**

1. **Заголовок текста** — `payload.title`.
1. **Индикатор разделов** — продолговатые буллеты под заголовком (текущий из N). Скрыты, если раздел один.
1. **Текст текущего раздела** — акцентный белый; предыдущие сереют.
1. **Кнопка «Continue»** внизу — **появляется не сразу**.

- **Появление «Continue»:** изначально кнопки нет. Виден первый сегмент. **Свайп снизу вверх** (Gesture Handler + Reanimated) открывает следующий сегмент. **«Continue» появляется только на последнем сегменте.** Один сегмент → доступна сразу.
- **Аудио и автостарт:** при переключении на задание озвучка текущего раздела (если `audio_url != null`) **стартует сразу** (`expo-audio`/`expo-av`). Если аудио нет — пользователь читает сам.
- **Переключение разделов:** свайп вверх → мягкий скролл к следующему; новый раздел акцентный, прежний сереет; прежнее аудио **останавливается**, новое **стартует**.
- **Сегменты** — из `payload.segments[]`. Один сегмент = цельный текст без буллетов.
- **Завершение:** «Continue» (на последнем сегменте) → следующее задание.
- **Деградация:** битый `audio_url` → раздел как чисто текстовый (без краша); см. A13.
- Все надписи — английский, i18n.

#### 12.5б Задание `reading_media` (иллюстрированное чтение) — детально

- **Назначение:** текст с опц. TTS, **без проверки**, кнопка **«Continue»**, **плюс одно медиа сверху**. Картинка с текстом и видео с текстом — один тип (`reading_media`), отличаются `media.kind`.
- **Раскладка (сверху вниз):**

1. **Медиа** — верхняя ~половина экрана. Картинка через `expo-image` (`media.image_url`); в видео-режиме (позже) — общий плеер (`features/player`).
1. **Нижняя панель** (`surface`, скруглённая сверху): **заголовок** → **текст разделов** (как в reading_text) → кнопка **«Continue»**.

- **Кнопка «Continue» — всегда активна сразу** (в отличие от reading_text).
- **Текст не перекрывается кнопкой (§6.5в):** кнопка зафиксирована снизу (абсолютная, над safe-area) и не наезжает на текст. Длинный текст → область скроллится (`ScrollView`), текст уходит **под** кнопку; у кнопки **fade-out градиент** (`expo-linear-gradient`).
- **Аудио, разделы, деградация текста** — как в `reading_text` (12.5а), **кроме** появления кнопки (здесь сразу).
- **Деградация медиа:** битая картинка → placeholder на месте медиа-области, экран не падает (A13).
- **Рендерер:** `features/lesson/tasks/ReadingMediaTask` (переиспользует логику чтения из `ReadingTextTask`).
- Все надписи — английский, i18n.

### 12.6 Reward

- **Назначение:** случайная Rive-анимация маскота + текст похвалы + (если есть) начисленные алмазы + разблокировка следующей ноды. Открыт как `modal`.
- **Текущее поведение анимации:** маскот играет случайную анимацию из **единого нейтрального пула** (без деления по качеству прохождения). **Урок как таковой не «провален/пройден»** (см. A7).
- **Реакция маскота на качество заданий — управляется флагом из админки (`app_config.economy.mascot_tone_reaction_enabled`, дефолт `false`; B6.8).** Когда флаг `true` **и** в установленной сборке есть анимации обоих пулов: тон выбирается по числу неверных ответов (`errors_count`) — пул **celebrate** (0/мало неверных) или **encourage** (больше неверных), по 10 анимаций в каждом пуле, выбор случайный. Когда флаг `false` **или** анимации пулов отсутствуют в сборке → играет нейтральный пул (**мягкий откат**, без краша). Сейчас флаг выключен (анимации ещё не нарисованы). Счётчик `errors_count` пишется на сервере независимо от флага (§A6/A7), поэтому фичу можно включить тумблером, как только нужная сборка с анимациями доедет до пользователей — без правок бэкенда.
- **Действие:** «Continue» → назад на `course_path` (следующая нода теперь `active`).

### 12.7 Quests (вкладка 2, блок F) — *MVP, обязателен*

- **Назначение:** задания Daily / Monthly / Exclusive → награда алмазами + **встроенный магазин** заморозки стрика.
- **UiState:** `loading | content(daily, monthly, exclusive, shop) | error`.
- **Данные:** `get_quests`.
- **Действия:** забрать награду → `claim_quest_reward(quest_id)` (идемпотентно); покупка заморозки → `buy_streak_freeze(days)` (1/3/… дней); недостаток алмазов → понятное сообщение.

### 12.8 Subscription (вкладка 3, блок G)

- **Назначение:** карточки Super / Super Family (скруглённые блоки) + CTA подписки. Текст CTA и наличие бесплатной недели зависят от `get_subscription_offer` (§A14).
- **Данные:** локальная цена и пакеты — из **RevenueCat `Offerings`/`Packages`** (стор сам знает страну/валюту); тексты — из `price_tiers` для отображения; **предложение** (`trial_available`, `timer_*`) — из `get_subscription_offer` (server-authoritative).
- **Условный CTA по предложению:**
  - `trial_available=true` → кнопка **«Try 1 week for $0»** (free-trial offer; на iOS — introductory offer, на Android — free-trial offer).
  - `trial_available=false` → кнопка обычной покупки (например **«Subscribe»** с ценой).
- **Таймер обратного отсчёта (если `timer_active=true`):** над карточками — **персональный счётчик** «Special price ends in 23:59:58», тикающий до `timer_ends_at` (тикает локально, но истина — `timer_ends_at` с сервера). Персональное окно на юзера. При `timer_expired=true` → счётчик исчезает, триал больше не предлагается.
- **Действие:** покупка → **RevenueCat `purchasePackage()`** → вебхук `revenuecat-webhook` обновляет `subscriptions` → (опц.) `validate-purchase` для немедленной реконсиляции. **Restore Purchases** — `restorePurchases()` (обязательно для обеих платформ).
- **Эффект подписки:** при `trial`/`active` батарея перестаёт тратиться, в HUD на месте батареи — ассет `app_config.subscription_badge` (§11, §A4). При отмене/истечении — батарея возвращается полной.

### 12.9 Paywall

- **Триггер:** `battery=0` при попытке теста, или из Subscription. Открыт как `modal`.
- **Контент:** зависит от `get_subscription_offer` — при `trial_available=true` «Buy subscription. First week — $0»; иначе обычная покупка. При `timer_active=true` — тот же персональный обратный отсчёт. Покупка — через RevenueCat (как 12.8).

### 12.10 Shop (секция внутри Quests, бывший блок J)

- **Не отдельный экран** — секция/блок `SHOP` внутри вкладки Quests (см. 12.7).
- Покупка заморозки стрика за алмазы (1/3/… дней) → `buy_streak_freeze(days)`. Это **внутренняя валюта (алмазы)**, не IAP — стор-биллинг не задействован.

### 12.11 Profile (вкладка 4, блок H)

- Дубль HUD (level, streak, diamonds, battery); ачивки (костюмы/медали); шестерёнка ⚙️ вверху справа → `settings`.

### 12.12 Settings и подразделы

> Полная структура — concept 1.18а. Реализуется как список → подэкраны.

- **Preferences:** Sound Effects / Vibration (Haptics) / Daily Practice Reminders (тумблеры, хранятся в MMKV; reminder включает локальное уведомление `expo-notifications`). Vibration → `expo-haptics` (кросс-платформенно).
- **Profile:** Name / Username / Password / Email (редактирование) + кнопка **DELETE ACCOUNT** (→ `delete_account`; обязательно для Apple и Google).
- **Course:** список начатых/завершённых курсов + удалить (→ `reset_course_progress`, с подтверждением).
- **Account Linking:** тумблер Google + Apple (iOS) + статус Email; отвязка соц-входа требует предварительной привязки email (диалог). Нельзя отвязать единственный способ входа.
- **Documents:** Terms / Privacy Policy (in-app, `react-native-webview` или нативный текст).
- **Support:** Help Center / Send Feedback (in-app). Кнопки **Choose / Restore Subscription** (RevenueCat), **LOG OUT**. На iOS — ссылка «Manage Subscription» ведёт в системные настройки App Store; на Android — в Play подписки.

-----

## 13. Спецификации фич с критериями приёмки

> Сжатый формат: **AC** = acceptance criteria. Тестовые требования вынесены в отдельный документ.

**F1. Авторизация.** AC: вход через Google создаёт профиль с 300 алмазами; **на iOS доступен «Sign in with Apple»**; сессия сохраняется (шифрованно, secure-store) и переживает перезапуск на обеих платформах; logout очищает сессию; email-регистрация требует код подтверждения.

**F2. Батарейка.** AC: тест при `battery=0` (у не-подписчика) блокируется и открывает paywall; **любой ответ (верный/неверный) уменьшает батарею на 1 на сервере**; **за завершение урока начисляется фикс `battery_reward_per_lesson` (дефолт +3), не выше `battery_max`, идемпотентно**; видео и чтение не тратят батарею; восстановление по конфигу, не выше `battery_max`; HUD красит иконку красным при ≤3; **у подписчика батарея не тратится и в HUD заменена ярким ассетом подписки**, после отмены — батарея возвращается полной. Все числа настраиваются в админке. Поведение идентично на iOS и Android.

**F3. Алмазы.** AC: завершение **любого** урока начисляет случайную сумму по весам (`10`/`15`/`20`, 10 — чаще всего) ровно один раз на урок (идемпотентно по леджеру); сумма разыгрывается на сервере; баланс меняется только через RPC; покупка заморозки (150 алмазов/день, из конфига) проверяет достаточность.

**F4. Стрик.** AC: ≥1 урок в день увеличивает стрик; пропуск дня сбрасывает, если нет заморозки; покупка/наличие заморозки прощает день; день засчитывается по `last_activity_date < today`.

**F5. Прогресс/разблокировка.** AC: следующий урок разблокируется только после `complete_lesson` на сервере; клиент не может разблокировать локально; прогресс-бар курса = `lessons_completed / total`.

**F6. Видео.** AC: видео стартует на низком качестве почти мгновенно (HLS, `expo-video`); следующий ролик прелоадится; показывается постер вместо чёрного экрана; повтор играет из кэша. **Управление (общий плеер):** тап по центру = пауза/play (иконка ▶️ при паузе); перемотка перетаскиванием по прогресс-полосе; (опц.) двойной тап = ±10 сек. Работает одинаково на AVPlayer (iOS) и ExoPlayer (Android).

**F7. Маскот-награда.** AC: после урока показывается случайная анимация маскота, каждый раз потенциально разная (`rive-react-native`); текст похвалы; показ начисленных алмазов; **урок не помечается как успешный/неуспешный**. **Реакция по качеству заданий (celebrate/encourage по `errors_count`) управляется флагом `app_config.economy.mascot_tone_reaction_enabled` (дефолт `false`, тумблер в админке B6.8):** при `true` и наличии анимаций обоих пулов в сборке — тон по `errors_count`; при `false` или отсутствии анимаций — нейтральный пул (мягкий откат, без краша). `errors_count` пишется независимо от флага (§A7). Сейчас флаг выключен (анимации не нарисованы).

**F8. Подписка.** AC: покупка проходит через **RevenueCat** на **обеих платформах** (Google Play Billing + Apple StoreKit); статус приходит вебхуком в `subscriptions` и подтверждается `validate-purchase`; **Restore Purchases** работает; локальная цена соответствует стране пользователя (стор определяет автоматически через RevenueCat Offerings). **Предложение управляется `get_subscription_offer` (§A14):** при включённом триале CTA даёт бесплатную неделю (Android free-trial offer / iOS introductory offer), при выключенном — обычную покупку; персональный таймер (если включён) показывает обратный отсчёт и по истечении убирает бесплатную неделю для этого юзера; триал и таймер включаются независимо из админки.

**F9. Server-driven UI.** AC: дефолтный конфиг встроен (`default_ui_config.json`) и работает офлайн-старте; активный конфиг применяется по `min_app_version` (build-номер, см. A11); отключённый блок/вкладка не отображается; неизвестный тип блока не роняет приложение.

**F10. Региональные цены.** AC: пользователь видит цену своей страны (стор определяет автоматически через RevenueCat); тексты-заглушки до загрузки берутся из `price_tiers`/`country_price_map`.

**F11. Настройки.** AC: все тумблеры сохраняются (MMKV); DELETE ACCOUNT удаляет данные и аккаунт (Apple + Google требование); Course-сброс удаляет прогресс; Account Linking не даёт отвязать единственный способ входа; управление подпиской ведёт в системные настройки соответствующего стора.

**F12. Виджеты.** AC: виджеты показывают напоминание/стрик и открывают приложение по тапу (deep link) — **на Android через Glance, на iOS через WidgetKit** (≥2 размера на каждой платформе). Данные виджету поставляются единым мостом из приложения (shared storage / App Group на iOS).

**F13. Аналитика/краши.** AC: ключевые события логируются (lesson_completed, paywall_shown, subscription_started, streak_lost) на обеих платформах (Firebase Analytics); краши уходят в Crashlytics; **пойманные клиентские ошибки** (сетевые сбои, RPC-ошибки, ошибки покупок) репортятся через `crashlytics().recordError(e)`; **серверные ошибки** (Edge Functions, RPC) пишутся в таблицу `error_logs`.

**F14. Quests + Магазин (блок F).** AC: вкладка Quests показывает задания daily/monthly/exclusive со статусом; награда за выполненный квест начисляется один раз (идемпотентно по леджеру `quest_reward`); встроенный магазин позволяет купить заморозку (1/3/… дней) через `buy_streak_freeze`; недостаток алмазов даёт понятную ошибку, баланс не уходит в минус; отдельной вкладки/маршрута `shop` нет.

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

-----

## 15. Нефункциональные требования

- **Скорость видео — приоритет №1.** Старт ролика и переходы — мгновенные (HLS + прелоад + кэш + постер), одинаково на iOS/Android.
- **Холодный старт** ≤ 2–3 c до Home (с кэшем). Hermes v1 (New Architecture) — дефолтный движок; следить за размером JS-бандла.
- **Офлайн:** режима нет; при отсутствии сети на старте — заглушка с текстом и Retry; кэш (TanStack Query persist / SQLite) — для ускорения, не как офлайн-функция.
- **Безопасность:** RLS на всех таблицах; критичные мутации только через RPC; секреты только на сервере/в RevenueCat; токены сессии — в secure-store.
- **Темы:** light и dark на обеих платформах (`useColorScheme`).
- **Локализация:** все строки в i18n-ресурсах; структура готова к доп. языкам; `expo-localization` для определения локали.
- **Доступность:** RN accessibility-пропсы (`accessibilityLabel`/`role`/`accessible`); размеры тап-целей ≥ **44pt (iOS HIG)** / **48dp (Android)**; контраст по токенам; поддержка системного масштаба шрифта (с разумными лимитами).
- **Возраст:** 13+ (COPPA не применяется; App Store rating и Google content rating выставить соответствующе); без детского режима.
- **Платформенные минимумы:** iOS 15.1+ (Expo SDK 55) / 16+ (SDK 56); Android minSdk 24, target — последний требуемый сторами.

-----

## 16. Обработка ошибок и edge cases

- **Battery empty** при тесте → paywall, не краш.
- **Insufficient diamonds** при покупке заморозки → понятное сообщение.
- **Нет сети** → заглушки/ретраи (TanStack Query retry); кэш где возможно.
- **Конфиг новее приложения** → неизвестные блоки/иконки пропускаются.
- **Двойное завершение урока** → идемпотентно (награда один раз).
- **Отвязка единственного способа входа** → запрещена (диалог).
- **Протухшая сессия** → тихий refresh (supabase-js auto-refresh); при неудаче → экран логина.
- **Покупка/валидация не прошла** → подписка не активируется, сообщение пользователю; статус сверяется с RevenueCat/вебхуком.
- **Покупка прервана/отменена пользователем** (`userCancelled` из RevenueCat) → молча возвращаемся, без ошибки.
- **Платформенные крэши при отсутствии нативного модуля** (напр. Rive/виджет на старой ОС) → graceful degradation, не краш.

-----

## 17. Вне зоны MVP (ссылка)

Офлайн-режим, FCM/APNs пуши, RTDN/расширенный server billing, Cloudflare R2 — см. concept 1.24. Заводятся после релиза MVP. **iOS — в MVP** (целевая платформа наравне с Android, на едином RN-коде). **Блок F (Quests с магазином) — в MVP** (Phase 11). CI/CD реализуется через EAS уже на этапе разработки (не откладывается).

-----

*Спек синхронизирован с `concept.md`. При изменении продуктовых решений — сначала правится concept.md, затем этот спек. Тестовые требования — в отдельном тест-документе.*

-----

# ПРИЛОЖЕНИЕ A — Закрытие пробелов реализации (RN + iOS)

> Добавлено по итогам инженерной ревизии. Это места, где разработка через Claude Code встала бы из-за нехватки данных. Каждый пункт закрыт принятым решением (помечено **[РЕШЕНИЕ]**). При желании заказчик может пересмотреть — но дефолт задан, чтобы не блокировать работу. В RN-версии пункты по версиям, секретам, подписанию и биллингу переписаны под Expo / EAS / RevenueCat и **обе платформы**.

## A1. Версии зависимостей (Phase 0 встаёт без этого)

> Проблема: «последний стабильный» не воспроизводим — через месяц это другие версии, и ИИ сгенерирует несовместимый набор. Нужен зафиксированный baseline. В Expo это особенно важно: версии нативных модулей должны соответствовать SDK.

**[РЕШЕНИЕ]** Зафиксировать через **Expo SDK** (он сам пинит совместимые версии RN и большинства expo-* модулей). Стартовый baseline (Claude Code обновит до совместимых актуальных при старте через `npx expo install --fix`, но эти — известно-рабочие на июнь 2026):

```jsonc
// package.json (фрагмент, версии управляются Expo SDK)
{
  "dependencies": {
    "expo": "~55.0.0",                       // baseline SDK; цель апгрейда — 56
    "react": "19.1.0",
    "react-native": "0.83.x",                // пинится Expo SDK 55
    "@react-navigation/native": "^7.x",
    "@react-navigation/native-stack": "^7.x",
    "@react-navigation/bottom-tabs": "^7.x",
    "@tanstack/react-query": "^5.x",
    "zustand": "^5.x",
    "@supabase/supabase-js": "^2.x",
    "expo-image": "~2.x",                     // замена Coil
    "expo-video": "~2.x",                     // замена Media3/ExoPlayer (HLS)
    "rive-react-native": "^9.x",              // маскот (замена Rive Android)
    "react-native-reanimated": "~4.x",        // анимации/жесты
    "react-native-gesture-handler": "~2.x",
    "expo-sqlite": "~16.x",                   // замена Room (кэш)
    "react-native-mmkv": "^3.x",              // замена DataStore (key-value)
    "expo-secure-store": "~15.x",             // токены/сессия
    "i18next": "^24.x",
    "react-i18next": "^15.x",
    "expo-localization": "~17.x",
    "expo-haptics": "~15.x",
    "expo-linear-gradient": "~15.x",
    "@gorhom/bottom-sheet": "^5.x",
    "react-native-purchases": "^8.x",         // RevenueCat (Play + StoreKit)
    "expo-apple-authentication": "~8.x",      // Sign in with Apple
    "@react-native-google-signin/google-signin": "^14.x",
    "expo-application": "~7.x",               // nativeBuildVersion для min_app_version
    "expo-linking": "~8.x",
    "zod": "^3.x",
    "@react-native-firebase/app": "^21.x",    // Crashlytics/Analytics
    "@react-native-firebase/crashlytics": "^21.x",
    "@react-native-firebase/analytics": "^21.x"
  }
}
```

- **Node 20 LTS+**, пакетный менеджер — `npm` (или `pnpm`), lockfile коммитится.
- **New Architecture включена** (обязательно для SDK 55+).
- Платформенные таргеты: **iOS deployment target 15.1** (SDK 55) / 16.0 (при апгрейде на 56); **Android `minSdkVersion` 24, `targetSdkVersion`/`compileSdkVersion` — по Expo SDK** (35+).
- **Правило:** не обновлять мажорные версии (SDK, RN, Reanimated, Navigation) в середине фазы; апгрейд SDK — отдельной задачей с прогоном сборки **на обеих платформах** (`eas build -p ios` и `-p android`).
- Версии нативных модулей не пинить вручную — использовать `npx expo install <pkg>`, чтобы Expo подобрал совместимую с SDK.

## A2. Управление секретами и конфигурация сборки (Phase 0/2)

> Проблема: где живут Supabase URL/anon-key, Google Web Client ID, RevenueCat ключи, Cloudflare-домен. Без этого не собрать ни auth, ни сеть, ни покупки — причём на двух платформах сразу.

**[РЕШЕНИЕ]**

- Несекретные клиентские значения (Supabase URL, Supabase **anon** key, Google Web/iOS Client ID, RevenueCat **public** SDK keys для Apple и Google, Cloudflare-домен) → переменные окружения `EXPO_PUBLIC_*` в `.env` (НЕ в git), читаются в `app.config.ts` и пробрасываются в `expo-constants` / `extra`. `anon`-ключ и RevenueCat public keys публичны по дизайну, но всё равно не коммитим.
- `.env.example` — шаблон с пустыми ключами, коммитится.
- `app.config.ts` (динамический конфиг вместо статического `app.json`) задаёт `ios.bundleIdentifier = com.lorebinge.app`, `android.package = com.lorebinge.app`, `scheme = "lorebinge"`, associated domains (Universal Links) и intent-filters (App Links), плагины (`expo-apple-authentication`, `react-native-purchases`, firebase, и т.д.).
- Серверные секреты (RevenueCat **secret** API key, RevenueCat webhook authorization header, Google Play service account JSON, App Store Connect API key) → **только** в Supabase Edge Functions secrets (`supabase secrets set`). Никогда в бандле приложения.
- **Нативные конфиг-файлы платформ (НЕ в git, по одному на профиль сборки):** `google-services.json` (Android/Firebase) и `GoogleService-Info.plist` (iOS/Firebase). Подкладываются через EAS file-based secrets или `app.config.ts` → `googleServicesFile`.
- Профили сборки в `eas.json`: `development` (dev client), `preview` (internal distribution / TestFlight + internal track), `production` (store). Разные `EXPO_PUBLIC_*` через EAS environment variables; на старте — один Supabase-проект, структуру под dev/prod заложить с Phase 0.
- `.gitignore`: `.env`, `.env.*` (кроме `.env.example`), `google-services.json`, `GoogleService-Info.plist`, `*.keystore`, `*.p8`, `*.mobileprovision`, `/supabase/.env`, `ios/`, `android/` (если используется CNG — Continuous Native Generation через `expo prebuild`; нативные папки регенерируются).

## A3. Подписание приложения (Phase 16, но ключи/сертификаты завести заранее)

> Проблема: потеря Android keystore = невозможность обновлять в Play навсегда; неверная настройка iOS-сертификатов/provisioning блокирует сборку под App Store.

**[РЕШЕНИЕ]** Подписание обеих платформ управляется **EAS Build** (managed credentials), что снимает большую часть ручной работы.

- **Android:** включить **Google Play App Signing** (Google хранит ключ подписи приложения). Upload keystore генерирует и хранит EAS (managed) либо создаётся локально и загружается в EAS; в любом случае — забэкапить в надёжное место (не в git). Release-артефакт — **`.aab`** (Android App Bundle).
- **iOS:** Distribution Certificate и Provisioning Profile (App Store) управляются EAS (`eas credentials`), привязка к Apple Developer account. Release-артефакт — **`.ipa`**. Для Sign in with Apple и Push (post-MVP) — корректные capabilities в provisioning.
- **Submit:** `eas submit -p ios` (в App Store Connect / TestFlight) и `eas submit -p android` (в Play Console). App Store Connect API key и Play service account — в секретах EAS/Supabase, не в git.
- Сертификаты/ключи завести в Phase 0/1 (как минимум аккаунты Apple Developer и Google Play Console), даже если релиз — в Phase 16.

## A4. Точная механика батарейки и восстановления (Phase 8 — иначе логика неоднозначна)

> Проблемы, на которых RPC встанет: тратит ли батарею КАЖДЫЙ ответ; восстановление по «тикам» или по факту захода; что со стартовым значением; что показывать подписчику. **Логика серверная — от платформы не зависит.**

**[РЕШЕНИЕ]**

- **Единица траты:** −`battery_cost_per_answer` (=1) за **каждый ответ** в задании-тесте (`requires_check=true`) — **и верный, и неверный**. Задания без проверки (`video`, `reading_text`, `reading_media`) батарею не тратят.
- **Бонус за завершение урока:**
  - При `complete_lesson` начисляется фикс `battery_reward_per_lesson` (дефолт **+3**, настраивается в админке): `battery = min(battery_max, battery + battery_reward_per_lesson)`. Внутриурочных бонусов и серий верных ответов **нет** — экономика плоская: трата за каждый ответ, разовый бонус в конце урока.
  - Смысл: дойдя до конца урока, пользователь получает запас ещё примерно на `battery_reward_per_lesson` тест-заданий — «отличнику» чуть свободнее, но не хватит на целый второй урок (paywall остаётся близко).
  - **Идемпотентно:** бонус привязан к завершению конкретного урока (тот же `ref_id=lesson_id`, что и алмазы); повторное/перепрохождение урока заряд не доначисляет.
  - Бонус **капится `battery_max`** (избыток сгорает) и **не выдаётся подписчику** (у него заряд бесконечен).
- **Стартовое значение:** `battery = battery_max = 10` при регистрации (полная) — ровно на один урок из ~10 тест-заданий. Второго урока подряд бесплатно не хватает (это и есть точка монетизации).
- **Восстановление — “ленивое” (lazy), по факту обращения, не cron:**
  - `refill_battery` вызывается при: открытии приложения (cold start), открытии Home, попытке начать тест.
  - Формула: `ticks = floor((now − last_battery_refill_at) / battery_refill_minutes)`; `new_battery = min(battery_max, battery + ticks * battery_refill_amount)`; если `ticks > 0` → `last_battery_refill_at += ticks * battery_refill_minutes` (не `now`, чтобы не терять остаток времени).
  - `battery_refill_minutes = 60` → один заряд каждый час → **полные 10 зарядов за 10 часов**; за сутки ≈ 24 заряда ≈ **2 урока бесплатно в день** (сознательный выбор экономики).
  - Если уже `battery_max` — таймер не двигаем (или сдвигаем на now, чтобы отсчёт пошёл от момента первой траты).
- **Подписчики (ключевой смысл монетизации):**
  - При активной подписке (`trial`/`active`) ответы **не тратят батарею**, `BATTERY_EMPTY`/paywall не возникают (заряд бесконечен).
  - В HUD **иконка батареи заменяется ярким ассетом** из `app_config.subscription_badge` (`asset_url`/`asset_type`, опц. `label`). Счётчик заряда не показывается.
  - При **отмене/истечении** подписки (статус → `expired`/`cancelled`/`none`) батарея **возвращается**: HUD снова показывает батарею. На момент возврата `battery` устанавливается в `battery_max` (полная), `last_battery_refill_at = now()`.
- **Клок:** все расчёты по серверному времени (`now()` в Postgres), не по времени телефона (защита от перевода часов; одинаково на iOS и Android).

## A5. Точная механика стрика (Phase 8 — граничные случаи)

> Проблемы: что значит «пропустил день» при разных часовых поясах; авто- или ручная заморозка; считается ли сегодня, если уже заходил. **Логика серверная.**

**[РЕШЕНИЕ]**

- **Часовой пояс стрика:** фиксируем по серверному UTC-дню на старте (проще и предсказуемо). Документировать как известное упрощение; локальный TZ — улучшение на будущее.
- **Засчитывание дня:** при `complete_lesson` вызывается `touch_streak`. Если `last_activity_date = today` → стрик не меняется. Если `last_activity_date = yesterday` → `current_streak += 1`. Если `last_activity_date < yesterday` → разрыв (см. ниже). `last_activity_date = today`.
- **Разрыв и заморозка:** заморозка **автоматическая, если доступна** (`freezes_available > 0`): при заходе после пропуска система сама тратит заморозку за каждый пропущенный день (до доступного количества), сохраняя стрик; если заморозок не хватает — стрик сбрасывается в 0 (`longest` сохраняется). Покупка заморозки заранее (`buy_streak_freeze`) пополняет `freezes_available`.
- **«Прошло менее 24 часов»** трактуем как календарный день (UTC), а не скользящие 24ч.

## A6. Начисление алмазов за урок — взвешенный рандом (Phase 8)

> Алмазы даются **за завершение каждого урока, независимо от правильности ответов** (уроки не делятся на успешные/неуспешные — успешными бывают только задания внутри урока, см. A7). Сумма — случайная по весам. **Логика серверная.**

**[РЕШЕНИЕ]**

- При `complete_lesson` начисляется **случайная сумма** из `app_config.economy.lesson_diamond_rewards` — список `{amount, weight}` (дефолт: `10` вес 70, `15` вес 25, `20` вес 5 → **10 чаще всего, 15 реже, 20 реже всего**). И суммы, и веса редактируются в админке.
- **Розыгрыш — на сервере** (Postgres `random()` по нарастающей сумме весов), клиент влиять не может. EV при дефолтах ≈ 11.75 алмаза/урок.
- **Идемпотентность (награда один раз на урок):** результат фиксируется записью в `diamond_transactions` (`reason='lesson_completion_reward'`, `ref_id = lesson_id`). Повторный `complete_lesson` того же урока (в т.ч. перепрохождение) **не перекатывает кубик и не доначисляет** — возвращается ранее зафиксированная сумма.
- `diamonds_awarded` в ответе `complete_lesson` = выпавшая (или ранее зафиксированная) сумма.
- `errors_count` урока (число неверных ответов на задания) на размер награды **не влияет** — задуман как вход для тона маскота (F7). Эта реакция включается тумблером `app_config.economy.mascot_tone_reaction_enabled` (дефолт `false`, B6.8) и **сейчас выключена**; счётчик всё равно пишется на сервере, чтобы фичу можно было включить из админки без правок бэкенда.

## A7. Завершение урока: что значит «все задания пройдены» (Phase 6/8)

> Проблема: урок засчитывается, даже если на тесты ответили неверно? (Да, тест необязательно правильный.) Нужно явно.

**[РЕШЕНИЕ]**

- Урок завершается, когда пользователь **прошёл все задания по порядку до конца** (дошёл до последнего и нажал «завершить»), **независимо от правильности** ответов. Видео нужно досмотреть/проскипать до конца просмотра.
- **Уроков «успешных/неуспешных» не существует.** Успешным/неуспешным бывает только **отдельное задание-тест** (`requires_check=true` → верно/неверно). Урок не «проваливается» из-за неверных ответов: он либо завершён (все задания пройдены), либо нет. `errors_count` — это просто счётчик неверных тест-ответов в уроке (для тона маскота, F7), а не оценка урока.
- `complete_lesson` проверяет, что для всех `tasks` урока есть хотя бы одна запись в `user_task_attempts` ИЛИ задание не требует ответа (video — отметка о просмотре; `reading_text`/`reading_media` — отметка о прохождении по «Continue»). Для контентных заданий (`requires_check=false`) вводим «attempt» с `is_correct=true` при «Continue» / достижении конца.
- При `battery=0` пользователь **не может начать тест-задание** → не может дойти до конца урока с тестами → урок не завершится без батареи/подписки. Это и есть точка монетизации.

## A8. Воспроизведение видео: учёт «просмотрено» и интеграция Cloudflare (Phase 6/7)

> Проблема: как фиксируем выполнение видео-задания; какой формат ссылки от Cloudflare Stream; нужен ли подписанный URL. **HLS работает на обеих платформах нативно** (AVPlayer на iOS, ExoPlayer на Android) через `expo-video`.

**[РЕШЕНИЕ]**

- Cloudflare Stream отдаёт HLS-манифест вида `https://customer-<code>.cloudflarestream.com/<videoUID>/manifest/video.m3u8`. Храним `cloudflare_video_id` (UID) и собираем/храним `hls_url` в `media_assets`. HLS — единый кросс-платформенный формат, отдельные сборки под iOS/Android не нужны.
- **Доступ к видео:** на старте — публичные (не signed) URL. Signed URLs / allowed origins — улучшение на будущее (TODO).
- **«Просмотрено»:** видео-задание считается выполненным при достижении ~95% длительности ИЛИ ручном «Continue» после окончания (через `onPlaybackStatusUpdate`/события `expo-video`). Тогда вызывается `submit_task_attempt(task_id, is_correct=true)`.
- Постер (`poster_url`) — генерируемый Cloudflare thumbnail (`.../thumbnails/thumbnail.jpg`), задаётся как `posterSource` плеера.

## A10. Биллинг: точные product ID и связка с UI (Phase 10)

> Проблема: какие SKU/идентификаторы подписок и как отображаемые цены сопоставляются с реальными — теперь на **двух сторах сразу**. RevenueCat абстрагирует обе.

**[РЕШЕНИЕ]**

- **Идентификаторы продуктов (задать в обоих сторах одинаково по смыслу):**
  - Google Play (subscription + base plan + offer):
    - `super_monthly` (base plan `monthly`, offer `free-trial-1w`)
    - `super_yearly` (base plan `yearly`) — опц., позже
    - `super_family_monthly` (base plan `monthly`, offer `free-trial-1w`)
    - `super_family_yearly` — опц., позже
  - App Store (auto-renewable subscription product IDs, в одной subscription group):
    - `super_monthly`, `super_yearly`, `super_family_monthly`, `super_family_yearly` (те же smysловые id; бесплатная неделя = **introductory offer** типа free trial на продукте).
- **RevenueCat** связывает оба набора через **Entitlements** (например `super` и `super_family`) и **Offerings/Packages**. Приложение работает с offerings/entitlements, а не с сырыми product id → один код на обе платформы. Бесплатная неделя — free-trial offer (Play) / introductory free-trial (Apple), оба отражаются в `package.storeProduct.introPrice`.
- **Реальная цена и валюта** берутся из `storeProduct` (RevenueCat уже знает страну/стор). Таблицы `price_tiers`/`country_price_map` — только для **маркетинговых текстов** (баннеры «было/стало», заглушки до загрузки offerings). При расхождении показываем цену из стора.
- Состояние подписки в приложении читается из `subscriptions` (наполняется `validate-purchase` и вебхуком RevenueCat), не из локального кэша покупок. Поле `subscriptions.platform` = `app_store` | `google_play`; `rc_app_user_id` связывает с RevenueCat.

## A11. Загрузка/применение server-driven конфига и версия приложения (Phase 4)

> Проблема: что такое `min_app_version` в сравнении с версией сборки; что если активных конфигов несколько; когда тянется. На двух платформах номера сборок различаются — нужна единая числовая ось.

**[РЕШЕНИЕ]**

- `min_app_version` (int) сравнивается с **числовым номером нативной сборки**: `expo-application` → `nativeBuildVersion` (это `android.versionCode` на Android и `CFBundleVersion`/build number на iOS). **Соглашение:** держать `versionCode` и iOS build number **синхронными** (одно число на релиз обеих платформ), чтобы `min_app_version` работал одинаково. EAS `autoIncrement` настраивается на синхронный инкремент.
- Приложение выбирает **активный** конфиг с наибольшим `min_app_version`, который `<= nativeBuildVersion` и `is_active=true`.
- Конфиг тянется **один раз при старте** (Splash), кэшируется в **MMKV**; если сеть недоступна → берётся кэш; если кэша нет → встроенный дефолтный (бандл-ассет `default_ui_config.json`).
- Смена раскладки на лету в рамках одной версии **не происходит** — конфиг применяется на следующем холодном старте после успешной загрузки.
- Гарантия консистентности: только **один** конфиг с `is_active=true` на каждый диапазон версий (админка следит; на чтении берём топ-1 по `min_app_version`).

## A12. Definition of Done (все фазы)

> Тестовые требования вынесены в отдельный тест-документ и здесь намеренно не описаны. Ниже — только критерий готовности фазы.

**[РЕШЕНИЕ] Definition of Done фазы:**

- Код собирается на **обеих платформах** без ошибок: `eas build -p ios` и `eas build -p android` (или локально через dev client / `expo run:ios` / `expo run:android`).
- AC фазы выполнены вручную **на обоих** симуляторе iOS и эмуляторе Android.
- Нет хардкод-строк (всё через i18n `t('key')`) и хардкод-цветов/отступов (всё через токены §5).
- Нет утечек секретов в git (см. A2).
- Новые таблицы/функции под RLS (§7).
- TypeScript в strict-режиме без ошибок типов; линт проходит.
- **Ручной прогон ключевого флоу** после каждой фазы, затрагивающей пользовательский путь: Splash → Auth → Home → Course → Lesson → Reward — на iOS и Android.
- **Логирование:** на время разработки — обёртка над `console` (только `__DEV__`); в релизе debug-логи отключены. Пойманные ошибки — `crashlytics().recordError(error)` (RN-Firebase). Серверные ошибки — в таблицу `error_logs` (Phase 14), с полем `platform`.

## A13. Прочие зафиксированные мелочи

- **Название, bundleId/package зафиксированы:** `Lorebinge` / `com.lorebinge.app` (одинаково для `ios.bundleIdentifier` и `android.package`). Использовать везде консистентно с Phase 0. Scheme диплинков — `lorebinge://`.
- **Формат дат/времени в API:** все timestamptz в ISO-8601 UTC; клиент конвертирует в локальное только для отображения.
- **Идентификаторы строк локализации:** все `label_key` из ui_config обязаны существовать в i18n-ресурсах (`en` как базовый namespace); неизвестный ключ → фолбэк на сам ключ (i18next `fallback`/`returnKey`), не краш.
- **Обработка отсутствующих медиа:** если у video-задания нет валидного `hls_url` → задание помечается недоступным, пропускается с логом (не краш урока). Битая картинка → плейсхолдер через `expo-image` `onError` (не краш).
- **Пустые состояния:** для каждого блока Home и списка предусмотреть empty-state.
- **Лимит длины username/полей** — валидация на клиенте (zod) и в БД (CHECK), username 3–20 символов, уникальность.
- **Безопасные зоны и тач-таргеты:** учитывать `react-native-safe-area-context` (вырезы/Dynamic Island/жест-бар); минимальный тап-таргет 44pt (iOS) / 48dp (Android).

## A14. Бесплатный триал и персональный таймер спец-предложения (Phase 8/10)

> Две **независимые** настройки в `app_config.economy.trial_offer`, управляемые из админки (B6.x). Логика — **server-authoritative**: клиент только отображает ответ `get_subscription_offer`. Поведение одинаково на iOS и Android; различается лишь нативная реализация бесплатного периода (Play free-trial offer vs Apple introductory offer, см. A10).

**Настройки (`trial_offer`):**

- `trial_enabled` (bool) — предлагается ли бесплатная неделя вообще.
- `trial_days` (int) — длительность бесплатного периода (Play free-trial offer / Apple introductory free-trial на продукте, §A10).
- `timer_enabled` (bool) — включён ли персональный обратный отсчёт.
- `timer_duration_hours` (int) — длительность персонального окна.
- `timer_starts_on` (enum: `first_subscription_view` | `install`).

**Все 4 комбинации валидны:**

1. триал ON, таймер OFF → бесплатная неделя без срока.
1. триал ON, таймер ON → срочное предложение: «успей оформить бесплатную неделю за 23:59».
1. триал OFF, таймер ON → срочная цена без бесплатной недели.
1. оба OFF → обычная платная подписка без срочности.

**Персональный таймер (на юзера):**

- Стартует индивидуально: при первом срабатывании `timer_starts_on` сервер проставляет `subscriptions.offer_timer_started_at`.
- `timer_ends_at = offer_timer_started_at + timer_duration_hours`. Клиент рисует тикающий обратный отсчёт (ЧЧ:ММ:СС) на Subscription и Paywall.
- **Когда таймер истёк** (`now ≥ timer_ends_at`): этому юзеру бесплатный триал больше не предлагается — `get_subscription_offer` возвращает `trial_available=false`, CTA → обычная покупка. Необратимо для данного юзера.
- Если `timer_enabled=false` — таймер не стартует и не показывается.

**`get_subscription_offer` (RPC, §8.1)** — единственный источник истины для UI: `trial_available`, `trial_days`, `timer_active`, `timer_ends_at`, `timer_expired`.

**Граничные случаи:**

- Юзер уже оформлял триал (`trial_offered=true` + использован) → `trial_available=false`. Дублируем проверкой на сервере (стор и так не даст второй free trial на тот же аккаунт — ни Google, ни Apple).
- Смена `timer_duration_hours` в админке **не** перезапускает стартовавшие таймеры.
- Часы/время — `timestamptz` UTC; обратный отсчёт от серверного `timer_ends_at`, клиент не доверяет локальным часам для логики (только для отрисовки).

-----

*Конец Приложения A. Все решения — рабочие дефолты, разблокирующие разработку; заказчик может скорректировать любой пункт.*

-----

# ПРИЛОЖЕНИЕ B — Подпроект «Админ-панель Lorebinge»

> Самостоятельный подпроект в папке `/admin`. Управляет контентом, конфигурацией, экономикой и показывает бизнес-аналитику. Разрабатывается через Claude Code по фазам B-Phase 1…7 после релиза MVP-приложения (но аналитический фундамент в БД — раньше, см. B8).
> 
> **Платформо-независимость.** Админка — это **веб-приложение (React + Vite)** и от перехода мобильного клиента на React Native не меняется. Единственное отличие RN+iOS-версии: данные о платежах теперь приходят из **двух сторов** (Google Play и App Store) через **RevenueCat** (вебхук + `validate-purchase`), поэтому в `subscriptions`/`payment_events` присутствует поле `platform` (`google_play` | `app_store`), а downloads агрегируются из Play Console, App Store Connect и Firebase.
> 
> **Референс по стилю и архитектуре:** **shadcn/ui** (Radix + Tailwind) — та же дизайн-система, что и в приложении (React Native Reusables = shadcn для RN), общие токены из tweakcn (§5). Каркас собираем из shadcn-компонентов и блоков: коллапсируемый сайдбар, карточки-метрики, графики, таблицы, dark/light.

## B1. Технологический стек админки

|Слой            |Решение                                               |Примечание                                                                                              |
|----------------|------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
|Фреймворк       |**React 19 + TypeScript**                             |база shadcn/ui                                                                                          |
|Сборка          |**Vite**                                              |быстрый dev-server                                                                                      |
|Стили           |**Tailwind CSS v4**                                   |база shadcn/ui                                                                                          |
|UI-основа       |**shadcn/ui** (Radix + Tailwind)                      |копируемые компоненты, кастом под бренд Lorebinge; та же дизайн-система, что в приложении (RN Reusables)|
|Роутинг         |React Router v6                                       |                                                                                                        |
|Данные          |**Supabase JS-client**                                |через **service role key** (только на защищённом доступе!)                                              |
|Серверная логика|Supabase RPC + Edge Functions (общие с приложением)   |+ аналитические RPC/views                                                                               |
|Графики         |**ApexCharts** (react-apexcharts)                     |line/area/bar/donut (см. примечание о shadcn Charts ниже)                                               |
|Карта           |**react-simple-maps** (или jsVectorMap)               |Customers demographic (world map)                                                                       |
|Таблицы         |TanStack Table (сортировка/фильтр/пагинация)          |для recent orders, списков                                                                              |
|Формы           |React Hook Form + Zod                                 |валидация контента                                                                                      |
|Состояние       |TanStack Query (server state) + Zustand (ui state)    |кэш запросов, инвалидация                                                                               |
|Drag-and-drop   |dnd-kit                                               |переупорядочивание уроков и блоков                                                                      |
|Графики-загрузки|Recharts = **shadcn Charts** (альтернатива ApexCharts)|консистентно с дизайн-системой shadcn — см. примечание ниже                                             |
|Деплой          |Vercel / Netlify / Cloudflare Pages                   |статический хостинг SPA                                                                                 |


> **Графики — ApexCharts vs shadcn Charts.** ApexCharts оставлен как готовое решение (богатые типы из коробки). Но официальный чарт-компонент shadcn/ui построен на **Recharts** и использует те же токены палитры (§5), поэтому для максимальной консистентности дизайн-системы допустимо взять **shadcn Charts (Recharts)** основным, а ApexCharts — fallback. Выбор не блокирует фазы; решить при B-Phase 4 (аналитика).

## B2. Доступ и безопасность (КРИТИЧНО)

> Админка работает с **service role key** Supabase, который обходит RLS. Это самый опасный ключ в проекте.

**[РЕШЕНИЕ]**

- Service role key **никогда не кладётся в браузерный бандл**. Прямой доступ из SPA к Supabase с этим ключом — запрещён.
- Все привилегированные операции идут через **отдельный backend-слой админки** — Supabase Edge Functions с проверкой роли админа (или отдельный тонкий Node/Deno-сервер). Браузер ходит туда с обычным auth-токеном.
- **Роль админа:** таблица `admin_users (user_id uuid PK FK→auth.users, role text)`, роли `superadmin` / `editor` / `analyst`. RLS и Edge Functions проверяют членство.
- Вход в админку — через Supabase Auth (тот же механизм), но доступ только для `admin_users`.
- **Аудит действий:** таблица `admin_audit_log` (кто, что изменил, когда, старое/новое значение) — для всех мутаций контента и конфигов.
- **2FA** для админ-аккаунтов — заложить (Supabase MFA), включить до продакшена.

## B3. Лучшие практики (frontend) — чек-лист

> Свод того, что делает админку удобной и красивой. Claude Code следует этим правилам при генерации UI.

1. **Layout-каркас (shadcn/ui sidebar + блоки):** фиксированный коллапсируемый сайдбар слева, верхний хедер (поиск, профиль, переключатель темы, уведомления), контент в сетке карточек.
1. **Информационная иерархия:** сверху — KPI-карточки (крупные числа + тренд ±%), ниже — графики, ещё ниже — таблицы. Самое важное — над «линией сгиба».
1. **Консистентность:** единые компоненты (Card, StatCard, ChartCard, DataTable, Badge, Button) — переиспользуются, не дублируются.
1. **Dark/light режим** из коробки (shadcn — переключение через класс `.dark`), **единая бренд-палитра проекта** (тот же источник, что и приложение — см. §5; палитра из tweakcn в виде CSS-переменных, light/dark). Конкретный бренд-цвет фиксируется один раз в §5, а не хардкодится отдельно здесь.
1. **Состояния данных:** для каждого виджета — loading (skeleton), empty (заглушка), error (сообщение + retry). Никогда не «пустой белый блок».
1. **Отзывчивость:** desktop-first (админка), но не ломается на планшете.
1. **Таблицы:** серверная пагинация и сортировка (не тянуть 100k строк в браузер), фильтры, поиск, экспорт в CSV.
1. **Графики:** подписи осей, легенды, тултипы, выбор периода (месяц/год), сравнение с прошлым периодом.
1. **Обратная связь действий:** toast-уведомления на успех/ошибку, подтверждения на необратимые действия (удаление урока/курса).
1. **Доступность:** контраст, фокус, навигация с клавиатуры (Radix-примитивы shadcn дают основу).
1. **Командная палитра (Cmd+K)** — быстрый переход между разделами (приятный бонус, не MVP).

## B4. Лучшие практики (backend / данные) — чек-лист

1. **Аналитика — на стороне БД, не в браузере.** Считать агрегаты через SQL-views и RPC, а не тянуть сырые строки и складывать в JS.
1. **Материализованные views** для тяжёлой статистики (revenue by month, signups by country) с периодическим refresh — чтобы дашборд открывался мгновенно.
1. **Денежные суммы — в minor units** (центы, integer), не float. Конвертация валют — по курсу на момент транзакции (хранить курс).
1. **Все выборки — пагинированы** на сервере.
1. **Read-only где можно:** роль `analyst` видит статистику, но не может менять контент.
1. **Идемпотентность и аудит** мутаций (см. B2).
1. **Не дублировать источник истины:** реальные платежи — из событий сторов (Google Play + App Store) через **RevenueCat** (вебхук `revenuecat-webhook` / `validate-purchase`), не вводятся вручную.

## B5. Структура папки `/admin`

```
admin/
├── src/
│   ├── main.tsx
│   ├── App.tsx                  # роутер + layout
│   ├── layout/                  # Sidebar, Header, Shell (на shadcn/ui)
│   ├── lib/
│   │   ├── api.ts               # клиент к backend-слою админки (не service key!)
│   │   ├── auth.ts              # вход, проверка роли админа
│   │   └── queryClient.ts       # TanStack Query
│   ├── components/
│   │   ├── ui/                  # StatCard, ChartCard, DataTable, Badge, Button, Modal...
│   │   └── charts/              # RevenueChart, SubsChart, DemographicMap...
│   ├── features/
│   │   ├── dashboard/           # главный дашборд (KPI + графики + recent orders)
│   │   ├── analytics/           # детальная экономическая статистика
│   │   ├── content/             # CMS: courses, lessons, tasks, media
│   │   ├── layout-builder/      # server-driven UI: блоки и вкладки (dnd)
│   │   ├── courses-roads/       # управление уроками курса (порядок/удаление, обложки, цвет фона сетки)
│   │   ├── economy/             # настройки экономики (app_config)
│   │   ├── pricing/             # price_tiers / country_price_map
│   │   ├── users/               # список пользователей, демография
│   │   ├── orders/              # подписки/покупки со статусами (+ platform)
│   │   ├── errors/              # просмотр error_logs
│   │   └── settings/            # админ-аккаунты, роли, аудит
│   └── styles/
├── .env.example                 # только публичные значения
├── vite.config.ts
└── package.json
```

## B6. Разделы и экраны админки

### B6.1 Dashboard (главный)

KPI-карточки сверху (см. B7), графики (revenue, subscriptions), Recent Orders (последние подписки/покупки со статусами и платформой), Customers Demographic (карта мира), мини-виджет последних ошибок.

### B6.2 Analytics (экономическая статистика)

- **Downloads / installs** — всего, динамика (источник: Firebase / Play Console / App Store Connect, импорт или ручной ввод на старте).
- **Платящие пользователи** — всего, конверсия (paid / total).
- **Revenue** — графики помесячный и погодовой; разбивка по продуктам (Super / Family) **и по платформам** (App Store / Google Play).
- **Subscriptions** — активные/триал/отменённые; график помесячный и погодовой; churn; срез по платформе.
- **Trial → Paid (отдельный блок мониторинга, §A14):**
  - **Конверсия trial→paid** — доля триалов, ставших платными за период. Главная метрика здоровья воронки.
  - Динамика во времени (график по месяцам) + сравнение с прошлым периодом.
  - **Trial-туристы** — доля триалов, отменённых до первого списания.
  - Воронка: installs → trial started → paid (3 ступени с % перехода).
  - Срез по тому, был ли активен персональный таймер (`offer_timer_started_at` заполнен).
- **Разбивка по странам** — выручка и пользователи по `country_code`.
- Выбор периода, сравнение с прошлым периодом, экспорт CSV.

### B6.3 Customers Demographic (карта)

Мировая карта (react-simple-maps) с интенсивностью по странам (пользователи/выручка); таблица топ-стран рядом.

### B6.4 Recent Orders

Таблица последних транзакций: пользователь, продукт, **платформа** (App Store / Google Play), сумма, валюта, страна, статус (`trial` / `active` / `expired` / `cancelled` / `refunded`), дата. Фильтры и поиск.

### B6.5 Content (CMS)

CRUD: категории → курсы → уроки → задания → медиа. Загрузка изображений в Supabase Storage; привязка Cloudflare video ID; редактор `tasks.payload` по типам (формы под каждый TaskType).

**Поля карточки Lores (блок B) в форме курса:** загрузка `cover_image_url` (Grid), `illustration_url` (главный логотип карточки), `card_decoration_url` (декор внизу справа); пикеры цветов `card_gradient_start` / `card_gradient_end` и `path_background_color`; привязка `category_id` (отображается пилюлей). Счётчики `lessons`/`tasks` на карточке **не вводятся вручную** — считаются автоматически.

**Редактор задания (форма под каждый TaskType):** выбор `type`, затем **глобальный параметр проверки** — тумблер `requires_check` (с проверкой / без). При создании предзаполняется дефолтом по типу (§6.5а), редактор может изменить. Для `requires_check=true` редактор задаёт **подпись disabled-кнопки** (2–3 слова: `Type the word` и т.п.) или дефолт по типу. Затем `sort_order` и payload по типу: для `reading_text` — заголовок + редактор списка разделов (текст + загрузка аудио на раздел в Storage → `audio_url`); для `video` — привязка Cloudflare video ID; для `matching`/`fill_blank`/`image_recognition`/`multiple_choice`/`true_false`/`timeline` — соответствующие поля payload.

### B6.6 Courses & Lessons

Переупорядочивание уроков (dnd), удаление урок(ов)/всех, **загрузка/замена обложки урока** (`lessons.cover_image_url`), цвет фона сетки уроков (`path_background_color`), публикация курса (`is_published`), `popularity_score`, `sort_order`. *Раздел исторически назывался «Courses & Roads»; «дорога» заменена сеткой уроков (§12.4).*

### B6.7 Layout Builder (server-driven UI)

Визуальный конструктор: вкладки навигации и блоки внутри них; вкл/выкл, переупорядочивание (dnd), перенос блока между вкладками; привязка к `min_app_version`; запись в `ui_configs`; гарантия одного активного конфига на диапазон версий.

### B6.8 Economy

Редактор `app_config.economy`: стартовые алмазы (300), `battery_max` (10), `battery_cost_per_answer`, **`battery_reward_per_lesson`** (фикс заряда за завершение урока, дефолт +3, §A4), `battery_refill_minutes`, **`lesson_diamond_rewards`** (редактируемый список «сумма → вес» для рандомной награды за урок, дефолт 10/15/20 с весами 70/25/5, §A6), стоимость заморозки стрика (`streak_freeze_cost_per_day`, дефолт 150 алмазов/день). **Тумблер `mascot_tone_reaction_enabled`** (дефолт `off`) — включает реакцию маскота на качество заданий (пулы celebrate/encourage по `errors_count`, F7/§12.6); при включении без готовых анимаций клиент мягко откатывается на нейтральный пул. Отдельно — загрузка **ассета-замены батареи** (`app_config.subscription_badge`: `asset_url` в Storage, `asset_type` rive/lottie/svg/png, опц. `label`). Изменения версионируются/логируются.

**Блок «Trial & Offer» (`trial_offer`, §A14) — два независимых переключателя:**

- **Тумблер «Free trial» (`trial_enabled`)** + поле `trial_days`.
- **Тумблер «Countdown timer» (`timer_enabled`)** + поля `timer_duration_hours` и выбор `timer_starts_on` (`first_subscription_view` / `install`).
- Оба независимы (все 4 комбинации валидны, §A14); UI показывает предпросмотр-подсказку (напр. «Триал ON + Таймер ON → “Try 1 week for $0, ends in HH:MM”»).
- Изменения применяются немедленно к новым ответам `get_subscription_offer`; стартовавшие у юзеров таймеры не сбрасываются.

### B6.9 Pricing

Маппинг `country → tier`, отображаемые цены UI, коэффициент семейной подписки. Напоминание: реальная цена настраивается в **Play Console / App Store Connect** (и берётся из стора на устройстве через RevenueCat, B4.7).

### B6.10 Users

Список пользователей, поиск, детали (профиль, прогресс, баланс, подписка + платформа), демография.

### B6.11 Errors

Просмотр `error_logs`: фильтр по `source` (client/rpc/edge_function), по `platform` (ios/android/server), по дате, по коду; детали записи. Бейджи серьёзности.

### B6.12 Settings (админ)

Админ-аккаунты и роли (`admin_users`), просмотр `admin_audit_log`, профиль, 2FA.

## B7. KPI-карточки дашборда (метрики)

|Карточка            |Метрика                                     |Источник                          |
|--------------------|--------------------------------------------|----------------------------------|
|Total Downloads     |всего установок (обе платформы)             |Play / App Store / Firebase       |
|Active Users        |DAU/MAU                                     |analytics                         |
|Paying Users        |кол-во + конверсия %                        |`subscriptions`                   |
|MRR                 |месячный регулярный доход                   |`subscriptions` + цены            |
|Revenue (this month)|выручка за месяц + тренд %                  |агрегаты                          |
|Active Subscriptions|активные подписки + тренд                   |`subscriptions`                   |
|Trial → Paid        |конверсия из триала в платную + тренд (§A14)|`subscriptions` / `payment_events`|
|Churn Rate          |отток за период                             |`subscriptions`                   |

## B8. Данные для аналитики (БД)

> Закладывается **в Phase 1** основного проекта (вместе со схемой), наполняется по мере роста; читается админкой.

- **`payment_events`** — журнал событий сторов (trial_started, purchase, renewal, cancel, refund) из **обоих** Google Play и App Store, нормализованных **RevenueCat**: `id, user_id, platform (google_play|app_store), product_id, country_code, amount_minor int, currency, event_type, occurred_at`. Источник истины по деньгам и для воронки trial→paid (наполняется `revenuecat-webhook` / `validate-purchase`).
- **`app_installs`** (опц.) — агрегаты установок по дням/странам/платформам (импорт из Play Console / App Store Connect / Firebase).
- **SQL-views** (read-only, для админки):
  - `v_revenue_monthly` — выручка по месяцам (сумма `amount_minor` по `payment_events`).
  - `v_revenue_yearly` — по годам.
  - `v_subscriptions_monthly` — новые/активные/отменённые по месяцам.
  - `v_users_by_country` — пользователи и выручка по странам (для карты).
  - `v_recent_orders` — последние транзакции со статусами и платформой.
  - `v_conversion` — total users, paying users, конверсия, trial→paid, churn. **Plus (§A14):** trial started / trial→paid / trial-туристы, срез trial→paid по наличию активного таймера, воронка installs→trial→paid.
- Тяжёлые — как **materialized views** с refresh по расписанию (pg_cron) или по событию.
- Доступ к аналитике — через **аналитические RPC** (`SECURITY DEFINER`, проверка роли админа), не прямой доступ к таблицам из браузера.

## B9. Фазы разработки админки

> Идут после релиза MVP-приложения (Phase 16). Исключение: таблицы/views из B8 создаются в Phase 1 основного проекта, чтобы данные копились с первого дня.

### B-Phase 1 — Каркас и доступ

**Цель:** запускается, вход только для админов.
**Задачи:** Vite+React+TS+Tailwind; инициализация **shadcn/ui** (`init` + базовые компоненты) и палитры из tweakcn (§5); брендинг (палитра Lorebinge, лого); layout (сайдбар+хедер+тема); Supabase Auth-вход; `admin_users` + проверка роли; backend-слой для service-операций (B2); пустые маршруты разделов.
**AC:** не-админ не входит; layout и тёмная/светлая темы работают; навигация по пустым разделам.

### B-Phase 2 — CMS контента (B6.5, B6.6)

**Цель:** управление контентом без кода.
**Задачи:** CRUD категорий/курсов/уроков/заданий; загрузка медиа в Storage; привязка Cloudflare video ID; редактор payload по TaskType; dnd-переупорядочивание уроков, удаление, цвет фона, публикация курса; аудит изменений.
**AC:** курс создаётся целиком и появляется в приложении; порядок/удаление уроков отражаются; контент валидируется.

### B-Phase 3 — Layout Builder + Economy + Pricing (B6.7–B6.9)

**Цель:** управление конфигурацией приложения.
**Задачи:** визуальный конструктор блоков/вкладок (dnd) → `ui_configs` с `min_app_version`; редактор `app_config.economy` (включая блок «Trial & Offer» — два независимых тумблера `trial_enabled`/`timer_enabled` + параметры, §A14/B6.8); маппинг цен `price_tiers`/`country_price_map`; гарантия одного активного конфига на версию.
**AC:** изменения раскладки применяются в приложении по версии; экономика и цены редактируются и читаются приложением; тумблеры триала и таймера переключаются независимо и немедленно влияют на `get_subscription_offer`.

### B-Phase 4 — Аналитический фундамент (B8)

**Цель:** данные для дашборда.
**Задачи:** убедиться, что `payment_events` наполняется из `revenuecat-webhook`/`validate-purchase` (обе платформы); создать SQL-views и materialized views; аналитические RPC с проверкой роли; (опц.) импорт downloads из Play/App Store/Firebase.
**AC:** views возвращают корректные агрегаты на тестовых данных; RPC доступны только админам.

### B-Phase 5 — Дашборд и аналитика (B6.1–B6.4, B7)

**Цель:** визуализация бизнес-метрик.
**Задачи:** KPI-карточки (B7, включая Trial→Paid); графики revenue (месяц/год) и subscriptions (месяц/год) на ApexCharts; **блок Trial→Paid (B6.2): конверсия, динамика, trial-туристы, воронка installs→trial→paid, срез по таймеру**; Customers Demographic (карта react-simple-maps); Recent Orders (TanStack Table со статусами и платформой); выбор периода, сравнение, экспорт CSV; состояния loading/empty/error.
**AC:** дашборд показывает реальные агрегаты; графики переключают период; карта отражает страны; таблица заказов фильтруется; метрика Trial→Paid отображается с трендом.

### B-Phase 6 — Errors, Users, Settings (B6.10–B6.12)

**Цель:** наблюдаемость и администрирование.
**Задачи:** просмотр `error_logs` с фильтрами (включая `platform`); список/детали пользователей и демография; админ-аккаунты и роли; `admin_audit_log`; 2FA.
**AC:** ошибки видны и фильтруются; роли разграничивают доступ (analyst — read-only); аудит пишется; 2FA работает.

### B-Phase 7 — Деплой и полировка

**Цель:** продакшен-готовность админки.
**Задачи:** деплой на Vercel/Netlify/Cloudflare Pages; защита доступа (только админы, 2FA); проверка, что service key не утёк в бандл; финальная полировка UI.
**AC:** админка доступна по защищённому URL только админам; нет утечек ключей; все разделы работают на проде.

-----

*Конец Приложения B. Админка — отдельный веб-подпроект; реальные платежи и источник истины по деньгам — события сторов (Google Play + App Store) через RevenueCat, не ручной ввод.*

-----

# ПРИЛОЖЕНИЕ C — Подпроект «Контент-конвейер (Pipeline) Lorebinge»

> **Платформо-независимость.** Конвейер — это **офлайн-рантайм на Python** (Mac mini), который генерирует контент и пишет его в продуктовые таблицы `public` (§6). От перехода мобильного клиента на React Native он **не меняется**: контракт — схема БД §6, а не платформа клиента. Раздел приводится без изменений (тестовых требований в нём нет).

> Самостоятельный подпроект в папке `/pipeline`. Из одного промпта рождает идею курса → крючок → сценарий → **полностью собранный курс** (все поля `courses` + уроки + задания + видео + озвучка + картинки), с ручным апрувом на каждом ключевом шаге через веб-интерфейс, и импортирует результат в продуктовые таблицы `public` (§6). Работает 24/7 на Mac mini. Разрабатывается через Claude Code по фазам C-Phase 1…6.
> 
> **Приоритет схемы.** Спек (§6) — источник истины. Конвейер генерирует ровно те поля, что есть в схеме, и в той же нотации. Любое расхождение чинится в пользу §6.
> 
> **Язык.** Этот документ — русский. Контент на выходе конвейера — английский (рынок США, см. `concept.md`).

## C1. Технологический стек конвейера

|Слой               |Решение                                            |Примечание                                                                                                                                                                                                                                                                                                                                                                                                                   |
|-------------------|---------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|Оркестратор 24/7   |**Python 3.12** (APScheduler + asyncio) — воркер   |Один воркер: расписание генерации + **поллинг `pipeline.*` в Supabase**. UI апрувов — в админке, НЕ здесь                                                                                                                                                                                                                                                                                                                                                                                    |
|Состояние конвейера|**Supabase Postgres**, схема `pipeline`            |Та же БД, отдельная схема; импорт в `public` по `slug`                                                                                                                                                                                                                                                                                                                                                                       |
|Человек-в-цикле    |**Раздел админки** (Прил. B): React + shadcn/ui + Tailwind, адаптив |Апрувы в едином дизайне (веб + телефон); хостинг как у админки (Vercel/Netlify); связь с конвейером **только через Supabase**; email — уведомления                                                                                                                                                                                                                                                                                                                                                        |
|Мозг (генерация)   |**Anthropic API (Claude)**, через `pipeline/llm.py`|Строгий JSON. Провайдер абстрагирован: альтернативы OpenRouter/OpenAI закомментированы                                                                                                                                                                                                                                                                                                                                       |
|Картинки           |**Leonardo.ai API** (PAYG) → Supabase Storage      |Все картинки курса: **обложка курса** (`courses.cover_image_url`), **главная иллюстрация** (`illustration_url`), **декор карточки** (`card_decoration_url`), **обложка каждого урока** (`lessons.cover_image_url` — по теме урока, для сетки §12.4; +5–6 картинок на курс), **картинки к `reading_media`**, картинки к `image_recognition`, референсы персонажей. Провайдер абстрагирован за `pipeline/leonardo.py` — см. C1а|
|Видео              |**Kling** через API (fal.ai / официальный)         |3 клипа × 10с = 30с на ролик. **В MVP видео генерится только для `type=video`-уроков; `reading_media` — пока только картинка.** Видео для иллюстрированного чтения — после обкатки (≈3 мес)                                                                                                                                                                                                                                  |
|Озвучка            |**ElevenLabs** (text → audio)                      |По спикерам. Озвучивается **и видео-реплики, и весь текст курса** (`reading_text` + `reading_media` сегменты)                                                                                                                                                                                                                                                                                                                |
|Видео-хостинг      |**Cloudflare Stream** (как у приложения, §2)       |Финальное видео → `cloudflare_video_id` + `hls_url`                                                                                                                                                                                                                                                                                                                                                                          |
|Хранение           |**Supabase Storage** (как у приложения)            |Все картинки, аудио, PNG-референсы; URL пишутся в `courses.*_url` / `tasks.payload`                                                                                                                                                                                                                                                                                                                                          |
|Хост               |**Mac mini** + launchd (автозапуск)                |Дёшево, дома, всегда онлайн. **Без Tailscale/туннеля** — общение только через Supabase. Docker не нужен                                                                                                                                                                                                                                                                                                                                                                                 |


> **Граница секретов.** Приложение (A) и админка (B) держат ключи ИИ **только в Edge Functions** (§3, принцип 7). Конвейер (C) — отдельный офлайн-рантайм: ключи (`ANTHROPIC_API_KEY`, `KLING_API_KEY`, `ELEVENLABS_API_KEY`, `LEONARDO_API_KEY`, `SUPABASE_SERVICE_KEY`) лежат в `.env` на Mac mini, **никогда в git и никогда в APK/бандле**. Service key обходит RLS — он допустим только внутри этого процесса. Это сознательное разделение, а не нарушение §7.

> **LLM в проекте используется только в конвейере** (генерация контента на Mac mini, `pipeline/llm.py`). Никакого LLM в рантайме приложения нет — пользователь не взаимодействует с ИИ.

### C1а. Выбор сети для генерации картинок

Генератор картинок абстрагирован за `pipeline/leonardo.py` (как LLM за `llm.py`) — модель меняется одной переменной в `.env`, без правки стадий. Дефолт — **Leonardo.ai** (уже в стеке, дёшево PAYG, есть обучение консистентности персонажей). Рекомендуемые альтернативы на 2026 (раскомментировать в обёртке):

- **Leonardo.ai (дефолт).** Хорош для стилизованных/иллюстративных сцен и концепт-арта; имеет character-consistency. Один вендор на всё (обложки + персонажи + картинки текста).
- **Flux (Black Forest Labs), напр. Flux 1.1 Pro / Flux.2 Pro** — через **fal.ai** или Replicate. Лучшая фотореалистичность; **fal.ai уже подключён для Kling** → не добавляет нового вендора, один ключ на видео и картинки. Рекомендую как основную альтернативу.
- **Recraft** — если нужен векторный/брендовый стиль для мелкого декора карточки (`card_decoration_url`) и иконок.
- **Ideogram** — только если внутри картинки нужен читаемый текст (нам обычно не нужен).
- Прочее (Google Imagen, OpenAI GPT-Image, Stable Diffusion self-host) — на усмотрение; держим за тем же интерфейсом.

> Все варианты дают на выход растровый URL, который складывается в Supabase Storage и пишется в соответствующее поле БД. Конкретные промпты автор задаёт позже; задача спека — зафиксировать, что под каждую картинку **есть место в БД** и шаг генерации в конвейере.

## C2. Архитектура процесса (`/pipeline`)

```
pipeline/
├── main.py              # точка входа: scheduler + поллер Supabase (asyncio)
├── scheduler.py         # APScheduler-джобы (ежедневная генерация идей, heartbeat)
├── poller.py            # поллинг pipeline.* в Supabase: реагирует на апрувы/доработки из админки
├── stages/
│   ├── s1_ideas.py      # генерация идей (промпт P1)
│   ├── s3_scenarios.py  # генерация сценариев (P2)
│   ├── s5_final.py      # финальная сборка курса (P3)
│   ├── s7_chars.py      # референсы персонажей (Leonardo)
│   └── s7_media.py      # Kling + ElevenLabs + картинки + файлы
├── review.py            # нарезка course_json на review_items + цикл доработки (P4)
├── importer.py          # импорт course_json → public (courses/lessons/tasks/media_assets)
├── db.py                # supabase-py: CRUD по схеме pipeline + импорт в public
├── llm.py               # обёртка Claude (промпты P1/P2/P3/P4); альтернативы закомментированы
├── kling.py             # обёртка Kling/fal.ai
├── elevenlabs.py        # обёртка ElevenLabs
├── leonardo.py          # обёртка Leonardo.ai (все картинки и референсы)
├── characters.py        # поиск по aliases + сохранение новых персонажей
├── cloudflare.py        # загрузка видео в Cloudflare Stream → video_id/hls_url
├── notify.py            # email-уведомления (smtplib) — ссылка на раздел апрувера в админке
├── render.py            # рендер .md из course_json (для ревью)
└── config.py            # переменные окружения (.env)
```

Один постоянно работающий воркер объединяет расписание (APScheduler) и **поллер Supabase** через `asyncio`. Состояние между стадиями — только в схеме `pipeline` (никаких глобальных переменных): процесс может упасть и рестартовать без потери прогресса. launchd держит его живым. **UI апрувов — раздел админки** (React + shadcn/ui, Приложение B/D); Mac mini с ним напрямую не общается — только через таблицы `pipeline.*` в Supabase. **Без Tailscale и туннелей.**

**Зависимости (`pipeline/requirements.txt`):** `anthropic`, `apscheduler>=3.10`, `supabase>=2`, `httpx`, `elevenlabs`, `python-dotenv`. (FastAPI/uvicorn/jinja2 больше не нужны — UI апрувов в админке.) Опционально (для альтернативных LLM-провайдеров) — `openai`.

## C3. Схема `pipeline` (рабочее состояние конвейера)

> Отдельная схема в той же БД. RLS: **только сервисная роль** (приложение и обычные пользователи доступа не имеют). Не пересекается с `public`. Миграции — в `/supabase/migrations` вместе с остальными (Phase 1).

```sql
create schema if not exists pipeline;

-- Один «проект курса», движется по стадиям
create table pipeline.projects (
  id              uuid primary key default gen_random_uuid(),
  stage           text not null default 'idea_pool',
    -- idea_pool|idea_approved|scenario_draft|scenario_approved|final_draft
    -- |final_approved|in_production|published|rejected
  media_status    text,
    -- chars_pending|chars_approved|clips_pending|clips_approved
    -- |stitched|vo_pending|vo_approved|ready_to_mux|done
  slug            text,           -- будущий courses.slug (идемпотентность импорта)
  working_title   text,
  era             text,
  region          text,
  category        text,           -- строка; импорт резолвит в categories.id
  premise         text,
  hook_type       text,
  hook            text,
  user_note       text,           -- опц. дополнение к промпту P2, привязано к ТОПИКУ; NULL → P2 без изменений
  idea_json       jsonb,          -- стадия 1
  scenarios_json  jsonb,          -- стадия 3 (массив вариантов)
  chosen_scenario int,
  course_json     jsonb,          -- стадия 5: ПОЛНЫЙ курс по контракту C5/C6
  final_md_path   text,           -- .md для ревью (Supabase Storage)
  media_json      jsonb,          -- стадия 7: ссылки на медиа + статусы апрува
  imported_course_id uuid,        -- ← public.courses.id после импорта
  created_at      timestamptz default now(),
  updated_at      timestamptz default now()
);

-- Библиотека персонажей (глобальная, переиспользуется между курсами)
create table pipeline.characters (
  id               uuid primary key default gen_random_uuid(),
  canonical_name   text not null unique,
  aliases          text[] not null,
  is_historical    bool default true,
  description      text,                  -- character_lock
  png_urls         text[] not null,       -- 4 ссылки в Supabase Storage
  kling_subject_id text,
  created_at       timestamptz default now()
);

create table pipeline.project_characters (
  project_id   uuid references pipeline.projects(id),
  character_id uuid references pipeline.characters(id),
  primary key (project_id, character_id)
);

create table pipeline.batches (
  id          uuid primary key default gen_random_uuid(),
  project_ids uuid[] not null,
  prompt_used text,
  created_at  timestamptz default now()
);

create table pipeline.events (
  id         uuid primary key default gen_random_uuid(),
  project_id uuid references pipeline.projects(id),
  type       text not null,
  payload    jsonb,
  created_at timestamptz default now()
);

create table pipeline.seed_topics (
  id          uuid primary key default gen_random_uuid(),
  title       text not null,
  premise     text,
  hook_type   text,
  hook_note   text,
  user_note   text,                  -- опц. дополнение к промпту P2, привязано к ТОПИКУ; переносится в projects.user_note; NULL → P2 без изменений
  mode        text default 'to_scenario',  -- to_scenario | to_idea_batch
  priority    bool default true,
  status      text default 'pending',       -- pending | consumed
  created_at  timestamptz default now()
);

-- Поэлементный апрув: каждый проверяемый элемент курса/медиа — отдельная строка
create table pipeline.review_items (
  id             uuid primary key default gen_random_uuid(),
  project_id     uuid references pipeline.projects(id),
  kind           text not null,
    -- text|reading_media_text|question|matching|fill_blank|image_recognition|multiple_choice|true_false|timeline
    -- |video_prompt|video_clip|character_png|voiceover|course_meta|cover_image
    -- |illustration|card_decoration|reading_media_image
  locator        jsonb not null,   -- путь внутрь course_json
  content        jsonb,
  status         text default 'pending',  -- pending|approved|needs_revision|regenerating
  comment        text,
  revision_count int default 0,
  created_at     timestamptz default now(),
  updated_at     timestamptz default now()
);
```

**Антидубли идей:** перед стадией 1 собираем `working_title`/`premise` прошлых проектов → кладём в промпт P1 как `avoid_list`. **Антидубли персонажей:** поиск по `aliases` перед генерацией. **Поэлементный апрув:** курс переходит дальше только когда все его `review_items` в статусе `approved`.

> **«Ideas» и «topic» — разные объекты (важно не путать).**
> 
> - **Ideas (идеи)** — черновой **лонг-лист кандидатов**: короткие формулировки (слова/словосочетания), над которыми ЛЛМ предлагается подумать и развернуть их в полноценный топик. Дёшево, много, грубо. Живут в `pipeline.projects (stage=idea_pool)` как выход P1; апрув — на `/ideas`.
> - **Topic (топик)** — **полноценное название курса** для генерации: committed-единица, которая идёт в генерацию сценария (P2) и сборку курса (P3). Возникает либо вручную (`/topics` → `seed_topics`), либо когда идею апрувнули и она «доросла» до топика (`/ideas` → `project` с `working_title`).
> - **Промпт-дополнение привязывается к ТОПИКУ, не к идее** (см. ниже).

> **Опциональное дополнение к сценарному промпту (`user_note`).** К топику можно подать не только название, но и свободный текст — дополнение к промпту генерации сценариев (P2). Поле **необязательное** (одно поле `user_note`, не плодим дубли): вводится при желании на `/topics` (вместе с темой → `seed_topics.user_note`) и/или на `/ideas` в момент апрува идеи в топик (`projects.user_note`); из `seed_topics` переносится в `projects`. Если поле **непустое** — P2 подмешивает его как **отдельный блок дополнительных авторских инструкций** к базовому сценарному промпту (базовый промпт не переписывается, только дополняется). Если **пустое (NULL)** — P2 работает без изменений, как сейчас. Итоговый использованный промпт логируется (`pipeline.events` / `batches.prompt_used`) для трассировки.

## C4. Стадии конвейера (соответствие фазам и схеме)

|Стадия             |Что делает                                                                                                                                            |Страница апрува           |Пишет в                           |
|-------------------|------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------|----------------------------------|
|0 — свои темы      |ручной ввод приоритетной темы                                                                                                                         |`/topics`                 |`pipeline.seed_topics`            |
|1 — генерация идей |~20 идей строгим JSON (P1)                                                                                                                            |—                         |`pipeline.projects (idea_pool)`   |
|2 — апрув идей     |выбор идей в Web UI                                                                                                                                   |`/ideas/{batch_id}`       |`stage=idea_approved`             |
|3 — сценарии       |3 варианта с крючками (P2); если у топика задан `user_note` — он подмешивается в промпт P2, иначе P2 без изменений                                    |—                         |`scenarios_json`                  |
|4 — апрув сценария |выбор варианта / свой крючок                                                                                                                          |`/scenario/{project_id}`  |`stage=scenario_approved`         |
|5 — сборка курса   |полный `course_json` (P3) → нарезка на `review_items`                                                                                                 |—                         |`course_json`, `stage=final_draft`|
|5а — доработка     |точечная перегенерация одного элемента (P4 / Kling / Leonardo / ElevenLabs)                                                                           |—                         |`review_items`                    |
|6 — апрув курса    |поэлементный апрув всего курса                                                                                                                        |`/final/{project_id}`     |`stage=final_approved`            |
|7.0 — персонажи    |поиск по aliases / Leonardo → 4 PNG                                                                                                                   |`/chars/{project_id}`     |`pipeline.characters`             |
|7.1 — Kling Subject|загрузка PNG → `kling_subject_id`                                                                                                                     |—                         |`pipeline.characters`             |
|7.2 — клипы        |3 клипа × 10с на видео-урок                                                                                                                           |—                         |`media_json`                      |
|7.3 — апрув клипов |видеоплеер на каждый клип                                                                                                                             |`/clips/{project_id}`     |`media_status=clips_approved`     |
|7.4 — склейка      |ffmpeg concat (авто) или папка                                                                                                                        |—                         |`media_status=stitched`           |
|7.5 — озвучка      |ElevenLabs: видео-реплики **+ весь текст курса** (`reading_text` + `reading_media`)                                                                   |—                         |`media_json`                      |
|7.6 — апрув озвучки|аудиоплеер на каждую реплику                                                                                                                          |`/voiceover/{project_id}` |`media_status=vo_approved`        |
|7.7 — картинки     |Leonardo: обложка курса + иллюстрация + декор карточки + **обложка каждого урока** + картинка к каждому `reading_media` + картинки `image_recognition`|`/final` (как часть курса)|`media_json`                      |
|7.8 — сведение     |вручную (синк рта); конвейер складывает в папку                                                                                                       |—                         |`media_status=ready_to_mux`       |
|7.9 — импорт       |`importer.py`: `course_json` → `public` (см. C6)                                                                                                      |—                         |`public.*`, `stage=published`     |


> **Видео-формат зафиксирован:** каждый видео-урок = ровно **3 шота × 10с = 30с** (Kling caps клип на 10с). Жёсткие склейки между шотами — норма (стиль brain-rot). Единый `character_lock` переиспользуется во всех 3 шотах для консистентности. Озвучка ≤28 слов на шот (умещается в 10с).

## C5. Контракт `course_json` (выход стадии 5) — все поля по §6

> Все имена полей соответствуют §6 и §6.5. Поля с суффиксом `_prompt` и блок `_production` — **служебные** (по ним генерится медиа); в продуктовые таблицы они не пишутся, см. C6.

```jsonc
{
  "course": {
    "slug": "bards-blood",                 // → courses.slug (идемпотентность импорта)
    "title": "The Bard's Blood",            // → courses.title
    "category": "Great People",             // строка → резолвится в courses.category_id
    "description": "...",                   // → courses.description
    "card_gradient_start": "#4C1D95",       // → courses.card_gradient_start
    "card_gradient_end": "#7C3AED",         // → courses.card_gradient_end
    "path_background_color": "#2A1758",     // → courses.path_background_color
    "popularity_score": 0,                  // → courses.popularity_score (старт 0)
    "sort_order": 0,                        // → courses.sort_order
    // --- служебные: image-gen промпты для всех трёх картинок карточки ---
    "cover_image_prompt": "...",            // → Leonardo → courses.cover_image_url
    "illustration_prompt": "...",           // → Leonardo → courses.illustration_url
    "card_decoration_prompt": "..."         // → Leonardo → courses.card_decoration_url
  },
  "characters": [
    {
      "canonical_name": "William Shakespeare",
      "aliases": ["Shakespeare", "Will", "the Bard", "William"],
      "is_historical": true,
      "character_lock": "Young man, 18, sharp jaw, dark Tudor doublet, ink-stained fingers, shoulder-length brown hair",
      "leonardo_prompt": "detailed portrait reference sheet, front view, ... plain white background, studio lighting, photorealistic"
    }
    /* по одному на говорящего/появляющегося персонажа; → pipeline.characters */
  ],
  "lessons": [                              // → public.lessons
    {
      "sort_order": 1,                      // → lessons.sort_order
      "title": "Who Killed the Glove-Maker's Son?",  // → lessons.title
      "cover_image_prompt": "...",          // служебное: → Leonardo → lessons.cover_image_url (обложка карточки урока, §12.4); по теме урока
      "tasks": [                            // → public.tasks
        {
          "type": "video",                  // → tasks.type
          "requires_check": false,          // → tasks.requires_check (дефолт по §6.5а)
          "sort_order": 1,                  // → tasks.sort_order
          "payload": { "autoplay": true },  // → tasks.payload (media_asset_id дозаполнит импорт)
          "_production": {                  // служебное (видео)
            "scene": "...",
            "target_seconds": 30,
            "shots": [                      // РОВНО 3 шота
              { "shot": 1, "duration_sec": 10,
                "kling_prompt": "Subject -> Action -> Context -> Style",
                "on_screen_text": "He married her in secret. She was already pregnant.",
                "voiceover": "Narrator: Before the plays, before the fame — there was a scandal.",
                "sfx_music": "tense low strings" }
            ]
          }
        },
        {
          "type": "reading_text",
          "requires_check": false, "sort_order": 2,
          "payload": {                      // строго по §6.5
            "title": "The Marriage Nobody Talks About",
            "segments": [
              { "text": "...", "audio_url": null, "_tts_voice": "Narrator", "_needs_audio": true }
              // _tts_voice/_needs_audio — служебные; audio_url дозаполнит TTS-импорт
            ]
          }
        },
        {
          "type": "reading_media",          // иллюстрированное чтение: картинка сверху + текст
          "requires_check": false, "sort_order": 3,
          "payload": {                      // строго по §6.5
            "media": { "kind": "image" },   // image_url дозаполнит импорт после генерации
            "title": "A Blight Across the Land",
            "segments": [
              { "text": "The blight does not stay in one place...", "audio_url": null,
                "_tts_voice": "Narrator", "_needs_audio": true },
              { "text": "Word spreads faster than any horse...", "audio_url": null,
                "_tts_voice": "Narrator", "_needs_audio": true }
            ],
            // служебное: промпт картинки для этого текста → Leonardo → payload.media.image_url
            "_media_prompt": "editorial illustration, desaturated relief map of Ireland, glowing county pins (Donegal, Belfast, Galway, Dublin, Limerick, Cork), dark vignette, ominous mood, 19th-century famine era"
          }
        },
        {
          "type": "matching",
          "requires_check": true, "sort_order": 4,
          "payload": { "prompt": "Match the date to the event",
                       "pairs": [ {"left":"1066","right":"Battle of Hastings"} ] }
        },
        {
          "type": "fill_blank",
          "requires_check": true, "sort_order": 5,
          "payload": { "text": "The war lasted ___ years.", "answer": "116",
                       "options": ["100","116","120"] }
        },
        {
          "type": "image_recognition",
          "requires_check": true, "sort_order": 6,
          "payload": { "prompt": "What does this picture show?",
                       "options": ["A","B","C","D"], "answer_index": 1,
                       "_image_prompt": "image-gen prompt" }
          // _image_prompt → Leonardo → payload.image_url дозаполнит импорт
        }
      ]
    }
  ]
}
```

**Дефолт `requires_check`** конвейер ставит по таблице §6.5а; редактор в админке может переопределить после импорта.

## C6. Контракт импорта `course_json` → `public` (`importer.py`)

> Идемпотентно по `course.slug` (повторный прогон не плодит дубликаты). Порядок: сначала генерится всё медиа (картинки/видео/аудио), затем создаются строки и подставляются реальные URL/ID.

|`course_json`                                                                                      |Таблица / поле `public`                                                      |Как импортируется                                                                                                                                                           |
|---------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|`course.slug`                                                                                      |`courses.slug`                                                               |ключ идемпотентности (upsert по slug)                                                                                                                                       |
|`course.title / description / *_gradient_* / path_background_color / popularity_score / sort_order`|одноимённые поля `courses`                                                   |напрямую                                                                                                                                                                    |
|`course.category` (строка)                                                                         |`courses.category_id`                                                        |**резолв**: `select id from categories where name = course.category`; если нет — создаётся/логируется ошибка импорта                                                        |
|`course.cover_image_prompt`                                                                        |`courses.cover_image_url`                                                    |Leonardo → загрузка в Storage → URL                                                                                                                                         |
|`course.illustration_prompt`                                                                       |`courses.illustration_url`                                                   |Leonardo → Storage → URL                                                                                                                                                    |
|`course.card_decoration_prompt`                                                                    |`courses.card_decoration_url`                                                |Leonardo → Storage → URL                                                                                                                                                    |
|`characters[]`                                                                                     |`pipeline.characters` (НЕ `public`)                                          |поиск по aliases → найдено: берём; нет: Leonardo → вставка. `kling_subject_id` пишется после загрузки в Kling. **В `public` не импортируется** — это производственные данные|
|`lessons[]`                                                                                        |`lessons`                                                                    |`course_id` (из созданного курса) + `sort_order`, `title`                                                                                                                   |
|`lessons[].cover_image_prompt`                                                                     |`lessons.cover_image_url`                                                    |Leonardo → загрузка в Storage → URL (обложка карточки урока, §12.4)                                                                                                         |
|`lessons[].tasks[]`                                                                                |`tasks`                                                                      |`lesson_id` + `type`, `requires_check`, `sort_order`, `payload` (без служебных полей)                                                                                       |
|`task._production.shots[]`                                                                         |`media_assets` (`kind=video`, `cloudflare_video_id`, `hls_url`, `poster_url`)|Kling → склейка → Cloudflare Stream; затем `tasks.payload.media_asset_id` ← новый `media_assets.id`                                                                         |
|`reading_text`/`reading_media` `payload.segments[]._needs_audio`                                   |`media_assets` (`kind=tts_audio`, `audio_url`) + `segments[].audio_url`      |ElevenLabs по `_tts_voice` → Storage → URL. Озвучивается **весь текст курса**, не только видео                                                                              |
|`reading_media.payload._media_prompt`                                                              |`payload.media.image_url` (при `media.kind="image"`)                         |Leonardo → Storage → URL картинки этого текста                                                                                                                              |
|`image_recognition.payload._image_prompt`                                                          |`payload.image_url`                                                          |Leonardo → Storage → URL                                                                                                                                                    |

**Что НЕ пишется в `public`:** все поля с `_` (`_production`, `_tts_voice`, `_needs_audio`, `_image_prompt`, `_media_prompt`), блок `characters[]`, `course.slug` пишется только в `courses.slug`. Служебные поля используются на этапе генерации медиа и затем отбрасываются.

## C7. Надёжность, безопасность, стоимость

- **Секреты** — только в `.env` на Mac mini (см. границу в C1). Service key — только в процессе конвейера.
- **Идемпотентность** — импорт по `slug`; поиск персонажей по aliases. Каждый шаг пишет `pipeline.events`.
- **Ретраи** — `httpx` с backoff. Кривой JSON от Claude → авто-повтор «return valid JSON only». Падение Kling/ElevenLabs/Leonardo → проект застывает на `media_status`, email-алерт, повтор кнопкой.
- **Kill-switch** — `PIPELINE_ENABLED=false` останавливает всю автогенерацию.
- **Стоимость (порядок величин):** Claude — центы за курс; Leonardo — ~$0.01–0.02 за 4 PNG; Kling — ~$0.07–0.14/сек → ролик 30с ≈ $2–4; ElevenLabs — центы за минуту; Supabase free / хостинг админки (Vercel/Netlify free) / email — $0; **Tailscale и туннель не используются.**

## C8. Фазы разработки конвейера

> Зависят от Phase 1 основного проекта (схема `public` + `pipeline`). Рекомендация: сначала C-Phase 2–4 (текстовый цикл без медиа — самое ценное и дешёвое), медиа (C-Phase 5) подключать после.

### C-Phase 1 — Каркас на Mac mini

**Цель:** процесс живёт 24/7, схема `pipeline` применена.
**Задачи:** pmset/autorestart; venv + `requirements.txt`; launchd поднимает `main.py` (APScheduler + поллер Supabase); `db.py` CRUD по схеме `pipeline`; email-уведомление. (Без Tailscale/FastAPI-UI — апрувер в админке.)
**AC:** процесс переживает рестарт; поллер видит изменения в `pipeline.*` и реагирует; heartbeat-письмо приходит.

### C-Phase 2 — Идеи (стадии 0–2)

**Цель:** воронка идей с апрувом.
**Задачи:** `/topics` + кнопка «Запустить сейчас» + **необязательное текстовое поле «Дополнение к сценарному промпту» (`user_note`, привязано к топику)**; P1 генерит 20 идей; приоритетные `seed_topics` забираются первыми; `/ideas` с чекбоксами + **то же необязательное поле в момент апрува идеи в топик** (пишется в `projects.user_note`, переносится из `seed_topics` если было задано на вводе); email.
**AC:** идеи генерятся по расписанию и кнопкой; апрув работает с телефона; поле дополнения необязательно — пустое не ломает поток.

### C-Phase 3 — Сценарии (стадии 3–4)

**Цель:** 3 варианта сценария по апруву идеи.
**Задачи:** P2 → `scenarios_json` (**если у топика `projects.user_note` непустой — подмешивается в промпт P2 отдельным блоком инструкций; пустой/NULL → P2 как есть**); `/scenario` с 3 колонками; выбор варианта / свой крючок.
**AC:** сценарии генерятся; выбор сохраняется и запускает стадию 5; **заданное дополнение к промпту отражается в сгенерированных сценариях, при пустом поле поведение P2 не меняется**.

### C-Phase 4 — Сборка курса + поэлементный апрув (стадии 5–6 + 5а)

**Цель:** полный `course_json` по контракту C5, апрув каждого элемента.
**Задачи:** P3 → `course_json` (все поля `courses` + уроки + задания + служебные промпты); нарезка на `review_items`; `/final` поэлементно; цикл доработки P4; переход дальше только при 100% approved.
**AC:** `course_json` валиден по C5; счётчик approved N/M; кнопка «В продакшен» активна только при полном апруве.

### C-Phase 5 — Медиа (стадии 7.0–7.8)

**Цель:** все картинки, видео, озвучка сгенерированы и одобрены.
**Задачи:** Leonardo (4 PNG персонажей + обложка курса/иллюстрация/декор карточки + **обложка каждого урока** + **картинка к каждому `reading_media`** + картинки `image_recognition`); Kling Subject Library; 3 клипа × 10с с `/clips`; ffmpeg-склейка; **ElevenLabs — озвучка видео-реплик И всего текста курса** (`reading_text` + `reading_media`) с `/voiceover`; папка для ручного mux. **Видео для `reading_media` в этой фазе не генерим** (только картинка) — отложено на потом.
**AC:** поиск персонажей по aliases не плодит дубли; все три картинки карточки курса сгенерированы; **у каждого урока есть обложка**; у каждого `reading_media` есть своя картинка; весь текстовый контент озвучен; клипы и озвучка проходят поэлементный апрув.

### C-Phase 6 — Импорт и полировка (стадия 7.9)

**Цель:** курс попадает в приложение.
**Задачи:** `importer.py` (контракт C6, идемпотентно по `slug`, резолв категории, дозаполнение `media_asset_id`/`audio_url`/`image_url`); видео → Cloudflare Stream; мониторинг живости; kill-switch.
**AC:** прогон создаёт полноценный курс в `public`, видимый в приложении; повторный прогон не дублирует; все обязательные поля `courses` заполнены.

-----

## *Конец Приложения C. Конвейер — отдельный подпроект; источник истины по схеме контента — §6 этого спека. Конвейер генерирует ровно поля §6 и импортирует их идемпотентно по `slug`.*

-----

# ПРИЛОЖЕНИЕ D — Подпроект «Дашборд очереди апрувов (Approval Queue)»

> **Платформо-независимость.** Дашборд очереди — **раздел внутри админки** (Приложение B): React + shadcn/ui + Tailwind, адаптив под телефон. Без Tailscale / Jinja / отдельного pipeline-UI. Тестовые требования (бывший раздел D7 «Unit-тесты» и тест-ориентированная D-Phase 1) **вынесены в отдельный тест-документ** и здесь не приводятся.

> Раздел внутри **админки** (Приложение B), маршрут `/queue`. Это **витрина очереди**: единый список всех проектов курсов, которые находятся в работе или ждут моего апрува. Цель — с одного взгляда (веб или телефон — адаптив) понять, **что висит из-за меня** и на каком этапе.
> 
> **Граница ответственности.** D **не дублирует** страницы апрува из Приложения C (`/ideas`, `/scenario`, `/final`, `/chars`, `/clips`, `/voiceover`) — он их **оркестрирует**: показывает очередь, статусы и ведёт на нужный детальный экран апрува (тоже в админке). Это «приёмная» перед детальными экранами.
> 
> **Дизайн-референс — та же дизайн-система shadcn/ui**, что и у админки (Приложение B): карточки, бейджи статусов, прогресс-бары, dark/light, **единая бренд-палитра проекта** (тот же источник, что и приложение и админка — см. §5; палитра из tweakcn в виде CSS-переменных). Реализация — в стеке **админки** (React + shadcn/ui), те же компоненты используются напрямую.
> 
> **Адаптив (mobile-first).** Открывается и в вебе (десктоп), и с телефона (один столбец, крупные тап-цели, sticky-кнопки) — по обычной ссылке с авторизацией админки (B2). Без Tailscale.

## D1. Назначение и поведение

- **Маршрут:** `/queue` — раздел админки (на него же ведёт ссылка из email-уведомления).
- **Источник данных:** таблица `pipeline.projects` (Приложение C, C3) — без новой схемы. Очередь — это выборка проектов, отсортированная по приоритету ожидания апрува.
- **Авто-обновление:** через Supabase (realtime-подписка или периодический опрос), чтобы статусы не устаревали.

## D2. Карточка проекта

Каждый проект очереди = одна карточка. Внутри:

1. **Название темы** — `working_title` (или `slug`, если титул ещё не задан).
1. **Прогресс-бар** — % завершённости по сквозной шкале стадий (см. D3). Один общий прогресс на весь жизненный цикл проекта.
1. **Бейдж статуса** — на каком мы сейчас этапе: текстовый ярлык + цвет (см. D4).
1. **Степпер — последовательность апрувов** внутри карточки: список всех контрольных точек проекта, где:

- **пройденные** — отмечены как пройденные (✓, приглушённый цвет),
- **текущая** — выделена как активная (акцентный цвет, пульсация/рамка),
- **будущие** — показаны как будущие (серые, неактивные).

1. **Флаг «ждёт меня»** — если проект сейчас требует апрува и завис именно из-за меня, это **явно видно**: заметный бейдж `⏳ AWAITING YOUR APPROVAL`, акцентная рамка карточки, карточка поднимается в верх списка.
1. **CTA** — кнопка «Открыть апрув» ведёт на соответствующую детальную страницу C (`/ideas/{id}`, `/scenario/{id}`, `/final/{id}`, `/chars/{id}`, `/clips/{id}`, `/voiceover/{id}`).

## D3. Сквозной степпер апрувов (контрольные точки)

Последовательность апрувов проекта (соответствует стадиям Приложения C и полям `stage` / `media_status`):

|#|Контрольная точка                   |Стадия C                                       |Страница апрува      |
|-|------------------------------------|-----------------------------------------------|---------------------|
|1|Апрув идеи                          |стадия 2 (`idea_pool → idea_approved`)         |`/ideas`             |
|2|Апрув сценария                      |стадия 4 (`scenario_draft → scenario_approved`)|`/scenario`          |
|3|Апрув финального курса (поэлементно)|стадия 6 (`final_draft → final_approved`)      |`/final`             |
|4|Апрув референсов персонажей         |media `chars_pending → chars_approved`         |`/chars`             |
|5|Апрув видеоклипов                   |media `clips_pending → clips_approved`         |`/clips`             |
|6|Апрув озвучки                       |media `vo_pending → vo_approved`               |`/voiceover`         |
|7|Готов к mux / импорт                |media `ready_to_mux → done → published`        |(ручной mux + импорт)|

Прогресс-бар карточки = (пройденные точки / всего точек). Текущая точка = первая непройденная.

## D4. Статусы и цвета (бейджи)

- `AWAITING YOUR APPROVAL` — **акцентный/предупреждающий** (висит из-за меня; верх списка).
- `GENERATING` — нейтральный/синий (конвейер работает, мой ход не нужен).
- `IN PRODUCTION` (медиа генерится) — синий.
- `BLOCKED / ERROR` — красный (упал Kling/ElevenLabs/Leonardo, нужен повтор кнопкой).
- `PUBLISHED` — зелёный (готово, в приложении).
- `REJECTED` — серый.

Сортировка очереди: `AWAITING YOUR APPROVAL` сверху → `BLOCKED` → `GENERATING/IN PRODUCTION` → завершённые внизу.

## D5. Ограничения на сам апрув (анти-«случайный тап»)

> Это ключевое требование: апрув не должен срабатывать одним случайным касанием. Действует на **детальных страницах апрува (C)**, но фиксируется здесь как общий контракт для всех точек апрува.

**Нельзя подтвердить или отклонить, не выполнив явное действие:**

- **Approve** заблокирован (кнопка `disabled`), пока **не поставлена галочка «I confirm I reviewed this and agree»**. Без галочки — апрув не проходит.
- **Reject** заблокирован, пока **не поставлена галочка «I do not agree»** **И** не заполнено **поле комментария** (непустое, минимум N символов). Без комментария отклонение не отправляется.
- Approve и Reject — **взаимоисключающие**: нельзя одновременно отметить «согласен» и «не согласен».
- На необратимые/далеко идущие действия («В продакшен», импорт) — **дополнительное подтверждающее модальное окно** (двойное подтверждение).
- **Серверная валидация:** ограничения проверяются **на сервере (Supabase Edge Function / бэкенд админки)**, а не только в браузере — запрос на апрув без галочки / на reject без комментария **отклоняется бэкендом** (нельзя обойти через прямой запрос). Сервисный ключ в браузер не попадает — админка пишет в `pipeline.*` через Edge Function (или admin-RLS).

## D6. Структура файлов (в `/pipeline`)

Витрина и экраны апрува живут в **админке** (`/admin`), логика перехода стадий — в Mac-mini-поллере; связь через Supabase.

```
admin/                                  # React + shadcn/ui (Приложение B)
├── src/pages/queue/
│   ├── QueuePage.tsx          # /queue — витрина очереди (карточки, степпер, бейджи)
│   ├── QueueCard.tsx          # компонент карточки (прогресс, статус, CTA)
│   └── approval/              # детальные экраны: ideas/scenario/final/chars/clips/voiceover
└── src/services/queue.ts      # чтение pipeline.projects/review_items из Supabase

supabase/functions/
└── pipeline-approval/         # Edge Function: апрув/реджект с серверной валидацией D5
                               #   пишет статус/комментарий в pipeline.* (без service key в браузере)

pipeline/poller.py             # Mac mini: видит approved/needs_revision → двигает стадию /
                               #   запускает доработку (5а), пишет pipeline.events
```

## D7. Фазы разработки (D-Phase)

> Зависит от C-Phase 1 (каркас pipeline + схема `pipeline`). Не вводит новых таблиц — читает `pipeline.projects`. Тестовые требования — в отдельном тест-документе.

### D-Phase 1 — Модель очереди (логика)

**Цель:** реализована логика очереди и ограничений апрува, независимая от UI.
**Задачи:** `queue_service` — маппинг каждого `stage` / `media_status` → набор пройденных/текущих/будущих точек степпера, расчёт процента прогресса (граничные: 0 пройдено / всё пройдено), определение флага «ждёт меня» только для стадий, требующих апрува, сортировка очереди (awaiting → blocked → generating → done), фолбэк заголовка на `slug` при отсутствии `working_title`; `approval_service` — переход стадии при валидном апруве, запуск цикла доработки (5а) при reject с комментарием, идемпотентность (повторный апрув не двигает стадию дважды), серверная валидация D5.
**AC:** для любого `stage`/`media_status` степпер и прогресс считаются корректно; апрув без галочки и reject без валидного комментария отклоняются логикой; повторный апрув одной точки не двигает стадию дважды.

### D-Phase 2 — Витрина очереди

**Цель:** `/queue` показывает карточки.
**Задачи:** `QueuePage.tsx` + `QueueCard.tsx` — React/shadcn компоненты в `/admin` (по токенам §5); прогресс-бар, бейдж статуса, степпер (пройдено/текущее/будущее); флаг «ждёт меня» с акцентом и подъёмом наверх; адаптив (веб + телефон); авто-обновление (Supabase realtime/опрос).
**AC:** очередь открывается и в вебе, и с телефона (по ссылке, без Tailscale); проект, ждущий апрува, явно выделен и стоит сверху; CTA ведёт на нужный детальный экран апрува.

### D-Phase 3 — Ограничения апрува (серверные)

**Цель:** апрув/реджект защищены от случайного тапа.
**Задачи:** Edge Function `pipeline-approval` с серверной валидацией D5; галочки «согласен»/«не согласен»; обязательный комментарий при reject; disabled-состояния кнопок; двойное подтверждение необратимых действий.
**AC:** нельзя подтвердить без галочки и отклонить без комментария ни через UI, ни через прямой POST; необратимые действия требуют второго подтверждения.

-----

*Конец Приложения D. Дашборд очереди — витрина и оркестратор апрувов поверх `pipeline.projects` (Приложение C); детальные экраны апрува — в Приложении C. Дизайн-референс общий с админкой — shadcn/ui (Tailwind-токены §5). Тестовые требования — в отдельном тест-документе.*