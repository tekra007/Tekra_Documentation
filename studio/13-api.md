# 13. HTTP API

> Чернетка, уточниться при реалізації. Переписано 2026-10-05 під ТЗ і D31: API
> **двох сервісів** — Studio (редагування) і оркестратора (виконання,
> погодження, адміністрування). Між сервісами HTTP немає — лише брокер
> ([28-studio-service.md](28-studio-service.md)). Що йде по WebSocket —
> [09-protocol.md](09-protocol.md).

## Хто що обслуговує

Кожен сервіс — на **своєму домені** (D82):

| Сервіс | Хто викликає | Що там |
|--------|--------------|--------|
| Studio-back | лише `Tekra-Studio-front` | створення й редагування роботів, версії, компоненти, процеси, каталог, перевірка |
| оркестратор | `Tekra-Studio-front` і `Tekra-Orchester-Face` | реєстр роботів і їхні стани, запуски, машини, облікові дані, погодження, Test Period, аудит, політики, розклади |

Studio-front ходить в обидва сервіси (D31, варіант II); Face — лише в
оркестратор. CORS: оркестратор приймає домени обох фронтів, Studio-back — лише
Studio-front. Адреси API фронти читають під час запуску (`config.json`).

## Наскрізні правила

- **Вхід один:** токен видає оркестратор. Studio перевіряє підпис сама —
  публічним ключем, без звернення до оркестратора (D31).
- **Організація й роль — з токена**, ніколи з URL. Клієнт не називає, від чийого
  імені він діє.
- **Права** — за [24-rbac.md](24-rbac.md), перевіряються на сервері, а не
  хованням кнопок. Studio перевіряє свої дозволи (`ROBOT_CREATE`, `ROBOT_EDIT`,
  `VERSION_CREATE`, `VERSION_EDIT`), оркестратор — свої.
- **Ідентифікатори в адресах:** робот — UUID (`/robots/3f2a…`), версія — номер у
  межах робота (`/versions/2`). Людям показуємо `RBT-184` і `184.2` (D61, D62).
- **Пагінація** — як уже склалося в проєкті (`GET /logs`):
  `{ total_pages, current_page, pagination, items, counts }`.
- **Аудит:** кожна зміна стану в оркестраторі пише аудит у тій самій
  транзакції (D41). Studio аудит не пише — її зміни йдуть подіями через outbox,
  і аудит пише оркестратор (D67).
- **Сповіщення адмінам** — у оркестраторі, через `AdminNotificationService`, для
  змін, які мають бачити адміни (AGENTS.md, 5a).
- Шляхи у фронті згруповані по доменах (`ROBOT_URLS`, `EXECUTION_URLS` …) і в
  сервісах не хардкодяться — як уже склалося.

---

## Studio-back

### Каталог активностей

| Метод | Шлях | Що робить |
|-------|------|-----------|
| `GET` | `/activities` | увесь каталог (D2): `catalog_version`, `features`, `activities[]` (`name`, `provider`, `title`, `min_assistant_version`, `inputs[]` з `type` / `required` / `required_when` / `default` / `options`, `output`, `blocks[]`). Дані з `src/catalog/activities.json` (36-phase-4); `Cache-Control` — година |

### Роботи: створення й редагування

| Метод | Шлях | Що робить |
|-------|------|-----------|
| `POST` | `/robots` | створити: `{ name, description, owner_account_id }` → `{ id, number }`. Номер видає Studio (D68) |
| `PATCH` | `/robots/{id}` | назва, опис, власник |
| `GET` | `/robots/{id}/working-copy` | робоча копія + від якої версії відкрита (`working_copy_base_number`) |
| `PUT` | `/robots/{id}/working-copy` | зберегти робочу копію — це **ще не версія** (D34) |
| `POST` | `/robots/{id}/validate` | перевірити сценарій `{ definition? }` (порожньо — робоча копія) → `{ errors, warnings, unchecked, min_assistant_version }` ([14-validation.md](14-validation.md)); кожна знахідка — `code`, `message`, `step_id`, `field`; `unchecked` — що не вдалося перевірити без оркестратора (D68). Редактор Studio викликає через 600 мс після зміни |

Список роботів і картку зі станами браузер бере з оркестратора — там реєстр.

### Версії

| Метод | Шлях | Що робить |
|-------|------|-----------|
| `POST` | `/robots/{id}/versions` | **«Зберегти»** з робочої копії: якщо `definition` змінився — нова версія, **201**; ні — повертає наявну, **200** (D34). → `{ number, registered, created }`. Сценарій не за форматом або перевірка знайшла **помилки** — **422** `{ detail, errors }` (`loc`, `msg`, `type`, `step_id`, `field`); попередження не заважають; копія порожня — 400. У версії зберігається `min_assistant_version`. Оркестратор не чекає (D68) |
| `GET` | `/robots/{id}/versions` | список: номер, автор, дата, опис змін, `registered` |
| `GET` | `/robots/{id}/versions/{number}` | версія з повним `definition` |
| `PATCH` | `/robots/{id}/versions/{number}` | лише `change_description` (`VERSION_EDIT`); сама версія незмінна |

`registered` — чи вже зареєстрував версію оркестратор. Запустити можна лише
зареєстровану (D68).

### Компоненти (D58, D65)

| Метод | Шлях | Що робить |
|-------|------|-----------|
| `GET` | `/components` | бібліотека; фільтри: пошук, архівні |
| `POST` | `/components` | створити → `{ id, number }` |
| `PATCH` | `/components/{id}` | назва, опис, власник |
| `GET`, `PUT` | `/components/{id}/working-copy` | робоча копія |
| `POST` | `/components/{id}/versions` | зберегти версію |
| `GET` | `/components/{id}/versions`, `/components/{id}/versions/{number}` | версії |
| `GET` | `/components/{id}/usage` | які роботи й версії використовують |
| `POST` | `/components/{id}/archive` | у архів; версії не видаляються |

### Процеси (D57, D65)

| Метод | Шлях | Що робить |
|-------|------|-----------|
| `GET` | `/processes` | список |
| `POST` | `/processes` | створити → `{ id, number }` |
| `PATCH` | `/processes/{id}` | назва, опис, власник |
| `GET`, `PUT` | `/processes/{id}/working-copy` | BPMN XML у редакторі |
| `POST` | `/processes/{id}/versions` | зберегти версію |
| `GET` | `/processes/{id}/versions`, `/processes/{id}/versions/{number}` | версії: BPMN XML і задачі |
| `POST` | `/processes/{id}/archive` | у архів |

Здоровʼя процесу Studio не рахує: екран бере Health роботів процесу з
оркестратора (D31).

---

## Оркестратор

### Роботи: реєстр і стани

| Метод | Шлях | Що робить |
|-------|------|-----------|
| `GET` | `/robots` | список; фільтри: `number`, пошук, Lifecycle, «потребує уваги» (D59), архівні |
| `GET` | `/robots/{id}` | картка: усі виміри окремо — Lifecycle, Authorization, Availability, Health, Active Production Version, останній запуск |
| `POST` | `/robots/{id}/move-to-test` | DRAFT → TEST — явна дія розробника (ТЗ §5, крок 2) |
| `POST` | `/robots/{id}/move-to-prod` | `{ version_number, comment }` → PENDING_APPROVAL або активовано за політикою (§4.1.15) |
| `POST` | `/robots/{id}/rollback` | `{ version_number, reason }` (D53) |
| `POST` | `/robots/{id}/availability` | `{ action: enable \| disable \| suspend \| resume, reason }` (D50) |
| `POST` | `/robots/{id}/archive`, `/robots/{id}/restore` | архів і відновлення в TEST (D55) |
| `DELETE` | `/robots/{id}` | лише адмін, лише заархівованого (D55) |

Створити й перейменувати робота тут не можна — це робить Studio, а оркестратор
отримує подію (D31).

### Версії в реєстрі

| Метод | Шлях | Що робить |
|-------|------|-----------|
| `GET` | `/robots/{id}/versions` | версії з **станами**: Release State, Authorization, «активна» |
| `GET` | `/robots/{id}/versions/{number}` | версія; `definition` — копія, з якої виконують |
| `POST` | `/robots/{id}/versions/{number}/revoke` | відкликати погодження (`PROD_REVOKE`), `{ reason }` (§4.1.7) |

### Доступ до робота

| Метод | Шлях | Що робить |
|-------|------|-----------|
| `GET` | `/robots/{id}/access` | кому дозволено запускати в PROD |
| `POST` | `/robots/{id}/access` | дати доступ юзеру |
| `DELETE` | `/robots/{id}/access/{userId}` | забрати |

### Запуски

| Метод | Шлях | Що робить |
|-------|------|-----------|
| `POST` | `/executions` | запустити: `{ robot_id, version_number?, machine_id, lifecycle, input, scope? }` |
| `GET` | `/executions` | список; фільтри: робот, версія, машина, TEST / PROD, статус, дата |
| `GET` | `/executions/{id}` | картка |
| `GET` | `/executions/{id}/logs` | покрокові логи, пагіновано |
| `POST` | `/executions/{id}/cancel` | зупинити акуратно |
| `POST` | `/executions/{id}/pause`, `/resume`, `/step` | налагодження в TEST (D66) |

**Фаза 1 (L4, 2026-11-07):** `POST /executions { robot_id, version_number }` — лише
TEST, машина — та, де автор зараз увійшов в Assistant (D89), тому `machine_id` і
`lifecycle` не передаються. Відповідь 201 — запуск `QUEUED` (відправлений —
`dispatched_at`) або `BLOCKED` з `block_reason`; 403 — роль не може TEST (№14), 404 —
немає робота / версії (№1–3), 400 — автор не увійшов у жоден Assistant; відмови
теж в аудиті. `GET /executions?robot_id&limit` (новіші першими), `/executions/{id}`,
`/executions/{id}/logs?after_seq&limit`. Налагодження — пізніше.

**Фаза 5 ([37](37-phase-5.md), 2026-11-08):**

- `POST /executions { robot_id, version_number, input }` — `input` — значення
  параметрів версії (D22). Невідоме ім'я, відсутнє обов'язкове, не той тип, текст
  довший за 10 000 символів — **400** «Wrong parameters: …» (запуску немає, слід в
  аудиті). Порожні беруть `default`; у запуску зберігається повний `input`.
- `POST /executions/{id}/cancel` — `EXECUTION_CANCEL`: свої запуски, адмін — будь-які
  (чужий — 403). Не відправлений — одразу `CANCELLED`; відправлений / RUNNING —
  `cancel_requested_at`, агент зупиняється на межі кроку й звітує `CANCELLED`.
  Повтор — 200 без змін; завершений — 409.
- У `ExecutionSchema` нові поля: `input`, `waiting_reason` (на що чекає запуск у
  черзі), `connection_lost_at` (зв'язок з агентом обірвався, D51),
  `cancel_requested_at`.
Наживо (L5): `/me/ws` — підписка на `execution_id`, події `execution_updated` /
`execution_logs`, автору — `version_registered`; формат — [09-protocol.md](09-protocol.md).

- `version_number` для **TEST** обовʼязковий — запускається конкретна
  зареєстрована версія (D68). Для **PROD** не передається: виконується Active
  Production Version (перевірка №5).
- `scope` — лише для TEST: точки зупинки або виділені кроки (D66).
- Запуск — окремий ресурс, а не `POST /robots/{id}/run`: у нього своє життя,
  статус і логи, його показують списком і фільтрують.

### Машини

| Метод | Шлях | Що робить |
|-------|------|-----------|
| `GET` | `/machines` | список машин: номер, назва, ОС (D76); лише адмін (24-rbac) |
| `GET` | `/machines/{id}` | картка; попередження, якщо `MachineGuid` повторюється (D77) |
| `PATCH` | `/machines/{id}` | назва |
| `GET` | `/assistants` | список: машина, хто зараз увійшов (`signed_in_account_id`), версія, `capabilities`, дозволені середовища; лише адмін |
| `PATCH` | `/assistants/{id}` | `allowed_environments` — непорожня підмножина `TEST` / `PROD`, лише адмін (D76); аудит `ASSISTANT_UPDATED`, сповіщення `INFO`; черга машини перевіряється знову ✅ фаза 5 |

Створення немає навмисно: машина зʼявляється сама, коли агент підключається
(D10).

### Inspector

| Метод | Шлях | Що робить |
|-------|------|-----------|
| `POST` | `/inspector-sessions` | `{ provider }` (типово `desktop-uia`) → сесія (`id`, `assistant_id`, `status`, `expires_at`); машина — де розробник сам увійшов в Assistant (фаза 3); право `ROBOT_EDIT`; немає Assistant / не вміє провайдера / не на зв'язку — 400; результат приходить по `/me/ws` |
| `DELETE` | `/inspector-sessions/{id}` | скасувати свою сесію — 204; Assistant отримує `inspector/cancel`; чужа — 404 |

Оркестратор результат лише пересилає в браузер і не зберігає; браузер кладе
елемент у робочу копію через Studio (D31).

### Розклади

| Метод | Шлях | Що робить |
|-------|------|-----------|
| `GET` | `/schedules` | список — лише читання |

Розклади будуються з тригерів активної PROD-версії (D58). Змінити розклад —
змінити робота й зберегти нову версію.

### Облікові дані

| Метод | Шлях | Що робить |
|-------|------|-----------|
| `GET` | `/credentials` | список — **без секретів**: назва, логін, «лише одна сесія» |
| `POST` | `/credentials` | створити |
| `PATCH` | `/credentials/{id}` | назва, логін, `single_session` (D69) |
| `PUT` | `/credentials/{id}/secret` | змінити пароль |
| `DELETE` | `/credentials/{id}` | видалити |
| `GET` | `/credentials/{id}/usage` | які роботи й версії ним користуються |

Пароль не повертається **жодним** ендпоінтом (D20). `usage` — щоб не видалити
те, чим користуються.

**Фаза 4 (E4, 2026-11-07):** усе з таблиці реалізовано. Права: `GET` і `usage` —
`CREDENTIAL_VIEW` (адмін, розробник), решта — `CREDENTIAL_MANAGE` (адмін). Пароль
шифрується ключем організації (`OrganizationSecrets`). Назва в організації
унікальна — 409; `DELETE` того, що згадує хоч одна версія робота, — 409 зі списком
роботів; чужа організація — 404. Аудит `CREDENTIAL_*`, сповіщення адмінам (`INFO`).
`single_session` зберігається, але блокування між запусками (D69) — фаза 5.

### Погодження (D38)

| Метод | Шлях | Що робить |
|-------|------|-----------|
| `GET` | `/approvals` | список; фільтри: тип (`PROD_PROMOTION` / `TEST_PERIOD`), статус |
| `POST` | `/approvals/{id}/approve` | `{ reason? }` |
| `POST` | `/approvals/{id}/reject` | `{ reason }` |
| `POST` | `/approvals/{id}/cancel` | скасувати власний запит |

Наявні `GET /approvals/pending` і `POST /approvals/bulk-approve` лишаються як
частина цього набору. У картці `PROD_PROMOTION` — попередження, якщо агент на
PROD-машинах застарий (D72).

### Тестові періоди (D37)

| Метод | Шлях | Що робить |
|-------|------|-----------|
| `GET` | `/test-periods` | список |
| `POST` | `/test-periods` | запросити (розробник) або видати одразу (адмін) |
| `POST` | `/test-periods/{id}/revoke` | відкликати достроково |

### Аудит (D41–D43)

| Метод | Шлях | Що робить |
|-------|------|-----------|
| `GET` | `/audit-events` | фільтри: робот, версія, дія, людина, дата, `correlation_id`; розробнику — обмежено (D43) |

### Політики (D44)

| Метод | Шлях | Що робить |
|-------|------|-----------|
| `GET` | `/policies` | поточні значення — адмін і розробник |
| `PATCH` | `/policies` | змінити — лише адмін |

---

## Що вже є в коді оркестратора і що з ним станеться

`routers/robot.py` існує вже зараз:

| Наявний ендпоінт | Доля |
|------------------|------|
| `GET /robots/user/{user_id}` | → `GET /robots` |
| `DELETE /robots/user/{user_id}/robot/{robot_id}` | → `DELETE /robots/{id}` |
| `POST .../start`, `.../pause`, `.../stop` | → `POST /executions`, `/executions/{id}/pause`, `/cancel` |
| `POST .../schedule-robot` | → розклад із тригерів версії (D58) |
| `GET /robots/robot/{robot_id}/kpi` | рахується з `executions` |

Прод-даних немає, тож старі шляхи прибираються, а не підтримуються паралельно.

**Принцип змінюється:** наявні шляхи носять `user_id` у URL
(`/robots/user/{user_id}/robot/{robot_id}/start`). Нові — ні: організація й
роль беруться з токена.

**Розклад переїжджає на сервер.** Сьогодні він живе на десктопі
(`RobotScheduler` у `robot_layout.py`), а сервер лише проксіює команду по
WebSocket. Через це розклад зникає при перевстановленні застосунку, а адмін не
бачить, що й коли запускається. За D28 і D58 розклад — серверний, з тригерів
активної PROD-версії; десктоп лише виконує запуски, що приходять.
