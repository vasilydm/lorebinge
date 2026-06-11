# 33 · Реализация (Приложение A) — Lorebinge SPEC (RN9)

> Часть разбитого спека. Источник-архив: [`spec-rn9.md`](spec-rn9.md) — не редактировать. Содержит: Приложение A — версии зависимостей, секреты, подписание, механики (батарея/стрик/алмазы), биллинг, DoD, триал. Оглавление, граф зависимостей и карта перекрёстных ссылок — в [00-index.md](00-index.md).

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
- **Никакого бонуса заряда за завершение урока (Gap #6).** Экономика плоская: только трата за каждый ответ + восстановление по времени. **Награда за урок — только алмазы** (взвешенный рандом, §A6), отражаются в HUD. Поле `battery_reward_per_lesson` **удалено** из конфига; `complete_lesson` заряд не доначисляет; на экране Reward заряд **не показывается** (§12.6). Это усиливает точку монетизации: завершив урок, пользователь не получает «добавки» заряда — следующий урок упрётся в paywall тем быстрее.
- **Стартовое значение:** `battery = battery_max = 10` при регистрации (полная) — ровно на один урок из ~10 тест-заданий. Второго урока подряд бесплатно не хватает (это и есть точка монетизации).
- **Восстановление — “ленивое” (lazy), по факту обращения, не cron:**
  - Восстановление выполняется в `refresh_user_state` (бывш. `refill_battery`, §8.1/КРИТ-8), который вызывается при: открытии приложения (cold start), открытии Home, попытке начать тест. Та же функция лениво переоценивает стрик (§A5).
  - Формула: `ticks = floor((now − last_battery_refill_at) / battery_refill_minutes)`; `new_battery = min(battery_max, battery + ticks * battery_refill_amount)`; если `ticks > 0` → `last_battery_refill_at += ticks * battery_refill_minutes` (не `now`, чтобы не терять остаток времени).
  - `battery_refill_minutes = 60` → один заряд каждый час → **полные 10 зарядов за 10 часов**; за сутки ≈ 24 заряда ≈ **2 урока бесплатно в день** (сознательный выбор экономики).
  - **Если уже `battery_max` (ВАЖН-9):** держим `last_battery_refill_at = now()` (сдвигаем при каждом refill-проходе при полной батарее) — отсчёт восстановления начинается **с момента первой траты**, а не с накопленного времени. Так первый тик после траты приходит честно через `battery_refill_minutes`, а не почти мгновенно. (Зафиксированный вариант; альтернатива «не двигать таймер» отклонена.)
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
- **Разрыв и автозаморозка за алмазы (КРИТ-8):** заморозка **только автоматическая, за алмазы** — магазина и ручной покупки нет (`buy_streak_freeze`/`apply_streak_freeze` удалены, `streaks.freezes_available` убран). При обнаружении пропуска сервер за **каждый** пропущенный день списывает `streak_freeze_cost_per_day` алмазов (дефолт 150, настраивается в админке), пока хватает баланса, сохраняя стрик и записывая ledger (`reason='streak_freeze'`). Как только алмазов на следующий пропущенный день не хватает — стрик сбрасывается в 0 (`longest` сохраняется). Списание идёт по порядку дней; баланс не уходит в минус.
- **Ленивая переоценка при заходе (КРИТ-8):** стрик пересчитывается не только в `complete_lesson`, но и **при старте приложения** — одним вызовом `refresh_user_state` (батарея + стрик, §8.1), по аналогии с восстановлением батареи. Так HUD при входе не показывает устаревший `current_streak`, даже если пользователь несколько дней не заходил.
- **«Прошло менее 24 часов»** трактуем как календарный день (UTC), а не скользящие 24ч.

## A6. Начисление алмазов за урок — взвешенный рандом (Phase 8)

> Алмазы даются **за завершение каждого урока, независимо от правильности ответов** (уроки не делятся на успешные/неуспешные — успешными бывают только задания внутри урока, см. A7). Сумма — случайная по весам. **Логика серверная.**

**[РЕШЕНИЕ]**

- При `complete_lesson` начисляется **случайная сумма** из `app_config.economy.lesson_diamond_rewards` — список `{amount, weight}` (дефолт: `10` вес 70, `15` вес 25, `20` вес 5 → **10 чаще всего, 15 реже, 20 реже всего**). И суммы, и веса редактируются в админке.
- **Розыгрыш — на сервере** (Postgres `random()` по нарастающей сумме весов), клиент влиять не может. EV при дефолтах ≈ 11.75 алмаза/урок.
- **Идемпотентность первого прохождения:** результат фиксируется записью в `diamond_transactions` (`reason='lesson_completion_reward'`, `ref_id = lesson_id`). Взвешенный кубик разыгрывается **один раз на урок** и не перекатывается.
- **Перепрохождение (ВАЖН-15):** если урок уже `completed`, `complete_lesson` начисляет фикс `lesson_replay_diamond_reward` (дефолт **1**, настраивается в админке; `reason='lesson_replay_reward'`) и возвращает `already_rewarded=true`. Взвешенный рандом при повторе не разыгрывается. (Заряд за завершение не начисляется никому — §A4.) Анти-фарм не цель (NFR/НЕКР-15).
- `diamonds_awarded` в ответе `complete_lesson` = выпавшая (первое) или `lesson_replay_diamond_reward` (повтор).
- **Уровень (ВАЖН-3):** при завершении **последнего** урока курса `courses_completed += 1`; `knowledge_level = 1 + floor(courses_completed / courses_per_level)` (дефолт `courses_per_level=5`). Если уровень вырос → `leveled_up=true` (клиент показывает окно level-up, §12.0).
- `errors_count` урока на размер награды **не влияет** — это вход для тона маскота. Полное поведение реакции — каноном в **§12.6** (НЕКР-2); счётчик пишется на сервере независимо от флага `mascot_tone_reaction_enabled`.

## A7. Завершение урока: что значит «все задания пройдены» (Phase 6/8)

> Проблема: урок засчитывается, даже если на тесты ответили неверно? (Да, тест необязательно правильный.) Нужно явно.

**[РЕШЕНИЕ]**

- Урок завершается, когда пользователь **прошёл все задания по порядку до конца** (дошёл до последнего и нажал «завершить»), **независимо от правильности** ответов. Видео нужно досмотреть/проскипать до конца просмотра.
- **Уроков «успешных/неуспешных» не существует.** Успешным/неуспешным бывает только **отдельное задание-тест** (`requires_check=true` → верно/неверно). Урок не «проваливается» из-за неверных ответов: он либо завершён (все задания пройдены), либо нет. `errors_count` — это просто счётчик неверных тест-ответов в уроке (для тона маскота, F7), а не оценка урока.
- `complete_lesson` проверяет, что для всех `tasks` урока есть хотя бы одна запись в `user_task_attempts` ИЛИ задание не требует ответа (video — отметка о просмотре; `reading_text`/`reading_media` — отметка о прохождении по «Continue»). Для контентных заданий (`requires_check=false`) вводим «attempt» с `is_correct=true` при «Continue» / достижении конца.
- При `battery=0` пользователь **не может начать тест-задание** → не может дойти до конца урока с тестами → урок не завершится без батареи/подписки. Это и есть точка монетизации.
- **Прогресс урока сохраняется и возобновляется (Gap #3/#4).** Сервер хранит позицию внутри урока — `user_lesson_progress.current_task_index` (§6.3). `submit_task_attempt` продвигает индекс на следующее задание; при выходе в середине урока (крестик/«Назад») или при окончании заряда статус остаётся `active`, индекс **сохраняется** — следующий заход открывает урок **с этого же задания** (не с начала). Уже отвеченные задания повторно не оплачиваются (батарея за них уже списана и не возвращается). `start_lesson(lesson_id)` (§8.1) при входе возвращает `resume_task_index` и лениво создаёт `user_course_progress`/`user_lesson_progress`. **`errors_count`** сбрасывается в 0 только при **свежем** проходе (первый заход в `active`-урок с `current_task_index=0` или перепрохождение `completed`-урока), а при **возобновлении** — сохраняется, чтобы тон маскота отражал весь логический проход.
- **Момент завершения контролирует клиент** (осознанное упрощение, анти-чит не цель): `complete_lesson` вызывается, когда пройдено последнее задание. Кнопки «Restart» нет — перепрохождение = повторный заход в `completed`-урок (он сбрасывает `current_task_index`/`errors_count` в 0) с уменьшенной наградой (§A6).

## A8. Воспроизведение видео: учёт «просмотрено» и интеграция Cloudflare (Phase 6/7)

> Проблема: как фиксируем выполнение видео-задания; какой формат ссылки от Cloudflare Stream; нужен ли подписанный URL. **HLS работает на обеих платформах нативно** (AVPlayer на iOS, ExoPlayer на Android) через `expo-video`.

**[РЕШЕНИЕ]**

- Cloudflare Stream отдаёт HLS-манифест вида `https://customer-<code>.cloudflarestream.com/<videoUID>/manifest/video.m3u8`. Храним `cloudflare_video_id` (UID) и собираем/храним `hls_url` в `media_assets`. HLS — единый кросс-платформенный формат, отдельные сборки под iOS/Android не нужны.
- **Доступ к видео:** на старте — публичные (не signed) URL. Signed URLs / allowed origins — улучшение на будущее (TODO).
- **«Просмотрено»:** видео-задание считается выполненным при достижении ~95% длительности ИЛИ ручном «Continue» после окончания (через `onPlaybackStatusUpdate`/события `expo-video`). Тогда вызывается `submit_task_attempt(task_id, is_correct=true)`.
- Постер (`poster_url`) — генерируемый Cloudflare thumbnail (`.../thumbnails/thumbnail.jpg`), задаётся как `posterSource` плеера.

## A9. — намеренно пропущен

> Раздел A9 удалён при ревизии RN (НЕКР-7). Нумерация сохранена, чтобы перекрёстные ссылки не сбивались — искать потерянный раздел не нужно.

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

- Юзер уже использовал триал (`trial_used=true`, ставится `revenuecat-webhook` на событии `trial_started`, ВАЖН-10) → `trial_available=false`. Дублируем проверкой на сервере (стор и так не даст второй free trial на тот же аккаунт — ни Google, ни Apple). *(Поле переименовано из `trial_offered`; семантика — «триал использован», а срочность покрывают `offer_timer_started_at` + `timer_expired`.)*
- Смена `timer_duration_hours` в админке **не** перезапускает стартовавшие таймеры.
- Часы/время — `timestamptz` UTC; обратный отсчёт от серверного `timer_ends_at`, клиент не доверяет локальным часам для логики (только для отрисовки).

## A15. Принудительное обновление приложения / kill switch (Phase 4)

> Проблема (ВАЖН-13): server-authoritative приложение с эволюционирующими RPC-контрактами не имеет способа сказать старой сборке «обновись». Сломав совместимость RPC, мы получим крэши/ошибки у не обновившихся пользователей. Стандартный блок для такого класса приложений.

**[РЕШЕНИЕ]**

- Конфиг-ключ `app_config.min_supported_build` (int; §6.4). Дефолт `0` (выключено).
- На **Splash** после загрузки конфига сравниваем с `expo-application` → `nativeBuildVersion` (синхронный номер сборки обеих платформ, §A11).
- Если `nativeBuildVersion < min_supported_build` → **блокирующий полноэкранный экран** «Please update the app» с кнопкой в стор (две ссылки по платформе: App Store / Google Play). Дальше приложение не пускает.
- Поднимается администратором при выкатке несовместимого изменения RPC. Заводится в Phase 4 вместе с загрузкой конфига.

## A16. Account Linking — привязка/отвязка способов входа (Phase 12)

> Проблема (ВАЖН-6): UI Account Linking описан (§12.12), но бэкенд-контракт отсутствовал. Решение по best practices Supabase.

**[РЕШЕНИЕ] — Supabase Identity Linking (manual linking).**

- В настройках Supabase-проекта включить **Manual Linking** (`Enable manual linking`).
- **Привязка соц-метода** (Google/Apple): `supabase.auth.linkIdentity({ provider })` — добавляет identity к текущему пользователю.
- **Отвязка:** `supabase.auth.unlinkIdentity(identity)` — где `identity` получен из `supabase.auth.getUserIdentities()`.
- **Привязка email** к чисто соц-аккаунту: `supabase.auth.updateUser({ email })` + подтверждение кодом (email confirm). После этого в Settings появляется пункт «Password» (смена пароля, §12.12/ВАЖН-5).
- **Правило «нельзя отвязать единственный способ входа»** — enforce-ится двусторонне: (1) в UI кнопка отвязки задизейблена, если у пользователя ровно одна identity; (2) серверно `unlinkIdentity` сам не удаляет последнюю identity (проверить поведение и опираться на него). Отвязка соц-входа требует, чтобы уже была привязана email-identity (диалог-подсказка).
- Фронтенд (§12.12) приведён в соответствие с этим решением: тумблеры Google/Apple вызывают `linkIdentity`/`unlinkIdentity`, статус Email — из `getUserIdentities()`.

-----

*Конец Приложения A. Все решения — рабочие дефолты, разблокирующие разработку; заказчик может скорректировать любой пункт.*

