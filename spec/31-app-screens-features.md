# 31 · Экраны и фичи приложения — Lorebinge SPEC (RN9)

> Часть разбитого спека. Источник-архив: [`spec-rn9.md`](spec-rn9.md) — не редактировать. Содержит: §9 доменные модели, §10 навигация, §11 server-driven UI, §12 экраны, §13 фичи + критерии приёмки. Оглавление, граф зависимостей и карта перекрёстных ссылок — в [00-index.md](00-index.md).

-----

## 9. Доменные модели (TypeScript, `src/data/model/`)

> Kotlin data-классы заменены TypeScript-типами. Рантайм-валидация DTO — через zod (`src/data/supabase/dto.ts`), маппинг DTO → доменные модели — в репозиториях.

```ts
export interface UserProfile {
  id: string;
  username: string;
  displayName: string;
  knowledgeLevel: number;             // HUD-уровень; +1 за каждые coursesPerLevel курсов (§12.0)
  coursesCompleted: number;           // для прогресса до следующего уровня (дропдаун level, §12.0)
  diamonds: number;
  battery: number;
  batteryMax: number;
  isSubscriber: boolean;              // trial/active → батарея заменена ассетом
  subscriptionBadge: SubscriptionBadge | null;
}
// Аватара нет (убран из MVP, НЕКР-17).

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
  slug: string;                       // человекочитаемый id для маркетинговых диплинков (§10, КРИТ-6)
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
  isCompleted: boolean;               // курс пройден пользователем → тёмный overlay + значок «пройдено» на карточке (§12.3, D)
}

export interface CourseProgress {
  courseId: string;
  lessonsCompleted: number;
  totalLessons: number;
  // lastLessonId убран: карточка In Progress ведёт на course_path, не в урок (НЕКР-11/12)
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

// Точное соответствие app_config.economy (§6.4). Маппинг snake_case DTO → camelCase в репозитории.
export interface LessonDiamondReward { amount: number; weight: number; }

export interface TrialOffer {
  trialEnabled: boolean;
  trialDays: number;
  timerEnabled: boolean;
  timerDurationHours: number;
  timerStartsOn: 'first_subscription_view' | 'install';
}

export interface EconomyConfig {
  startingDiamonds: number;
  batteryMax: number;
  batteryCostPerAnswer: number;
  // batteryRewardPerLesson убран (Gap #6): бонуса заряда за завершение урока нет, награда — только алмазы.
  batteryRefillMinutes: number;
  batteryRefillAmount: number;
  lessonDiamondRewards: LessonDiamondReward[];   // взвешенный рандом за первое прохождение
  lessonReplayDiamondReward: number;             // фикс за перепрохождение (дефолт 1, ВАЖН-15)
  coursesPerLevel: number;                       // курсов на +1 knowledgeLevel (дефолт 5, §12.0)
  streakFreezeCostPerDay: number;                // автосписание алмазов за пропущенный день (§A5)
  mascotToneReactionEnabled: boolean;
  trialOffer: TrialOffer;                        // §A14
}
```

**Обёртка результата** (`src/core/common/result.ts`):

```ts
export type ErrorType =
  | 'NETWORK' | 'BATTERY_EMPTY' | 'INSUFFICIENT_DIAMONDS'
  | 'AUTH' | 'SERVER' | 'UNKNOWN';

export type AppResult<T> =
  | { ok: true; data: T }
  | { ok: false; type: ErrorType; message?: string };
```

> **UiState экранов** — discriminated union, напр. `type HomeUiState = { kind: 'loading' } | { kind: 'content'; blocks: HomeBlocksData } | { kind: 'error'; message: string }`. Возвращается хуком `useHomeScreen()` и потребляется компонентом-`Screen`.

-----

## 10. Навигация (граф маршрутов)

> **React Navigation v7.** Root — native-stack; внутри — `BottomTab` с **5 вкладками** (одна из них — **Catalog — по умолчанию выключена**, «спящая»: рендерится, только когда админ включит её в `ui_configs`, §11/§12.3г), **каждая со своим вложенным native-stack** (сохранение back stack на вкладку). Параметры между экранами — **только id**. Тип-безопасность через `RootStackParamList`/`TabParamList`.
>
> **HUD + нижняя панель вкладок = постоянный каркас.** И HUD (сверху), и таб-бар (снизу) рендерятся на уровне `BottomTabs` (`main`) и видны на **всех** экранах вкладок, включая запушенные внутри них `course_path` и `category`. Корневые экраны `lesson`/`reward`/`paywall` лежат **над** `main` и перекрывают его целиком — пока открыт любой из них, HUD и таб-бар не видны (это и есть «полноэкранный» режим урока/награды/пейволла).

```
RootStack (native-stack)
├── splash
├── auth (native-stack)
│   ├── auth/login
│   └── auth/register            (email + код подтверждения)
├── main  (BottomTabs — 5 вкладок; Catalog по умолчанию ВЫКЛЮЧЕНА (спящая); HUD сверху + таб-бар снизу = общий каркас)
│   ├── home          (вкладка 1, native-stack)
│   │   ├── home/feed                  (лента блоков: Lores, In Progress, Popular — БЕЗ Categories)
│   │   └── course_path/{courseId}     (сетка уроков; HUD+таб-бар видны) — presentation: card
│   ├── catalog       (вкладка 2 — СПЯЩАЯ; enabled:false по умолчанию, включается из админки B6.7, §12.3г)
│   │   ├── catalog/feed               (алфавитный индекс категорий, §12.3г)
│   │   ├── category/{categoryId}      (курсы одной категории)          — presentation: card
│   │   └── course_path/{courseId}     (сетка уроков)                   — presentation: card
│   ├── grid          (вкладка 3, native-stack)
│   │   ├── grid/feed                  (инстаграм-сетка обложек курсов)
│   │   └── course_path/{courseId}     (тот же экран курса)             — presentation: card
│   ├── subscription  (вкладка 4, блок G)
│   └── profile       (вкладка 5, native-stack)
│         ├── profile/main             (Profile — инфо профиля; квесты убраны)
│         └── settings
│               ├── settings/preferences
│               ├── settings/profile
│               ├── settings/course
│               ├── settings/account_linking
│               ├── settings/help_center      (in-app webview/native)
│               ├── settings/send_feedback
│               ├── settings/terms            (in-app)
│               └── settings/privacy          (in-app)
├── lesson/{lessonId}             (раннер заданий; HUD+таб-бар скрыты) — fullScreenModal
├── reward/{lessonId}             (маскот + награда; скрыты)           — fullScreenModal
└── paywall                       (ТОЛЬКО battery=0 во время задания)  — fullScreenModal
```

> **`course_path` и `category` — общие экраны вкладок.** Регистрируются в стеке каждой вкладки, откуда на них переходят (`course_path` — из `home`, `grid`, `catalog` и по диплинку — в стек `grid`; `category` — **только из `catalog`**, §12.3г). Поэтому таб-бар и HUD остаются на месте, а возврат — стрелкой/жестом внутри стека вкладки (§10.1). Параметр — только `courseId`/`categoryId`. **Пока Catalog выключена, экран `category` недостижим** (единственный вход в него — через индекс каталога; на Home категорий больше нет).
> **Квесты убраны из проекта целиком** (глобальное решение): отдельной вкладки/маршрута нет, блока `QUESTS` нет, экрана `§12.7` нет. **Магазина заморозки тоже нет** (убран, КРИТ-8): заморозка стрика автоматическая за алмазы, отдельного блока/маршрута `shop` нет.

**Deep links / Universal Links (заложить структуру сразу, обе платформы):**

> **Маркетинговые ссылки — по `slug`, внутренняя навигация — по `id` (КРИТ-6).** Внешняя ссылка содержит человекочитаемый `slug` курса (`bards-blood`), не uuid. В linking-конфиге описывается резолв `slug → id`: прямой `select id from courses where slug = ...` (RLS read разрешён), затем переход на `course_path/{id}`. Внутри приложения между экранами по-прежнему передаётся только `id`.

> **Таб-якорь диплинка и поведение «Назад» (Gap #5).** Диплинк-курс открывается как `course_path/{id}`, **запушенный в стек вкладки `grid`** (вкладка Grid становится активной). Кнопка/жест «Назад» с такого `course_path` ведёт **на корень этого таба — `grid/feed`** (инстаграм-сетка), а не наружу из приложения. Так пользователь из соц-сети попадает в курс, а «Назад» оставляет его в витрине Grid, продолжая воронку.

- Custom scheme: `lorebinge://course/{slug}` → резолв `slug→id` → открывает `course_path/{id}` **в табе Grid**.
- **HTTPS-ссылки для маркетинга:** `https://lorebinge.app/course/{slug}` → тот же экран (таб Grid).
  - **iOS Universal Links:** файл `apple-app-site-association` на домене + Associated Domains entitlement (`applinks:lorebinge.app`).
  - **Android App Links:** `assetlinks.json` (Digital Asset Links) на домене + intent-filter (`autoVerify`).
- Конфигурируется в React Navigation `linking` (`prefixes` + `config` с функцией-резолвером `slug→id`); `scheme` и associated domains — в `app.config.ts` (`scheme`, `ios.associatedDomains`, `android.intentFilters`).

**Видимость HUD и таб-бара.** Оба — части постоянного каркаса `main`: видны на всех экранах вкладок (`home`, `grid`, `subscription`, `profile`) и на запушенных внутри вкладок `course_path`/`category`. Скрыты только когда открыт корневой полноэкранный экран — `lesson`, `reward`, `paywall` (они перекрывают `main`).

**HUD: батарея ↔ ассет подписки.** Слот заряда отображается по статусу подписки:

- **Не-подписчик** (`subscriptions.status` ∈ `none`/`expired`/`cancelled`) → иконка батареи + счётчик `battery/battery_max`; цвет по токенам `batteryNormal`/`batteryLow` (красный при ≤3).
- **Подписчик** (`trial`/`active`) → вместо батареи яркий **ассет из `app_config.subscription_badge`** (Rive/Lottie/SVG/PNG по `asset_type`, опц. `label`). Заряд скрыт.
- Статус берётся из `subscriptions` (server-authoritative), кэш-фолбэк — последнее известное значение (MMKV).

> **Полный состав HUD, интерактивность (тапы/дропдауны) и окно повышения уровня — каноном в §12.0.** Здесь — только подмена батарея↔ассет и видимость каркаса.

> **Safe area / системные жесты.** Все экраны учитывают инсеты (`react-native-safe-area-context`): нотч/Dynamic Island (iOS), системную навигацию-жестами (Android). Модальные экраны (`lesson`, `paywall`) — с корректной кнопкой/жестом закрытия на обеих платформах; на iOS свайп-вниз для модалок при необходимости отключается на `lesson` (чтобы не прерывать урок случайно).

### 10.1 Возврат на предыдущий экран (навигация «назад»)

> Как пользователь возвращается с экрана на предыдущий. Стек и аффордансы дают **штатные средства native-stack** React Navigation; собственный костыль-навигатор не пишем.

**Кнопка «назад» (стрелка в хедере).** На **запушенных** экранах (не корни вкладок) сверху слева — иконка-стрелка **`ChevronLeft`** (Lucide, цвет — токен `foreground`), тап → `navigation.goBack()`. Применяется на:

- `course_path/{courseId}` — назад на экран, откуда зашли (`home`-лента, `grid`-сетка или `category`);
- `category/{categoryId}` (курсы категории) — назад на корень таба **Catalog** (`catalog/feed`, §12.3г);
- `settings` и **все** его подэкраны (`settings/preferences`, `settings/profile`, `settings/course`, `settings/account_linking`, `settings/help_center`, `settings/send_feedback`, `settings/terms`, `settings/privacy`) — каждый возвращает на предыдущий уровень (подэкран → `settings` → `profile`).

> Стрелка-назад и HUD не конфликтуют: на запушенных экранах вкладок (`course_path`, `category`, `settings/*`) HUD остаётся сверху, стрелка — отдельным элементом панели экрана слева (по дизайну кита). HUD скрыт только на корневых полноэкранных `lesson`/`reward`/`paywall`.

**Корни вкладок** (`home`, `grid`, `subscription`, `profile`) — **без** стрелки-назад: это нижние точки своих стеков. Переключение между вкладками идёт через `BottomTab`, а **не** кладётся в общий back stack — у каждой вкладки свой стек сохраняется независимо (см. §10).

**Системные жесты возврата (обе платформы):**

- **iOS** — свайп от левого края экрана (встроенный `gestureEnabled` native-stack) эквивалентен тапу по стрелке. Отключается только там, где это специально оговорено (модалка `lesson`, см. §12.5в / safe-area-заметку выше).
- **Android** — аппаратная/жестовая кнопка «Назад» вызывает тот же `goBack()` (обрабатывается React Navigation автоматически). На корне вкладки системный «Назад» уводит на вкладку `home`, с `home` — сворачивает приложение (стандартное поведение).

**Корневые полноэкранные экраны** (`lesson`, `reward`, `paywall`) стрелки-назад **не имеют** — закрываются своими средствами: крестик ✕ с подтверждением для `lesson` (§12.5в); **тап по окну** для `reward`/level-up (полноэкранные окна-оверлеи, закрываются касанием — §12.6/§12.0, Gap #8); крестик ✕ для `paywall`, который **возвращает в курс** (`course_path`) с сохранённой позицией урока (§12.9, Gap #3). Не через хедер-back.

**Подтверждение выхода из урока (прогресс сохраняется, Gap #3/#4).** Любой путь выхода из `lesson` (стрелка отсутствует, но аппаратный «Назад» на Android / свайп там, где не отключён) проходит **через ту же подтверждающую плашку** (§12.5в) — чтобы случайный «Назад» не выкинул из урока посреди задания. **Прогресс при этом сохраняется** (`user_lesson_progress.current_task_index`, §A7): следующий заход продолжит урок **с того же задания**. На остальных экранах возврат без подтверждения.

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
        { "id": "lores",       "type": "LORES",       "enabled": true, "order": 1 },
        { "id": "in_progress", "type": "IN_PROGRESS", "enabled": true, "order": 2 },
        { "id": "popular",     "type": "POPULAR",     "enabled": true, "order": 3 }
      ]
    },
    { "id": "catalog", "icon": "library", "label_key": "nav_catalog", "enabled": false, "order": 2,
      "blocks": [ { "id": "catalog", "type": "CATALOG", "enabled": true, "order": 1 } ] },
    { "id": "grid", "icon": "grid", "label_key": "nav_grid", "enabled": true, "order": 3,
      "blocks": [ { "id": "grid", "type": "GRID", "enabled": true, "order": 1 } ] },
    { "id": "subscription", "icon": "crown", "label_key": "nav_sub", "enabled": true, "order": 4,
      "blocks": [ { "id": "subscription", "type": "SUBSCRIPTION", "enabled": true, "order": 1 } ] },
    { "id": "profile", "icon": "profile", "label_key": "nav_profile", "enabled": true, "order": 5,
      "blocks": [ { "id": "profile", "type": "PROFILE", "enabled": true, "order": 1 } ] }
  ]
}
```

> **Раскладка вкладок.** Блок `GRID` — на **отдельной вкладке Grid** (инстаграм-сетка обложек, §12.3б); Home — это `LORES`/`IN_PROGRESS`/`POPULAR` (**блок `CATEGORIES` с Home убран**). **Категории переехали в новую вкладку `Catalog`** — алфавитный индекс категорий (§12.3г). **Catalog по умолчанию `enabled:false` (спящая):** код едет в сборке, но в таб-баре её нет, пока админ не включит вкладку через Layout Builder (B6.7) — это сделано осознанно: пока курсов мало, хватает лент Home/Grid; индекс категорий включат, когда каталог разрастётся. **Блоков `QUESTS`/`SHOP` нет** (убраны). Иконки: Grid — `grid` (Lucide `LayoutGrid`), Catalog — `library` (Lucide `LibraryBig`; визуально отличается от Grid). Порядок вкладок (когда Catalog включат): **Home · Catalog · Grid · Subscription · Profile**; пока выключена — **Home · Grid · Subscription · Profile**.

### Клиентская реализация (RN)

- **Реестр блоков:** `type BlockType = 'GRID' | 'LORES' | 'IN_PROGRESS' | 'POPULAR' | 'CATALOG' | 'SUBSCRIPTION' | 'PROFILE'`. *(Блок `CATALOG` — новый, алфавитный индекс категорий §12.3г; старый блок `CATEGORIES` с Home убран. Блоки `QUESTS` и `SHOP` удалены — квесты и магазин убраны.)*
- **Рендерер:** компонент `<RenderBlock type={...} />` мапит `BlockType` → соответствующий компонент через объект-словарь `Record<BlockType, React.ComponentType>`. Блок получает данные из своего хука, не зная о расположении.
- Каждая вкладка рендерит свой список `blocks` (отсортированный, только `enabled`) в **`FlatList`** (ленивый рендер; ⟵ LazyColumn).
- **Неизвестный `type`** (конфиг новее приложения) — пропускается без краха (`if (!registry[type]) return null`).
- **Иконки вкладок** — мапинг `icon`-строки → компонент иконки (напр. из `lucide-react-native`/`@expo/vector-icons`); неизвестная иконка → дефолтная.

-----

## 12. Спецификации экранов

> Формат: назначение · состояния UiState · ключевые действия. Каждый экран = `Screen` (компонент) + `useXxxScreen()` (хук) + `UiState` (тип). Списки — `FlatList`; жесты — Gesture Handler + Reanimated; картинки — `expo-image`; видео — `expo-video`; Rive — `rive-react-native`.
>
> **Все строки интерфейса — английский, через i18n** (`src/core/i18n/en.json`). Это сквозное правило (§3) — в подразделах ниже отдельно **не повторяется** (НЕКР-8).

### 12.0 HUD (постоянный верхний каркас) — *канон*

- **Назначение:** верхняя панель-каркас, видимая на всех вкладках и на `course_path`/`category` (видимость — §10). Не часть экрана-вкладки: рендерится на уровне `BottomTabs`.
- **Состав (слева направо, фикс-порядок): `level` · `streak` · `diamonds` · `battery/ассет`.** Уровень в HUD **есть** (подтверждено, ВАЖН-3); §12.11 (Profile) HUD **не дублирует**.
  - **level** — иконка уровня + число `knowledge_level`.
  - **streak** — иконка огня + `current_streak`.
  - **diamonds** — иконка алмаза + баланс.
  - **battery/ассет** — батарея (не-подписчик) ↔ ассет подписки (подписчик); подмена и цвета — §10.
- **Источник данных:** прямое чтение **своих** строк `profiles` + `streaks` через TanStack Query; инвалидация после мутаций RPC (`submit_task_attempt`, `complete_lesson`, `refresh_user_state`). При старте/заходе значения актуализирует `refresh_user_state` (батарея + стрик, §8.1).
- **Интерактивность (тап по элементам):**
  - **Тап по `level`** → **выпадающее сверху окошко** (дропдаун): «Level N» + сколько курсов осталось пройти до следующего уровня (`coursesPerLevel − coursesCompleted % coursesPerLevel`) — прогресс-трекинг. Закрывается **тапом вне окошка**.
  - **Тап по `diamonds`** → **ничего** (неинтерактивно).
  - **Тап по `streak`** → ничего (неинтерактивно в MVP).
  - **Тап по слоту заряда** — поведение зависит от подписки (Gap #6):
    - **Не-подписчик** (показана батарея) → **выпадающее окошко с кнопкой**; по кнопке → переход на вкладку `subscription` (раздел подписок). Пейволл отсюда **не** открывается (пейволл — только при `battery=0` во время задания, §12.9).
    - **Подписчик** (показан ассет-замена батареи) → **выпадающее окошко с надписью** «You're already subscribed» + **название тарифа по-английски** (из `subscription_badge.label`, напр. «SUPER», или маппинг `subscriptions.product_id` → «Super» / «Super Family»). Кнопки перехода нет — у пользователя уже есть подписка.
  - **Все окошки-дропдауны** закрываются тапом вне их области (overlay-перехватчик).
- **Повышение уровня (level-up):** при `complete_lesson` с `leveled_up=true` (каждые `coursesPerLevel` курсов, §A4) поверх всего показывается **полноэкранное окно-оверлей** — как экран награды в конце урока, но с **другой анимацией** маскота и сообщением «Level up! You reached level N». **Закрывается тапом по окну** (Gap #8), без кнопки. (Это отдельный оверлей, не `reward`-экран.) **Очерёдность:** если завершение урока дало и награду, и уровень, окна идут **последовательно** — сначала `reward`, затем level-up, затем `course_path`; каждое снимается тапом (два тапа закроют оба; быстрый двойной тап проскочит оба — §12.6).
- **UiState HUD:** часть состояния каркаса; при отсутствии данных — последнее кэшированное значение (MMKV), без блокировки экрана.

### 12.1 Splash

- **Назначение:** Rive-анимация на цветном фоне; проверка сессии; загрузка конфига и economy; **проверка минимальной версии** (force-update, §A15). Нативный splash (`expo-splash-screen`) удерживается до готовности JS, затем — Rive-сплэш.
- **Force-update (§A15):** после загрузки конфига сравнить `app_config.min_supported_build` с `nativeBuildVersion`; если сборка меньше → блокирующий экран «Please update the app» с кнопкой в стор (по платформе). Дефолт `min_supported_build=0` (выключено).
- **Переход:** сессия есть → `main` (выбор стартовой вкладки — ниже); нет → `auth/login`. **Нет интернета на старте → заглушка** «Service is temporarily unavailable. Please try again later.» (англ.) + кнопка Retry (ВАЖН-4). Холодный старт **требует сети** (нужны данные ленты) — «Home на кэше» при отсутствии сети **не показываем**; встроенный `default_ui_config.json` страхует только отказ загрузки UI-конфига, когда сеть в целом есть (см. F9, §A11).
- **Стартовая вкладка (холодный старт, без диплинка) — по пройденным курсам (Gap #1):** если у пользователя **нет ни одного пройденного курса** (`profiles.courses_completed = 0`) → он новичок → стартуем на **Grid** (витрина-крючок, §12.3б); иначе (`courses_completed ≥ 1`) → **Home**. После первого **пройденного курса** дефолт сам становится Home — признак серверный (`courses_completed`), самокорректируется и переживает переустановку (не локальный флаг). Так устраняется прежнее противоречие «по завершённому уроку» — критерий теперь единый: пройденный курс.
- **Диплинк имеет приоритет.** Любой переход из соц-сетей идёт через **Linkrunner** (`@linkrunner/react-native`) — единая ссылка на курс. Развилка — по факту установки приложения, а не по «новизне» пользователя:
  - **Приложение установлено** → ссылка (`lorebinge.app/course/{slug}`) резолвится `slug→id` → открывает `course_path/{id}` **в стеке вкладки Grid** (моментальный диплинк, КРИТ-6; таб-якорь — Gap #5).
  - **Не установлено** → ссылка ведёт в App Store / Google Play → после установки и **первого запуска** SDK Linkrunner отдаёт сохранённый `slug` (отложенный диплинк) → резолв `slug→id` → переходим в `course_path/{id}` **в табе Grid**. «Первый запуск» здесь — лишь технический момент доставки атрибуции для ветки «не установлено», а не отдельный сценарий.
  - В обоих случаях переход в курс имеет наивысший приоритет — **выше выбора стартовой вкладки**. Вкладка Grid становится активной, «Назад» из курса ведёт на её корень `grid/feed` (Gap #5, §10). Если данных диплинка нет — обычная логика стартовой вкладки. Сервис можно заменить собственным решением без изменений в логике навигации.

### 12.2 Auth (login / register)

- **Login:** кнопка «Continue with Google»; **на iOS обязательна** кнопка **«Sign in with Apple»** (Apple Guideline 4.8 — при наличии стороннего соц-входа); **email + пароль** (для аккаунтов с email-методом).
  - Google — `@react-native-google-signin/google-signin` (или Supabase OAuth); Apple — `expo-apple-authentication` → передача identity-token в `supabase.auth.signInWithIdToken`.
- **Register (email-путь):** **email + пароль + код подтверждения** (ВАЖН-5). Поток: ввод email и пароля → `supabase.auth.signUp({ email, password })` → код подтверждения на почту → ввод кода (email confirm). Соц-вход (Google/Apple) пароля и кода не требует. «Password» в Settings (§12.12) = смена пароля, доступна только аккаунтам с email-методом.
- **UiState:** `idle | loading | error(message)`.
- **Действие:** успешный вход → создаётся/находится `profiles` (стартовые алмазы) → `home`. Сессия Supabase сохраняется в `expo-secure-store` (Keychain/Keystore).

### 12.3 Home (вкладка 1)

- **Назначение:** лента блоков из конфига — **Lores, In Progress, Popular** (блок Grid — на отдельной вкладке, §12.3б; **блок Categories убран с Home** — категории переехали в спящую вкладку Catalog, §12.3г). HUD и таб-бар — общий каркас (§10), не часть экрана.
- **UiState:** `loading | content(blocksData) | error`.
- **Данные:** один вызов `get_home_feed` (через TanStack Query); **категории в нём больше не приходят** (их отдаёт `get_catalog`, §8.1).
- **Действия:** тап карточки (Lores / In Progress / Popular) → `course_path/{id}`. *(Плитки категорий с Home убраны — путь в категории теперь через вкладку Catalog, §12.3г.)*
- **Геометрия карточек Home (канон).** Все карточки курсов на Home — **вертикальные ~3:4** (портрет, но не «инстаграмные» 9:16 — это пропорция Grid, §12.3б) со **скруглёнными углами** (`radiusLarge`). Применяется к **Lores, In Progress, Popular**; та же геометрия — у карточек курсов на экране категории (§12.3в). Так Home читается как «полка карточек-историй», а Grid (9:16) — как отсылка к рилсам.
- **Состояние «курс пройден» (overlay, D).** Если `is_completed=true` (из `get_home_feed`) — поверх обложки/градиента карточки кладётся **более тёмный затемняющий overlay** (токен `overlay-completed`, §5.6) и в углу — **значок «пройдено»** (Lucide `CircleCheck`/`Check` в кружке, токен `success`). Карточка остаётся кликабельной (тап → `course_path`, можно перепройти). Затемнение «гасит» пройденный курс, не пряча его и не мешая соседним карточкам.

#### 12.3а Карточка Lores (Block B / `LoresBlock`) — детально

- **Контейнер блока:** заголовок секции **«Lores»** + подзаголовок; ниже — горизонтальный список (`FlatList horizontal`, `snapToInterval`/`pagingEnabled` по желанию). Карточек = числу опубликованных курсов; скролл бесконечный «до упора». Данные — `lores[]` из `get_home_feed`.
- **Одна карточка (`LoreCard`)** — вертикальная **~3:4** (см. геометрию Home выше), фон = **линейный градиент** `card_gradient_start → card_gradient_end` (`expo-linear-gradient`, сверху вниз), скругление `radiusLarge`. Если `is_completed=true` — поверх всего тёмный `overlay-completed` + значок «пройдено» (§12.3, D). Раскладка сверху вниз:

1. **Логотип** — `illustration_url`, крупный, по центру в верхней трети (`expo-image`).
1. **Пилюля категории** — `category_name` в `Pill` (светлый фон, текст = `primary`), UPPERCASE.
1. **Название** — `title`, `headlineMedium`/extra-bold, 1–2 строки.
1. **Счётчики** — две строки: `lessons_count` + label `lessons`, `tasks_count` + label `tasks` (число акцентно, label приглушён). Строки — из i18n (`lore_card_lessons`, `lore_card_tasks`).
1. **Кнопка «Get the lore»** — внизу слева (`labelLarge`), тап → `course_path/{course_id}`.
1. **Декор** — `card_decoration_url`, внизу справа, поверх градиента (не интерактивный).

- **Действие:** тап по карточке или кнопке «Get the lore» → `course_path/{course_id}`.
- **Состояния:** loading — skeleton-карточки; empty — «No lores yet»; ошибки изображений — placeholder (`expo-image`), не краш (см. A13).

#### 12.3б Grid (вкладка 2) — инстаграм-сетка обложек

- **Назначение:** главный экран-витрина, особенно для **новопришедших из соцсетей**. Показывает обложки-превью **первых видео** всех опубликованных курсов единой сеткой — как вкладка Reels в Instagram. Пользователь, узнавший обложку ролика, который зацепил его в ленте, тапает её, чтобы «продолжить историю». Маркетинговая воронка (§1): ролик в соцсети → та же обложка в Grid → курс.
- **Раскладка:** вертикальный бесконечный скролл, **3 колонки**, ячейки — **вертикальные ~9:16** (пропорция рилсов Instagram) со **скруглёнными углами** (`radiusMedium`) и **небольшими отступами между ячейками** (не бесшовно). Это сознательный баланс: ячейка сразу читается как **отсылка к ролику из Instagram-рилсов**, но скругление + отступы не дают спутать её с бесшовной сеткой видео в галерее. Ячейка = обложка **первого урока** курса = `lessons.cover_image_url` урока с минимальным `sort_order` («первое видео» курса; генерируется конвейером B3 `lesson_cover`, Приложение C — тот же кадр, что зацепил пользователя в соц-сети). `expo-image` (кэш/blurhash/placeholder; ошибка → placeholder, не краш, A13).
- **Одна ячейка = один курс.** Тап по ячейке → **`course_path/{course_id}`** (экран курса с карточками-уроками), **а не** проигрывание видео — само видео играется уже внутри первого урока.
- **Состояние «курс пройден» (D):** если `is_completed=true` — поверх обложки ячейки **более тёмный overlay** (`overlay-completed`) + значок «пройдено» (`success`, Lucide `CircleCheck`). Ячейка остаётся кликабельной (перепрохождение).
- **UiState:** `loading | content(items) | empty | error`.
- **Данные:** `get_grid_feed` → список `{ course_id, cover_image_url, title, is_completed }` по всем опубликованным курсам, где `cover_image_url` — обложка первого урока курса; порядок — `sort_order`/`popularity_score` (деталь UI-фазы).
- **Состояния:** loading — skeleton-сетка; empty — «Nothing here yet»; error — ретрай.

> **Стартовая вкладка новичка (Gap #1).** Пользователь **без единого пройденного курса** (`courses_completed=0`) попадает на Grid первым экраном (решает splash, §12.1); после первого **пройденного курса** дефолт — Home. Приход по диплинку из соц-сети ведёт сразу в курс (в стеке Grid), минуя этот выбор.

#### 12.3в Category (курсы одной категории)

- **Назначение:** список курсов выбранной категории. Открывается тапом по строке категории в индексе **Catalog** (§12.3г) — единственный вход (на Home категорий больше нет). Путь в курс наряду с Lores/In Progress/Popular/Grid.
- **Раскладка:** заголовок = имя категории; ниже — сетка **вертикальных ~3:4** карточек курсов со скруглёнными углами (та же геометрия, что на Home, §12.3): обложка `cover_image_url` + название + счётчики уроков/заданий (упрощённый вариант карточки Lores). Тап по карточке → `course_path/{course_id}`.
- **Состояние «курс пройден» (D):** при `is_completed=true` — тёмный `overlay-completed` + значок «пройдено» (`success`), как на Home/Grid.
- **UiState:** `loading | content(category, courses) | empty | error`.
- **Данные:** `get_category_courses(category_id)` → `{ category:{id,name}, courses:[{id,title,cover_image_url,lessons_count,tasks_count,is_completed}] }` (только `is_published`).
- **Возврат:** стрелка-назад/жест → корень таба **Catalog** (`catalog/feed`, §10.1).
- **Состояния:** empty — «No courses in this category yet»; error — ретрай.

#### 12.3г Catalog (вкладка — алфавитный индекс категорий) — *спящая, включается из админки*

- **Назначение:** раздел **«All Courses»** — все курсы, разложенные по категориям. Верхний уровень = **алфавитный индекс категорий**; тап по категории → её курсы (§12.3в). Сюда «переехал» бывший блок Categories с Home. **По умолчанию вкладка выключена** (`enabled:false`, §11): пока курсов/категорий мало, хватает лент Home/Grid; админ включает Catalog через Layout Builder (B6.7), когда каталог разрастётся (ориентир — ~8–10+ категорий).
- **Раскладка — `SectionList` с «липкими» заголовками-буквами:**
  - **Секции** = группировка категорий по **первой букве** имени; заголовок секции — буква-разделитель (`A ────`): мелкий `muted-foreground` + тонкая линия `border`, sticky при скролле. Не-буквенные/цифровые имена — в бакет `#`.
  - **Строка категории — мини-карточка** (в духе кита, не голый текст): маленькая обложка (`expo-image`) + имя категории + счётчик «N courses» + `ChevronRight` (Lucide); фон `card`/`surface`, скругление. Тап → `category/{categoryId}` (§12.3в) в стеке этого же таба.
  - **Сортировка** — по имени категории (англ., `localeCompare`, регистронезависимо); `categories.sort_order` в индексе **не** используется (только алфавит). **Пустые категории** (0 опубликованных курсов) в индекс **не попадают**.
- **UiState:** `loading | content(sections) | empty | error`.
- **Данные:** `get_catalog` (§8.1) → `{ categories:[{ id, name, slug, courses_count }] }`, отсортировано по имени, только категории с ≥1 `is_published`-курсом. Группировку по буквам и формирование `sections` делает клиент.
- **Возврат/навигация:** корень таба — `catalog/feed`; `category` и `course_path` пушатся в стек Catalog, «Назад» возвращает на уровень выше (course_path → category → catalog/feed).
- **Состояния:** empty — «No categories yet»; error — ретрай.
- **Пост-MVP (не сейчас):** A-Z скраббер справа (быстрый перескок по буквам) и поиск по каталогу — их естественное место именно здесь.

### 12.4 Course Path (сетка уроков курса)

- **Назначение:** экран курса — **сетка карточек-уроков** (grid), каждая карточка = один `lesson` с обложкой (`cover_image_url`). Заменяет прежнюю «дорогу с нодами»: тот же экран, тот же роут `course_path/{courseId}`, тот же HUD сверху — меняется только раскладка (сетка вместо дороги). Фон экрана — цвет из `path_background_color`.
- **UiState:** `loading | content(course, lessons) | error`.
- **Данные:** `get_course_path(courseId)` → `lessons[]` со `status`, `cover_image_url`, `sort_order` + `course.is_completed`.
- **Раскладка:** вертикальный скролл, сетка из карточек по `sort_order` (число колонок — фиксированное, например 2; точное значение — деталь UI-фазы). Карточка — **вертикальная ~3:4** (портрет, ближе к 3:4, **не** «инстаграмные» 9:16) со скруглением `radiusLarge`, фон = обложка урока (`expo-image`, кэш/blurhash/placeholder; ошибка картинки → placeholder, не краш, A13). Подпись урока (`title`) — поверх/под обложкой по дизайну кита.
- **Плашка «Course completed» (сверху, D).** Если `course.is_completed=true` — над сеткой уроков показывается компактная **плашка-баннер** «Course completed» со значком `success` (Lucide `CircleCheck`), вписанная в дизайн-кит (фон `surface`/`success`-акцент). Так пользователь, открыв уже пройденный курс, сразу видит, что курс завершён; карточки уроков при этом доступны для перепрохождения.

**Состояния карточки урока (через токены `node-*`, §5.6; без хардкода цвета):**

1. **`completed` (пройден).** Обложка в **обычном (полном) цвете**, без бейджей и значков — «прошли и прошли». Тап → можно перепройти урок (`lesson/{id}`). *(Это про отдельный урок; пройденность всего курса показывает верхняя плашка выше.)*
1. **`active` (текущий) — статичное выделение, без пульсации (по дизайн-концепции).** Обложка в полном цвете + **сплошная акцентная рамка/кольцо** `node-active` вокруг карточки (статичная, **без анимации-пульса**). Выделение **локальное** — обводка по контуру карточки, без широкого свечения/тени, чтобы не «залезать» на соседние карточки уроков. Активная карточка — **единственная** с такой рамкой; понятно, какой урок проходим сейчас, но соседи не перекрываются. Тап → `lesson/{id}`.
1. **`locked` (будущий).** Поверх обложки — **нежёсткий desaturate-overlay** (обложку по-прежнему видно; `opacity` карточки **не меняем**). Внизу справа — иконка **`Lock`** (Lucide), наложена **прямо на слой обложки** (без кружка, без обводки, без подложки; цвет — токен `node-locked`/`muted`). **Неинтерактивна:** тап ничего не делает (не ведёт в урок).

- **Действия:** тап `active` → `lesson/{id}`; тап `completed` → `lesson/{id}` (перепрохождение); тап `locked` → нет реакции. **Кнопки «Restart» нет:** перепрохождение — это повторный заход в уже `completed`-урок; он даёт меньшую награду (`lesson_replay_diamond_reward`, §12.6/ВАЖН-15).
- **Состояния экрана:** loading — skeleton-сетка; empty — «No lessons yet»; error — ретрай. Ошибки изображений — placeholder (`expo-image`), не краш (A13).

### 12.5 Lesson (раннер заданий)

- **Назначение:** последовательное прохождение заданий; нижний HUD скрыт; вверху — **панель урока** (прогресс-бар + кнопка выхода), см. 12.5в. Экран открыт как `fullScreenModal`.
- **При входе:** вызывается `start_lesson(lesson_id)` (§8.1) — возвращает `resume_task_index` и лениво создаёт `user_course_progress`/`user_lesson_progress`. Раннер открывается **с задания `resume_task_index`** (продолжение, если урок был прерван), а не всегда с первого. `errors_count` сбрасывается только при свежем проходе/перепрохождении (§A7).
- **UiState:** `loading | running(task, index, total) | verdict | finished | exitConfirm`. Поле `index` инициализируется из `resume_task_index`.
- **Нижняя кнопка задания** (овальная; семантика `requires_check` — §6.5а): `requires_check=true` → сначала **disabled с подписью-действием**, после действия — активная **«Done»** (нажатие запускает проверку → вердикт, §12.5г). `requires_check=false` → **«Continue»** (для `reading_text` — только на последнем сегменте, для `reading_media`/`video` — сразу). Автопроверки нет.
- **Раскладка кнопки (канон, перенесено из §6.5в, НЕКР-5):** кнопка `Continue`/`Done` **никогда не перекрывает текст**. Помещается текст — он целиком над кнопкой; длинный текст → область **скроллится** (`ScrollView`), кнопка зафиксирована снизу (абсолютная, над safe-area), у нижней кромки — **fade-out градиент** (`expo-linear-gradient`), не резкий обрыв. Учитывать нижний инсет на обеих платформах.
- **Логика батарейки (плоская, §A4, КРИТ-1):** видео и чтение — бесплатно; тест (`requires_check=true`) — перед началом проверка `battery>0` (у не-подписчика), иначе → **`paywall` оверлеем** (§12.9). **Любой ответ** (верный/неверный) → `submit_task_attempt` → `battery −= battery_cost_per_answer` (сервер) → вердикт → следующее задание. **Внутриурочных бонусов и серий верных подряд нет; бонуса заряда за завершение урока тоже нет** (Gap #6) — экономика плоская, награда за урок только алмазами (§A4/§12.6). У подписчика батарея не тратится.
- **Закончился заряд во время урока (Gap #4):** при попытке начать тест с `battery=0` (не-подписчик) открывается `paywall`. **Если пользователь покупает подписку** → возвращаемся в урок и продолжаем **с того же задания** (заряд больше не нужен). **Если закрывает пейволл без покупки** → прогресс урока **уже сохранён** (`current_task_index`, §A7), возвращаемся в `course_path` (§12.9, Gap #3); пользователь дожидается восстановления батареи и позже **продолжает урок с нужного задания**.
- **Видео-задание:** воспроизводится через **общий плеер** (`features/player`, `expo-video`) с тап-управлением — тап по центру = пауза/play, перемотка перетаскиванием по прогресс-полосе (Gesture Handler; см. F6). «Просмотрено» при ~95% длительности или ручном Continue → `submit_task_attempt(is_correct=true)`.
- **Завершение:** прогресс-бар дошёл до конца → `complete_lesson` → `reward/{lessonId}` с результатом.

#### 12.5в Верхняя панель урока: прогресс-бар и выход

> Три элемента поверх раннера.

**1. Прогресс-бар урока (сверху).** Горизонтальная полоса, заполняется **по мере прохождения заданий**: `доля = пройдено / всего`. Каждое завершённое задание (Continue/Done) сдвигает полосу. Полоса считает **все** задания, включая видео.

- **Поведение на полноэкранном видео.** Панель (прогресс-бар + крестик) **скрыта во время воспроизведения** (иммерсивный режим). При **тапе по видео** (пауза) — панель появляется; при возобновлении — скрывается. Прогресс-счётчик не останавливается.

**2. Кнопка выхода (крестик, справа вверху).** Иконка `✕`. Тап → **немедленно останавливает любой звук** (TTS текущего сегмента или аудио видео) → открывается подтверждающая плашка (`exitConfirm`). Урок не прерывается — только звук на паузе.

**3. Подтверждающая плашка выхода (bottom sheet).** Выезжает снизу (`@gorhom/bottom-sheet`), контент затемняется. Звук на паузе, пока плашка открыта:

- **Заголовок:** `Exit lesson?`
- **Текст:** `Your progress is saved — you can continue this lesson later from where you left off.`
- **Кнопки (сверху вниз):**
  - **`Exit lesson`** — обычная (не деструктивная; токен `primary`/`foreground`, пилюля): выход подтверждён, звук прекращается. **Прогресс по уроку сохраняется** — `user_lesson_progress` остаётся `active`, `current_task_index` зафиксирован; следующий заход продолжит **с того же задания**. Навигация → назад на `course_path`.
  - **`Cancel`** — текстовая: закрывает плашку, **звук возобновляется**, возврат к текущему заданию.

> **Что значит «прогресс сохраняется» (Gap #3/#4).** Завершение урока атомарно (`complete_lesson` срабатывает только когда пройдены все задания, см. A7). Выход в середине: `user_lesson_progress` остаётся `active` с сохранённым `current_task_index` (§6.3/§A7); следующий вход возобновляет урок **с этого задания** (не с №1). **Важно:** батарея, списанная за ответы в этом проходе, **не возвращается** (списание уже зафиксировано на сервере), но уже отвеченные задания при возобновлении **повторно не оплачиваются**.

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

#### 12.5б Задание `reading_media` (иллюстрированное чтение) — детально

- **Назначение:** текст с опц. TTS, **без проверки**, кнопка **«Continue»**, **плюс одно медиа сверху**. Картинка с текстом и видео с текстом — один тип (`reading_media`), отличаются `media.kind`.
- **Раскладка (сверху вниз):**

1. **Медиа** — верхняя ~половина экрана. Картинка через `expo-image` (`media.image_url`); в видео-режиме (позже) — общий плеер (`features/player`).
1. **Нижняя панель** (`surface`, скруглённая сверху): **заголовок** → **текст разделов** (как в reading_text) → кнопка **«Continue»**.

- **Кнопка «Continue» — всегда активна сразу** (в отличие от reading_text).
- **Текст не перекрывается кнопкой** (раскладка-канон — §12.5): кнопка зафиксирована снизу (абсолютная, над safe-area) и не наезжает на текст. Длинный текст → область скроллится (`ScrollView`), текст уходит **под** кнопку; у кнопки **fade-out градиент** (`expo-linear-gradient`).
- **Аудио, разделы, деградация текста** — как в `reading_text` (12.5а), **кроме** появления кнопки (здесь сразу).
- **Деградация медиа:** битая картинка → placeholder на месте медиа-области, экран не падает (A13).
- **Рендерер:** `features/lesson/tasks/ReadingMediaTask` (переиспользует логику чтения из `ReadingTextTask`).

#### 12.5г Вердикт после проверки (верно/неверно) — *канон, КРИТ-7*

> Обратная связь на ответ в задании с проверкой (`requires_check=true`). Применяется после нажатия `Done` (`submit_task_attempt`).

- **Подсветка ответа.** Выбранный вариант/ячейка подсвечивается токеном `success` (верно) или `error` (неверно). При ошибке дополнительно показывается **правильный ответ** (подсветкой правильного варианта).
- **Баннер-плашка вердикта (снизу).** Появляется плашка: `Correct!` (токен `success`) или `Incorrect` + правильный ответ (токен `error`). Нижняя кнопка превращается в **`Continue`** → следующее задание.
- **Одна попытка.** Повторных попыток нет — после вердикта идём дальше (согласуется с §A7). Батарея уже списана **за любой** ответ (§A4), верный он или нет.
- **Хаптика и звук.** На вердикт — `expo-haptics` (success/error) и звук (если включены тумблеры, см. таблицу ниже). Управляются `Sound Effects` / `Vibration` из Preferences (§12.12).
- **UiState:** `verdict` (между `running` и переходом к следующему заданию).

**Таблица звук + хаптика (ВАЖН-14):**

|Событие                  |Звук|Хаптика     |Примечание                                   |
|-------------------------|----|------------|---------------------------------------------|
|Верный ответ             |да  |success     |плашка `Correct!`                            |
|Неверный ответ           |да  |error/warning|плашка `Incorrect` + правильный ответ        |
|Завершение урока         |да  |success     |переход на `reward`                          |
|Получение награды/алмазов |да  |light       |на экране `reward`                           |

> **`Sound Effects`** (Preferences) управляет **только** этими эффект-звуками; **TTS-озвучка чтения под него не попадает** (TTS читается всегда, если есть `audio_url`). **`Vibration`** управляет всей хаптикой. Источник звуков — бандл-ассеты приложения (`expo-audio`).

#### 12.5д Задание `matching` (сопоставление) — *канон, ВАЖН-12*

> Два столбца (например «личность ↔ год рождения»). Цвета «правильно»/«неправильно» — из палитры (§5, токены `success`/`error`).

- **Выбор пары.** Сначала тап по значению в **одном** столбце (любом — левом или правом) → элемент **подсвечивается** (выделение выбора). Затем тап по значению в **противоположном** столбце.
- **Верная пара.** Оба значения **подсвечиваются одинаковым «правильным» цветом** (`success`), затем тухнут — превращаются в **серые неактивные кнопки** (из дальнейшего выбора исключены).
- **Неверная пара.** Второе выбранное значение получает **«неправильный» цвет** (`error`), затем выделение снимается с обоих — оба снова доступны. Значение можно выбрать заново в паре с другим, чтобы собрать верную пару.
- **Завершение.** Когда все пары собраны верно — нижняя кнопка активна (`Done`/`Continue`) → следующее задание.
- **Рендерер:** `features/lesson/tasks/MatchingTask`.

#### 12.5е Задание `fill_blank` (вставь слово кирпичиками) — *канон, Gap #7*

> Заголовок задания — `payload`-подпись (англ.). Предложение с пропуском `___` сверху; **кирпичики-слова** (`payload.options`) — внизу, под предложением. Цвета «правильно»/«неправильно» — из палитры (`success`/`error`, §5).

- **Раскладка.** Предложение с **пустым местом** (слот пропуска) сверху; ряд **кирпичиков** (карточки-слова из `options`) — снизу, каждый на своём «домашнем» месте.
- **Заполнение — двумя способами:**
  - **Перетягиванием:** берём кирпич, перетаскиваем (Gesture Handler) в слот пропуска → слот заполнен.
  - **Тапом:** просто тапаем по кирпичу — он сам «подтягивается» в пропущенное место.
- **«След» кирпича.** На «домашнем» месте кирпича остаётся **след-контур** (пустой слот того же размера), чтобы кирпичу было **куда вернуться**.
- **Снять кирпич с места:** тап по уже поставленному кирпичу (в пропуске) — он возвращается на свой след внизу; пропуск снова пустой.
- **Проверка:** после установки кирпича жмём **`Done`**. Если ответ **верный** — поставленный кирпич подсвечивается **«правильным» цветом** (`success`); если **неверный** — **«неправильным» цветом** (`error`) + показывается правильный вариант (общий вердикт — §12.5г). Затем кнопка → `Continue`.
- **Рендерер:** `features/lesson/tasks/FillBlankTask`. *(В MVP — один пропуск на предложение; `payload.answer` — верное слово, `options` — кирпичи.)*

#### 12.5ж Задание `timeline` (хронология перетягиванием) — *канон, Gap #7*

> Заголовок (англ.) — `payload.prompt` (напр. «Sort these events in chronological order»). Несколько **карточек-событий** (`payload.items`, на каждой — текст, что произошло; `year` хранится в payload для проверки, но **не показывается**).

- **Задание.** Карточки событий перемешаны; их нужно **перетаскиванием** (drag&drop, Gesture Handler) расставить в **правильной последовательности**, затем нажать снизу **`Done`**.
- **Проверка по `Done`:**
  - **Порядок корректный** → ок, все карточки подсвечиваются `success`, кнопка → `Continue`.
  - **Порядок неверный** → приложение **автоматически переставляет карточки в правильный порядок** (по возрастанию `year`); при этом карточки, **стоявшие на правильных местах**, помечаются **«правильным» цветом** (`success`), а **стоявшие неверно** — **«неправильным» цветом** (`error`). После показа результата кнопка → `Continue`.
- **Рендерер:** `features/lesson/tasks/TimelineTask`.

#### 12.5з Задание `image_recognition` (узнай по картинке) — *канон, Gap #7*

> Заголовок (англ.) — `payload.prompt`. Сверху — **картинка** (`payload.image_url`), ниже — **варианты** (`payload.options`).

- **Задание.** Один вариант **тапом** выбирается (подсветка выбора — токен `selected`, §5.6), затем **`Done`**.
- **Проверка (§12.5г):** правильный вариант подсвечивается **«правильным» цветом** (`success`); при ошибке выбранный — `error`, плюс подсвечивается правильный. Кнопка → `Continue`.
- **Рендерер:** `features/lesson/tasks/ImageRecognitionTask`.

#### 12.5и Задание `multiple_choice` (вопрос + варианты) — *канон, Gap #7*

> Заголовок-вопрос (англ.) — `payload.prompt`, ниже — несколько **вариантов ответа** (`payload.options`), без картинки.

- **Задание.** Тапаем по варианту — он **выделяется «цветом для выделения»** (токен `selected`, §5.6; это **не** success/error — лишь выбор до проверки). Затем **`Done`** → приложение оценивает.
- **Проверка (§12.5г):** верный вариант — `success`; при ошибке выбранный — `error` + подсветка правильного. Кнопка → `Continue`. *(MVP — одиночный выбор по `answer_index`.)*
- **Рендерер:** `features/lesson/tasks/MultipleChoiceTask`.

#### 12.5к Задание `true_false` (да/нет, мгновенная оценка) — *канон, Gap #7*

> Заголовок-вопрос (англ.) — `payload.statement`. Две кнопки: **True / False** (да/нет).

- **Особенность — кнопки `Done` нет.** Как только пользователь нажал **одну** из двух кнопок — он **сразу оценивается** (`submit_task_attempt` + вердикт), без отдельного `Done`. Это исключение из общей схемы «disabled→Done» (§6.5а): для `true_false` кнопка-ответ **сама** является подтверждением.
- **Проверка (§12.5г):** нажатая кнопка подсвечивается `success` (верно) или `error` (неверно, плюс подсвечивается правильная). Сразу появляется баннер-вердикт и переход → `Continue` к следующему заданию.
- **Батарея/`errors_count`:** как у любого теста (`requires_check=true`) — списание за ответ, инкремент `errors_count` при ошибке (§A4/§6.5а).
- **Рендерер:** `features/lesson/tasks/TrueFalseTask`.

### 12.6 Reward

- **Назначение:** случайная Rive-анимация маскота + текст похвалы + начисленная награда + разблокировка следующей ноды. **Полноэкранное окно-оверлей поверх всего; закрывается тапом по окну** (Gap #8), без кнопки `Continue`. Данные — из ответа `complete_lesson` (§8.1).
- **Награда на экране — только алмазы (Gap #6):** показывается `+N 💎` (заряд за урок не начисляется и **не показывается**). **Перепрохождение** (`already_rewarded=true`): показывается малая фикс-награда `lesson_replay_diamond_reward` (дефолт 1 💎) + маскот/похвала. Взвешенный рандом при повторе не разыгрывается. Алмазы отражаются в HUD (счётчик обновляется).
- **Завершение курса (ВАЖН-7):** при `course_completed=true` Reward-окно дополняется блоком **«Course completed!»** + показ нового `knowledge_level` (если `leveled_up`). Отдельного экрана празднования курса в MVP нет (кандидат пост-MVP). Если одновременно `leveled_up=true` — за Reward **последовательно** показывается окно level-up (§12.0). После закрытия всех окон → `course_path` (все карточки в полном цвете, пройденный курс с верхней плашкой «Course completed», §12.4).
- **Текущее поведение анимации:** маскот играет случайную анимацию из **единого нейтрального пула** (без деления по качеству прохождения). **Урок как таковой не «провален/пройден»** (см. A7).
- **Реакция маскота на качество заданий — управляется флагом из админки (`app_config.economy.mascot_tone_reaction_enabled`, дефолт `false`; B6.8).** Когда флаг `true` **и** в установленной сборке есть анимации обоих пулов: тон выбирается по числу неверных ответов (`errors_count`) — пул **celebrate** (0/мало неверных) или **encourage** (больше неверных), по 10 анимаций в каждом пуле, выбор случайный. Когда флаг `false` **или** анимации пулов отсутствуют в сборке → играет нейтральный пул (**мягкий откат**, без краша). Сейчас флаг выключен (анимации ещё не нарисованы). Счётчик `errors_count` пишется на сервере независимо от флага (§A6/A7), поэтому фичу можно включить тумблером, как только нужная сборка с анимациями доедет до пользователей — без правок бэкенда.
- **Закрытие и очерёдность окон (Gap #8):** Reward — это **полноэкранное окно поверх всего**, снимается **тапом по нему** (а не кнопкой). Если за ним идёт level-up (§12.0), окна показываются **последовательно**: тап закрывает Reward → появляется level-up → тап закрывает его → `course_path`. **Два тапа закрывают оба окна; быстрый двойной тап — проскакивает оба сразу.** После последнего окна следующая нода в `course_path` уже `active`.

### 12.7 Quests — *удалён (квесты убраны из проекта целиком)*

> Квесты вырезаны полностью (глобальное решение): они не несли бизнес-логики. Нет блока `QUESTS`, нет вкладки/маршрута квестов, нет RPC `get_quests`/`claim_quest_reward`, нет таблиц `quests`/`user_quests`, нет фичи F14/фазы Phase 11. Алмазы зарабатываются за уроки и тратятся **только** на автозаморозку стрика (§A5). При возврате к фиче — заводить отдельной миграцией.

### 12.8 Subscription (вкладка 3, блок G)

- **Назначение:** **самостоятельная вкладка**, в которую пользователь заходит когда хочет (Gap #2): все планы (Super / Super Family, скруглённые блоки), их цены, какой план **сейчас** у пользователя, предложение покупки и предложение пробного периода. Текст CTA и наличие бесплатной недели зависят от `get_subscription_offer` (§A14).
- **Это канон содержимого подписки.** Пейволл (§12.9) показывает **ту же информацию в том же порядке**, но открывается оверлеем при `battery=0` во время задания — **не** из этой вкладки. Вкладка Subscription и пейволл — два входа к одному и тому же набору планов/цен/предложений с одной целью (оформление подписки).
- **Состояние для активного подписчика (`trial`/`active`, ВАЖН-8):** вместо CTA покупки — **бейдж активного плана** («You're Super» / «Super Family»), **дата** следующего списания/истечения (`expires_at`), кнопка **«Manage Subscription»** (ведёт в системные настройки стора — App Store/Play, как §12.12). Триал показывает «Trial until …» до `expires_at`. Кнопка апгрейда на Family — опционально (кандидат пост-MVP).
- **Данные:** локальная цена и пакеты — из **RevenueCat `Offerings`/`Packages`** (стор сам знает страну/валюту); тексты — из `price_tiers` для отображения; **предложение** (`trial_available`, `timer_*`) — из `get_subscription_offer` (server-authoritative); статус и `expires_at` — из `subscriptions`.
- **Условный CTA по предложению:**
  - `trial_available=true` → кнопка **«Try 1 week for $0»** (free-trial offer; на iOS — introductory offer, на Android — free-trial offer).
  - `trial_available=false` → кнопка обычной покупки (например **«Subscribe»** с ценой).
- **Таймер обратного отсчёта (если `timer_active=true`):** над карточками — **персональный счётчик** «Special price ends in 23:59:58», тикающий до `timer_ends_at` (тикает локально, но истина — `timer_ends_at` с сервера). Персональное окно на юзера. При `timer_expired=true` → счётчик исчезает, триал больше не предлагается.
- **Действие:** покупка → **RevenueCat `purchasePackage()`** → вебхук `revenuecat-webhook` обновляет `subscriptions` → (опц.) `validate-purchase` для немедленной реконсиляции. **Restore Purchases** — `restorePurchases()` (обязательно для обеих платформ).
- **Эффект подписки:** при `trial`/`active` батарея перестаёт тратиться, в HUD на месте батареи — ассет `app_config.subscription_badge` (§11, §A4). При отмене/истечении — батарея возвращается полной.

### 12.9 Paywall (оверлей)

- **Триггер — только `battery=0` во время задания (Gap #2).** Пейволл возникает, когда не-подписчик пытается начать тест, а заряда нет. Открывается **оверлеем поверх экрана урока** (`fullScreenModal`). **Из вкладки Subscription пейволл не открывается** — у неё свои inline-CTA (§12.8).
- **Контент = та же информация в том же порядке, что и на вкладке Subscription (Gap #2):** те же планы (Super / Super Family), цены, текущий план, предложение покупки и предложение триала. Зависит от `get_subscription_offer` (§A14): при `trial_available=true` — «Buy subscription. First week — $0»; иначе обычная покупка. При `timer_active=true` — тот же персональный обратный отсчёт. Покупка — через RevenueCat (как §12.8). Цель та же — оформление подписки.
- **Закрытие и возврат (Gap #3/#4):**
  - **Куплено** → пользователь стал подписчиком (заряд бесконечен) → возвращаемся **в урок** и продолжаем **с того же задания** (`current_task_index`, §A7).
  - **Закрыто без покупки** (крестик ✕) → возвращаемся **в `course_path`** (курс), а **не** наружу. Прогресс урока **сохранён** (`current_task_index`): когда батарея восстановится, пользователь снова заходит в урок и **продолжает с нужного задания**. «Не хочешь платить — мы сохранили твой прогресс, дождись заряда и иди дальше» (§A4/§A7).

### 12.10 Shop — *удалён (КРИТ-8)*

> Магазин заморозки стрика убран целиком (глобальное решение ревизии). Заморозка теперь автоматическая: при пропуске дня сервер сам списывает алмазы (`streak_freeze_cost_per_day`, §A5). Блока `SHOP`, экрана `shop` и RPC `buy_streak_freeze` больше нет.

### 12.11 Profile (вкладка 4)

- **Назначение:** личный кабинет. Лента блоков (server-driven, §11): только **`PROFILE`** (инфо профиля). Шестерёнка ⚙️ вверху справа → `settings`. *(Квестов нет — убраны целиком; ачивок нет — убраны, ВАЖН-16; блока `SHOP` нет — убран, КРИТ-8.)*
- **Блок `PROFILE`:** базовая информация профиля — `display_name` / `username` / `knowledge_level` (без аватара — убран, НЕКР-17). **HUD не дублируется** — level/streak/diamonds/battery показывает общий HUD-каркас сверху (§12.0), он виден и здесь; отдельной копии в контенте экрана нет.
- **UiState:** `loading | content(profile) | error`.
- **Данные:** профиль — репозиторий профиля.

### 12.12 Settings и подразделы

> Полная структура — concept 1.18а. Реализуется как список → подэкраны.

- **Preferences:** Sound Effects / Vibration (Haptics) / Daily Practice Reminders (тумблеры, хранятся в MMKV; reminder включает локальное уведомление `expo-notifications`). **`Sound Effects`** управляет эффект-звуками урока (НЕ TTS), **`Vibration`** — всей хаптикой — перечень событий в таблице §12.5г. Vibration → `expo-haptics` (кросс-платформенно).
- **Profile:** Name / Username / Email + **Password** (редактирование) + кнопка **DELETE ACCOUNT** (→ `delete_account`; обязательно для Apple и Google). **`Password` = смена пароля** — пункт виден **только для аккаунтов с email-методом** (ВАЖН-5); для чистого соц-входа (только Google/Apple) пункт скрыт. Аватара нет (НЕКР-17).
- **Course:** список начатых/завершённых курсов + удалить (→ `reset_course_progress`, с подтверждением).
- **Account Linking (ВАЖН-6):** тумблеры **Google** + **Apple** (iOS) + статус **Email**. Бэкенд — **Supabase Identity Linking** (manual linking включён в настройках проекта): привязка соц-метода → `supabase.auth.linkIdentity({ provider })`; отвязка → `supabase.auth.unlinkIdentity(identity)`. Привязка **email** к соц-аккаунту → `supabase.auth.updateUser({ email })` + подтверждение кодом. **Правило «нельзя отвязать единственный способ входа»** enforce-ится и в UI (кнопка отвязки задизейблена, если метод последний), и серверно (`unlinkIdentity` не удаляет последнюю identity). Отвязка соц-входа требует предварительной привязки email (диалог). Подробнее — §A16.
- **Documents:** Terms / Privacy Policy (in-app, `react-native-webview` или нативный текст).
- **Support:** Help Center / Send Feedback (in-app). Кнопки **Choose / Restore Subscription** (RevenueCat), **LOG OUT**. На iOS — ссылка «Manage Subscription» ведёт в системные настройки App Store; на Android — в Play подписки.

-----

## 13. Спецификации фич с критериями приёмки

> Сжатый формат: **AC** = acceptance criteria. Тестовые требования вынесены в отдельный документ.

**F1. Авторизация.** AC: вход через Google создаёт профиль с 300 алмазами; **на iOS доступен «Sign in with Apple»**; сессия сохраняется (шифрованно, secure-store) и переживает перезапуск на обеих платформах; logout очищает сессию; **email-регистрация = email + пароль + код подтверждения** (`signUp` + email confirm, ВАЖН-5); пункт «Password» в Settings виден только для email-аккаунтов; Account Linking — через Supabase `linkIdentity`/`unlinkIdentity` (§A16).

**F2. Батарейка.** AC: тест при `battery=0` (у не-подписчика) блокируется и открывает paywall; **любой ответ (верный/неверный) уменьшает батарею на 1 на сервере**; **бонуса заряда за завершение урока нет** (Gap #6 — награда за урок только алмазами, §A4/§12.6); видео и чтение не тратят батарею; восстановление по конфигу, не выше `battery_max`; HUD красит иконку красным при ≤3; **у подписчика батарея не тратится и в HUD заменена ярким ассетом подписки**, после отмены — батарея возвращается полной. Все числа настраиваются в админке. Поведение идентично на iOS и Android.

**F3. Алмазы.** AC: **первое** завершение урока начисляет случайную сумму по весам (`10`/`15`/`20`, 10 — чаще всего) ровно один раз на урок (идемпотентно по леджеру); **перепрохождение** даёт фикс `lesson_replay_diamond_reward` (дефолт 1, из конфига); сумма разыгрывается на сервере; баланс меняется только через RPC; автосписание за заморозку стрика (150 алмазов/день, из конфига) не уводит баланс в минус.

**F4. Стрик.** AC: ≥1 урок в день увеличивает стрик; **заморозка автоматическая за алмазы** — при пропуске сервер списывает `streak_freeze_cost_per_day` за каждый пропущенный день, пока хватает баланса, и сохраняет стрик; если алмазов не хватает — стрик сбрасывается; день засчитывается по `last_activity_date < today`; стрик пересчитывается при заходе (`refresh_user_state`, КРИТ-8). Магазина заморозок и ручной покупки нет.

**F5. Прогресс/разблокировка.** AC: следующий урок разблокируется только после `complete_lesson` на сервере; клиент не может разблокировать локально; прогресс-бар курса = `lessons_completed / total`.

**F6. Видео.** AC: видео стартует на низком качестве почти мгновенно (HLS, `expo-video`); следующий ролик прелоадится; показывается постер вместо чёрного экрана; повтор играет из кэша. **Управление (общий плеер):** тап по центру = пауза/play (иконка ▶️ при паузе); перемотка перетаскиванием по прогресс-полосе; (опц.) двойной тап = ±10 сек. Работает одинаково на AVPlayer (iOS) и ExoPlayer (Android).

**F7. Маскот-награда.** AC: после урока показывается случайная анимация маскота, каждый раз потенциально разная (`rive-react-native`); текст похвалы; показ начисленных алмазов; **урок не помечается как успешный/неуспешный**. **Реакция по качеству заданий (celebrate/encourage по `errors_count`) управляется флагом `app_config.economy.mascot_tone_reaction_enabled` (дефолт `false`, тумблер в админке B6.8):** при `true` и наличии анимаций обоих пулов в сборке — тон по `errors_count`; при `false` или отсутствии анимаций — нейтральный пул (мягкий откат, без краша). `errors_count` пишется независимо от флага (§A7). Сейчас флаг выключен (анимации не нарисованы).

**F8. Подписка.** AC: покупка проходит через **RevenueCat** на **обеих платформах** (Google Play Billing + Apple StoreKit); статус приходит вебхуком в `subscriptions` и подтверждается `validate-purchase`; **Restore Purchases** работает; локальная цена соответствует стране пользователя (стор определяет автоматически через RevenueCat Offerings). **Предложение управляется `get_subscription_offer` (§A14):** при включённом триале CTA даёт бесплатную неделю (Android free-trial offer / iOS introductory offer), при выключенном — обычную покупку; персональный таймер (если включён) показывает обратный отсчёт и по истечении убирает бесплатную неделю для этого юзера; триал и таймер включаются независимо из админки.

**F9. Server-driven UI.** AC: дефолтный конфиг встроен (`default_ui_config.json`) и **страхует отказ загрузки UI-конфига** (когда сеть в целом есть), а **не** полный оффлайн-старт — при отсутствии сети показывается заглушка + Retry (ВАЖН-4, §12.1); активный конфиг применяется по `min_app_version` (build-номер, см. A11); отключённый блок/вкладка не отображается; неизвестный тип блока не роняет приложение.

**F10. Региональные цены.** AC: пользователь видит цену своей страны (стор определяет автоматически через RevenueCat); тексты-заглушки до загрузки берутся из `price_tiers`/`country_price_map`.

**F11. Настройки.** AC: все тумблеры сохраняются (MMKV); DELETE ACCOUNT удаляет данные и аккаунт (Apple + Google требование); Course-сброс удаляет прогресс; Account Linking не даёт отвязать единственный способ входа; управление подпиской ведёт в системные настройки соответствующего стора.

**F12. Виджеты.** AC: виджеты показывают напоминание/стрик и открывают приложение по тапу — **в MVP диплинк ведёт на Home** (`lorebinge://`); вариант «последний курс» (`lorebinge://course/{slug}`) — пост-MVP (НЕКР-16). **На Android через Glance, на iOS через WidgetKit** (≥2 размера на каждой платформе). Данные виджету поставляются единым мостом из приложения (shared storage / App Group на iOS).

**F13. Аналитика/краши.** AC: ключевые события логируются (lesson_completed, paywall_shown, subscription_started, streak_lost) на обеих платформах (Firebase Analytics); краши уходят в Crashlytics; **пойманные клиентские ошибки** (сетевые сбои, RPC-ошибки, ошибки покупок) репортятся через `crashlytics().recordError(e)`; **серверные ошибки** (Edge Functions, RPC) пишутся в таблицу `error_logs`.

**F14. Quests — *удалена (квесты убраны из проекта целиком).*** Фича вырезана: нет блока Quests, RPC `get_quests`/`claim_quest_reward`, таблиц `quests`/`user_quests`, фазы Phase 11. Алмазы зарабатываются за уроки и тратятся только на автозаморозку стрика (F4). При возврате к фиче — заводить заново.

