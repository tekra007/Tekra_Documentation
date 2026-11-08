# 39. Фаза 6 — шлях у PROD: задачі

> Розбивка фази 6 з [10-roadmap.md](10-roadmap.md). Складено 2026-11-08 після фази 5
> ([37](37-phase-5.md)) і повернення функцій десктопа ([38](38-desktop-return.md)).
> Рішення ухвалено того ж дня (людина доручила агенту вирішувати самому). Підстави —
> D36–D40, D44–D46, D50–D56, D72, [17-authorization](17-authorization.md),
> [18-version-release](18-version-release.md), [20-policies](20-policies.md),
> [22-availability](22-availability.md), [23-rollback-archive](23-rollback-archive.md),
> [24-rbac](24-rbac.md).
>
> **Мета фази 6:** робот проходить шлях ТЗ: TEST → Move to PROD → погодження адміна
> (якщо політика вимагає) → активна PROD-версія → PROD-запуски з усіма 17 перевірками;
> відкликання, rollback, вимкнення / призупинення, Health, архів; Regular User
> запускає PROD-роботів, до яких має доступ.

## Що вже є

Енуми станів (`RobotLifecycle`, `VersionReleaseState`, `VersionAuthorization`,
`RobotAvailability`), колонки `robots.lifecycle / availability`, `robot_versions.
release_state / authorization`, усі права D56 і матриця ролей, DRAFT → TEST і
DRAFT → TESTING (фаза 1), черга з PROD першим (фаза 1), скелет `approvals`
(`DEPLOYMENT`, bulk-approve без змін об'єктів).

## Етапи

Фаза велика — робимо її чотирма етапами, кожен закінчується живим прогоном:

| Етап | Що | Чому в такому порядку |
|------|----|-----------------------|
| **6A** | політики організації; Move to PROD; погодження PROD (approve / reject / cancel); Release State і миттєва активація (D39, D40); PROD-запуски з перевірками №5, 6, 13, 16-PROD, 17; Studio-front — версії й Move to PROD; Face — сторінка Approvals | серцевина шляху: без неї решта не має сенсу |
| **6B** | відкликання (revoke), rollback і «вперед» (D53), перестворення черги на нову версію і попередження про застарілих агентів (D72), Test Period (D37) і №9 | спирається на активну версію з 6A |
| **6C** | Availability (D50) і що стається із запущеними (D51), Health (D52), архів / відновлення / видалення (D55) | стани поверх готового шляху |
| **6D** | доступ до роботів (`robot_access`, D25) і Regular User: бачить і запускає PROD своїх роботів у Face і в Assistant (Robots) | останнє: потрібна активна версія й права |

## Рішення по відкритих питаннях

| # | Питання | Рішення | Чому |
|---|---------|---------|------|
| 1 | Health — фаза 5 чи 6 | 6C | у roadmap — фаза 6; потрібні PROD-запуски |
| 2 | Посилання на людей у нових таблицях | `*_account_id` (D83), як уже в коді | `admins` / `users` прибрано в E2 |
| 3 | `POST /approvals/bulk-approve` (кнопка «Approve all» у дзвіночку з Figma) | лишається, але **кожен** запит погоджується тим самим шляхом, що й один: повний аудит, активація, D72. Запити з попередженням про застарілих агентів bulk **пропускає** й повертає їх у `skipped` — їх адмін дивиться окремо | кнопка є в макеті; D41/D42 — аудит на кожну зміну стану; попередження D72 має бути побачене |
| 4 | Хто читає погодження | `GET /approvals`: адмін — усі (`PROD_APPROVE` або `TEST_PERIOD_APPROVE`); розробник — свої запити (статус, причина) | D43: автор бачить, що з його запитом |
| 5 | Доступ USER до роботів | `robot_access` у 6D — частина «RBAC повністю» | без нього USER не бачить жодного робота |
| 6 | Попередження D72 про застарілих агентів | за Assistant із `PROD` у `allowed_environments` (D76 замінив призначення машини D70) | у коді є лише середовища Assistant |
| 7 | Тайм-аути й «один PROD-робот» | колонки `organization_policies` (D44); читаються в момент дії (D45); константи лишаються значеннями за замовчуванням | D48, D49, D51 називають їх політиками |
| 8 | REJECT, коли активної версії немає | lifecycle повертається PROD → TEST (аудит `ROBOT_LIFECYCLE_CHANGED`) — інакше робот у PROD, якого не можна запустити й не можна тестувати як треба | ТЗ не забороняє; стан «PROD без жодної версії» безглуздий |
| 9 | PROD-запуск — де | на машині, де людина зараз увійшла в Assistant, і цей Assistant дозволяє `PROD` (D76); версія — **лише активна** (№5); `version_number` у запиті не потрібен | вибору машини адміном немає (адмін не запускає, D56); розклад — фаза 7 |
| 10 | «Automation defaults» у Face | поза фазою 6, лишаються як є | окреме рішення (20-policies) |
| 11 | Повторна подача відхиленої (D54) | `authorization` PENDING_APPROVAL, `release_state` лишається REJECTED; UI показує «на погодженні» | так у D54 |
| 12 | Тип погодження | `DEPLOYMENT` → `PROD_PROMOTION`, новий `TEST_PERIOD` (6B); підпис картки в дзвіночку лишається як у Figma | D38; макет — правда для тексту картки |

## Задачі 6A

| ID | Що | Де |
|----|----|----|
| **P1** ✅ | `organization_policies` (D44): таблиця + міграція + RLS, рядок створюється з організацією (і для наявних — міграцією), `GET` / `PATCH /policies` (`POLICY_VIEW` / `POLICY_EDIT`), аудит `POLICY_CHANGED`; виконання читає тайм-аути й «один PROD-робот» звідси | Orchester |
| **P2** ✅ | Погодження за D38: колонки `approvals` (`robot_id`, `version_number`, `test_period_id`, `requested_by_account_id`, `request_comment`, `decision_reason`, статус `CANCELLED`, тип `PROD_PROMOTION`); `robots.active_production_version_number`; `robot_versions.released_by_account_id / released_at`; нові `AuditAction` | Orchester |
| **P3** ✅ | `POST /robots/{id}/move-to-prod {version_number, comment}`: lifecycle → PROD; політика не вимагає — RELEASED / NOT_REQUIRED і активація; вимагає — PENDING_APPROVAL і запит у `approvals`; сповіщення адмінам `ACTION`; попередження про застарілих агентів у відповіді | Orchester |
| **P4** ✅ | `GET /approvals`, `POST /approvals/{id}/approve | reject {reason} | cancel`; approve — RELEASED / APPROVED, активація, попередня — SUPERSEDED, 4 рядки аудиту з одним `correlation_id`; reject — REJECTED, і якщо активної немає — lifecycle → TEST; bulk-approve — через той самий шлях | Orchester |
| **P5** ✅ | `GET /robots/{id}/versions` (стани кожної версії для Studio й Face) | Orchester |
| **P6** ✅ | PROD-запуск: `POST /executions { robot_id, lifecycle: "PROD", input }` → активна версія; перевірки №4–6, 13, 16-PROD (політика `prod_single_instance`), 17; `runnable` для розробника — і TEST, і PROD | Orchester |
| **P7** ✅ | Studio-front: версії з Release State / Authorization, «Move to PROD» з коментарем і попередженням, стан погодження, «Run in PROD» для активної | Studio-front |
| **P8** ✅ | Face: сторінка Approvals (список, погодити / відхилити з причиною, попередження D72), дзвіночок веде туди; Settings → Policies (`PATCH /policies`) | Face |
| **P9** ✅ (див. «Живий прогін 6A») | Живий прогін 6A: робот → Move to PROD → адмін погоджує у Face → PROD-запуск із Assistant | усі |

## Живий прогін 6A (2026-11-08)

Стек dev з кодом 6A; міграцію `d3a9f1c7e284` накочено на dev-БД (`alembic check`
чистий, downgrade / upgrade без помилок). Політики — типові (PROD потребує
погодження).

| Що | Результат |
|----|-----------|
| Studio (розробник): RBT-26, версія 26.1 «TESTING» → Move to PROD з коментарем | «Version 26.1 is waiting for an admin…», версія «WAITING FOR APPROVAL», робот одразу в PROD |
| Face (адмін): Approvals → Pending | запит RBT-26.1, хто й коли, коментар; Approve → «APPROVED, Approved by admin@tekra.local» |
| Після погодження | 26.1 — RELEASED / APPROVED, активна; `GET /robots/runnable` пропонує її як PROD |
| Studio: «Run in PROD» з параметром `who = Petro` | PROD-запуск 26.1 `COMPLETED`, тестове вікно отримало `LOGIN Petro 7` |

Знайдено й виправлено під час прогону: у списку версій Studio кнопки «Move to PROD» /
«Cancel request» переносились у вузьку першу колонку сітки й виглядали як текст —
стан і дії тепер в одній клітинці.

Не перевірено живцем (лише тестами на справжній БД): відхилення й повернення робота в
TEST, повторна подача, скасування запиту, bulk-approve з пропуском, «один PROD-робот».

## Етап 6B — рішення й задачі

| # | Питання | Рішення | Чому |
|---|---------|---------|------|
| 1 | Відкликання (revoke) | адмін (`PROD_REVOKE`), причина обов'язкова: `authorization` → REVOKED; якщо версія активна — вказівник знімається, робот без активної версії → TEST (рішення 8); PROD-запуски цієї версії: у черзі → BLOCKED, ті, що йдуть, — зупинка на найближчій межі кроку | D51: «REVOKED → PROD-запуски відкликаної версії обриваються»; Assistant уміє зупинятись лише між кроками — найближче до «обірвати» без сміття посеред кліку |
| 2 | Rollback і «вперед» (D53) | розробник (`ROBOT_MOVE_TO_PROD`) або адмін (`PROD_APPROVE`), причина обов'язкова; ціль — SUPERSEDED з APPROVED / NOT_REQUIRED, без нового погодження; робот → PROD; запуски заміненої версії, що йдуть, — зупинка; у черзі — перестворюються на нову (D72); аудит `VERSION_ROLLED_BACK` + активація | D53 |
| 3 | Активація після погодження / без нього | PROD-запуски попередньої версії в черзі: `CANCELLED` «замінено на …» і **одразу** новий запуск на новій версії з тими самими параметрами, машиною, автором і `requested_at` (`replaced_execution_id`, один `correlation_id`); ті, що йдуть, — доробляють | D72 |
| 4 | Тестовий період: хто й як | розробник просить (`TEST_PERIOD_REQUEST`) тривалість із пресетів політики (або свою, якщо дозволено, ≤ максимуму) — лише коли політика вимагає (інакше 400); запит іде в Approvals (тип `TEST_PERIOD`); адмін погоджує / відхиляє або видає сам без запиту (`TEST_PERIOD_APPROVE`), відкликає (`TEST_PERIOD_REVOKE`) | D37, 17-authorization |
| 5 | Статус «відкликав сам» | `CANCELLED` — додано до статусів періоду (у D37 його немає) | запит, який розробник забрав, не «відхилений» адміном |
| 6 | Один період | один REQUESTED або чинний на робота; другий — 409 | D37: «один чинний на робота» |
| 7 | Кінець періоду | планувальник при кожному проході: APPROVED з `valid_to` у минулому → EXPIRED (аудит SYSTEM); політика `TERMINATE` — TEST-запуски робота зупиняються, `ALLOW_TO_FINISH` — доробляють; те саме при відкликанні | D37, D44 |
| 8 | Перевірка №9 | коли політика вимагає — TEST-запуск лише з чинним періодом (за часом, не за статусом), при запиті й перед відправкою; немає — BLOCKED | D48, D37 |

| ID | Що | Де |
|----|----|----|
| **Q1** ✅ | міграція: `test_periods` (+RLS), `approvals.test_period_id` (рівно одне з версії / періоду), `executions.replaced_execution_id`, нові дії аудиту | Orchester |
| **Q2** ✅ | revoke + rollback + D72 (перестворення черги, зупинка запусків) | Orchester |
| **Q3** ✅ | тестові періоди: API, погодження через Approvals, видача адміном, відкликання, закінчення в планувальнику, №9 | Orchester |
| **Q4** ✅ | Studio-front: Rollback у списку версій, тестовий період (стан, запит) | Studio-front |
| **Q5** ✅ | Face: сторінка Robots — версії (Revoke), тестові періоди (видати / відкликати); Approvals показує запити `TEST_PERIOD` | Face |
| **Q6** ✅ (див. «Живий прогін 6B») | живий прогін 6B | усі |

Як зроблено в UI:

- **Studio-front** (сторінка робота): «Rollback» біля SUPERSEDED-версії з APPROVED / NOT_REQUIRED
  (право `ROBOT_MOVE_TO_PROD`), причина обов'язкова, попередження «PROD-запуски активної
  зупиняться, ті, що чекають, перейдуть на цю версію». Картка «Test Period» (право
  `TEST_PERIOD_REQUEST`): політика вимкнена — «Not needed»; чинний — до коли; запит —
  «WAITING FOR APPROVAL» і «Cancel request»; немає — пресети з `GET /policies` (≤ максимуму),
  свої години, якщо дозволено, коментар, «Request Test Period». Сторінка робота тепер
  вантажиться ліниво (`loadComponent`) — початковий бандл 518 → 420 kB, у межах бюджету.
- **Face**: сторінка Robots — список із розгортанням: версії (Revoke…, Roll back to this…),
  тестові періоди (Grant, Revoke…), усі дії з обов'язковою причиною; Approvals показує
  запити `TEST_PERIOD` як «Test Period · N h».

## Живий прогін 6B (2026-11-08)

Стек dev із кодом 6B (міграція `e6b2d8f4a017`), Assistant на машині розробника,
сценарій через API (адмін — токен Face, розробник — токен Studio), робот RBT-26.

| Що | Результат |
|----|-----------|
| Політика `test_approval_required = true`, TEST-запуск без періоду | `BLOCKED` «The robot has no Test Period in force» (№9) |
| Розробник просить період 1 год; другий запит | `REQUESTED` з `approval_id`; другий — 409 |
| Адмін погоджує запит `TEST_PERIOD` | період APPROVED, `in_force`, `valid_to` = +1 год; TEST-запуск 26.2 `COMPLETED` |
| Адмін відкликає період | REVOKED; наступний TEST-запуск знову `BLOCKED` |
| 26.2 → Move to PROD → погодження | 26.2 активна, 26.1 SUPERSEDED / APPROVED |
| PROD-запуск 26.2 іде, розробник робить Rollback на 26.1 | запуск `CANCELLED` «Stopped: rolled back to version 26.1» на межі кроку; 26.1 активна |
| Адмін відкликає активну 26.1 | робот → TEST, активної немає; PROD-запуск — 400 «no active PROD version» |
| Rollback «вперед» на 26.2 з TEST | робот → PROD, 26.2 активна; PROD-запуск `COMPLETED` («Hello Petro, code 7») |
| Аудит | `TEST_PERIOD_CREATED / APPROVED / REVOKED / CANCELLED`, `VERSION_PROMOTED … PRODUCTION_ACTIVATED`, `VERSION_ROLLED_BACK`, `AUTHORIZATION_REVOKED`, `ROBOT_LIFECYCLE_CHANGED` — кожна зміна стану записана |

Не перевірено живцем (лише тестами на справжній БД): перестворення запуску **в черзі** на
нову версію (D72) — у живому стеку другий PROD-запуск при «одному PROD-роботі» отримує
`BLOCKED`, а не чергу, а машина одна; закінчення періоду планувальником і політика
`TERMINATE`. Кнопки Studio / Face цього етапу перевірені лінтом, типами, юніт-тестами й
збіркою, не кліками.

