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

