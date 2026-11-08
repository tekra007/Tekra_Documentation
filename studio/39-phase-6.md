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
| **P1** | `organization_policies` (D44): таблиця + міграція + RLS, рядок створюється з організацією (і для наявних — міграцією), `GET` / `PATCH /policies` (`POLICY_VIEW` / `POLICY_EDIT`), аудит `POLICY_CHANGED`; виконання читає тайм-аути й «один PROD-робот» звідси | Orchester |
| **P2** | Погодження за D38: колонки `approvals` (`robot_id`, `version_number`, `test_period_id`, `requested_by_account_id`, `request_comment`, `decision_reason`, статус `CANCELLED`, тип `PROD_PROMOTION`); `robots.active_production_version_number`; `robot_versions.released_by_account_id / released_at`; нові `AuditAction` | Orchester |
| **P3** | `POST /robots/{id}/move-to-prod {version_number, comment}`: lifecycle → PROD; політика не вимагає — RELEASED / NOT_REQUIRED і активація; вимагає — PENDING_APPROVAL і запит у `approvals`; сповіщення адмінам `ACTION`; попередження про застарілих агентів у відповіді | Orchester |
| **P4** | `GET /approvals`, `POST /approvals/{id}/approve | reject {reason} | cancel`; approve — RELEASED / APPROVED, активація, попередня — SUPERSEDED, 4 рядки аудиту з одним `correlation_id`; reject — REJECTED, і якщо активної немає — lifecycle → TEST; bulk-approve — через той самий шлях | Orchester |
| **P5** | `GET /robots/{id}/versions` (стани кожної версії для Studio й Face) | Orchester |
| **P6** | PROD-запуск: `POST /executions { robot_id, lifecycle: "PROD", input }` → активна версія; перевірки №4–6, 13, 16-PROD (політика `prod_single_instance`), 17; `runnable` для розробника — і TEST, і PROD | Orchester |
| **P7** | Studio-front: версії з Release State / Authorization, «Move to PROD» з коментарем і попередженням, стан погодження, «Run in PROD» для активної | Studio-front |
| **P8** | Face: сторінка Approvals (список, погодити / відхилити з причиною, попередження D72), дзвіночок веде туди; Settings → Policies (`PATCH /policies`) | Face |
| **P9** | Живий прогін 6A: робот → Move to PROD → адмін погоджує у Face → PROD-запуск із Assistant | усі |
