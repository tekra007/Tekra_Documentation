# 16. Трасування: ТЗ ↔ наші рішення

> ТЗ [`spec/tz-lifecycle.md`](spec/tz-lifecycle.md) — **затверджений документ і
> канон** (D32). Цей файл показує, як решта розділу `studio/` лягає на ТЗ: що
> збігається, що переглядається, що доведеться спроєктувати заново і як ми
> відповідаємо на 28 питань, які ТЗ свідомо лишив відкритими.
>
> Посилання виду **§4.1.4** ведуть на розділи ТЗ.

## 1. Словник: терміни ТЗ стають головними

| Термін ТЗ | Що означає | Як було в нас | Як стає |
|-----------|------------|---------------|---------|
| **Execution** | один запуск конкретної Version | Job, таблиця `jobs` | Execution, таблиця `executions` (D35) |
| **Assistant** | десктоп-застосунок, що виконує | Agent — назва програми | Assistant |
| **Host** | машина, на якій виконується | Machine | запис у `machines` = один Assistant на одному Host |
| **Lifecycle** | DRAFT / TEST / PROD / ARCHIVED | — | нове поле робота |
| **Authorization** | чи дозволено виконання | — | новий вимір робота й версії |
| **Availability** | чи дозволено виконання **зараз** | частково `is_archived` | ENABLED / DISABLED / SUSPENDED |
| **Health** | технічний стан | — | HEALTHY / WARNING / ERROR / OFFLINE |
| **Release State** | стан версії | — | DRAFT / TESTING / RELEASED / SUPERSEDED / REJECTED |
| **Active Production Version** | яка версія зараз у PROD | `published_version_id` | `active_production_version_id` |
| **Test Period** | дозвіл тестувати протягом часу | — | нова сутність |
| **Process** (BPMN) | бізнес-процес із Task-ів людини, робота, ролі | прибрали в D29 | повертається як Level 1 |
| **Component** | цеглинка робота; буває складена користувачем | Activity | узгодити в D6 під §3.2–3.3 |

## 2. Що збігається — не чіпаємо

| ТЗ | Наше рішення |
|----|--------------|
| Robot + незмінні Version (§4.1.3) | D1, крок 1 моделі даних |
| Active Production Version — вказівник, а не статус (§4.1.10) | `published_version_id` |
| Rollback = зміна вказівника без нової версії (§4.1.11) | `POST /robots/{id}/versions/{v}/activate` |
| Execution — окрема сутність, посилається на Robot + Version (§4.1.16–17) | `jobs.robot_version_id` |
| Assistant не обходить Orchestrator (§2.3, §4.8) | D5 — усе через оркестратор |
| Один Assistant для TEST і PROD (§4.8) | один десктоп-застосунок |
| Regular User / Developer / Admin (§4.4) | D23 — `USER` / `DEVELOPER` / `ADMIN` |
| Code — поза scope (§3.2) | D1 — JSON-DSL, не код |
| Scheduling в оркестраторі (§2.2) | D28 |
| Admin Panel в оркестраторі (§4.7.4) | Face — адмін-консоль оркестратора |
| Оркестратор фізично одна інсталяція (§4.7) | D31 — не розділяти |

## 3. Що переглядається під ТЗ

| Наше | Як було | Що каже ТЗ | Стан |
|------|---------|------------|------|
| Крок 1 моделі даних | чернетка живе на роботі, версія зʼявляється лише кнопкою «Опублікувати» | нова Version на **кожне збереження**, що впливає на виконання (§4.1.4); тестують версії (§4.1.13) | ✅ D34 — робоча копія; версія на «Зберегти» або перед запуском; без змін — без нової версії |
| Крок 3 моделі — статуси запуску | PENDING / SENT / RUNNING / SUCCEEDED / FAILED / STOPPED / CANCELED | QUEUED / RUNNING / PAUSED / COMPLETED / FAILED / CANCELLED (§4.1.18) + BLOCKED (§4.7.2) | ✅ D35 — статуси ТЗ + BLOCKED; SENT → поля `dispatched_at` / `delivered_at` |
| D11 — машина офлайн | ручний запуск одразу падає, без черги | черга QUEUED; не пройшла перевірка → BLOCKED із причиною в Audit (§4.7.2) | 🟡 пропозиція D48 — [21](21-queues.md) |
| D28 — накладання за розкладом | новий запуск пропускається | черги, пріоритет PROD над TEST (§4.7.3) | 🟡 пропозиції D47, D49 — [21](21-queues.md) |
| D25 — матриця прав | права за ролями + `robot_access` | Visibility (§4.4) окремо від Permissions (§4.5) | ⚠️ розширюється; фінальну RBAC ТЗ лишив відкритою (#14) |
| D29 — назви в UI | `Processes` → `Robots` | BPMN-процес і Robot — різні рівні (§3.2) | ⚠️ «Роботи» лишаються, «Процеси» як BPMN повертаються |
| D14 — валідація при публікації | перевірки перед публікацією | + 17 перевірок **перед кожним запуском** (§4.7.2) | доповнюється, не конфліктує |

### Борг у коді, який уже зараз суперечить ТЗ

- **Дашборд Face:** `RobotStatusKey = 'RUNNING' | 'PAUSED' | 'ERROR'`
  (`src/app/types/dashboard.types.ts`). ТЗ §4.1.16 прямо забороняє статус
  RUNNING у робота — це статус запуску.
- **Заглушка сторінки роботів:** після перейменування за D29 у ній замінено
  підказку «моніторинг бізнес-процесів (BPMN)». За ТЗ BPMN — окремий рівень,
  тож розділ процесів доведеться повернути.

Код поки не чіпаємо — фіксуємо, щоб не загубилось.

## 4. Нове з ТЗ — проєктуємо з нуля

| Що | Розділ ТЗ | Масштаб | Що вже є в коді |
|----|-----------|---------|-----------------|
| TEST / PROD як логічні контексти, окремі черги | §3.1, §4.7 | великий | — |
| Approval на PROD, Move to PROD | §4.1.7, §4.1.15 | великий | скелет `approvals`: `ApprovalSource.STUDIO`, `ApprovalType.DEPLOYMENT`, PENDING / APPROVED / REJECTED |
| Test Period + поведінка після закінчення | §4.1.12–14 | середній | — |
| Audit з «було → стало» | §4.7.5 | середній | `system_logs` і `admin_notifications` цього не закривають |
| Pre-run checks, стан BLOCKED | §4.7.2 | середній | — |
| Пріоритет і черги | §4.7.3 | середній | APScheduler |
| Availability, Health | §4.1.8–9 | малий | — |
| Release State, Source Version, Definition Hash, Released By/At | §4.1.5 | малий | — |
| Owner, Updated By, Robot ID виду `RBT-000184` | §4.1, §4.1.1 | малий | — |
| TEST → PROD interactions policy | §4.6 | середній | — |
| Admin Panel: налаштування політик | §4.7.4 | середній | Face — адмін-консоль |
| **BPMN (Level 1)** | §2.1, §3.2 | великий, мало деталей | — (D33) |
| **Custom components** | §3.3 | великий, мало деталей | — (D33) |

## 5. 28 відкритих питань ТЗ — стан у нас

Легенда: ✅ ми вже відповіли · 🟡 частково · 💡 сам ТЗ підказує відповідь · ⬜ відкрито

| № | Питання ТЗ | Стан | Що маємо |
|---|------------|------|----------|
| 1 | Які зміни є metadata-only | 🟡 | D34: усе поза `definition` нову версію не створює; лишається зафіксувати склад `definition` |
| 2 | Чи можуть одночасно існувати кілька APPROVED PROD Versions | ⬜ | — |
| 3 | Чи може одночасно виконуватися кілька Versions одного Robot’а | 🟡 | пропозиція D47: так, на різних машинах; для PROD — обмеження D49 |
| 4 | Approval flow для rollback | ⬜ | — |
| 5 | Хто має право ініціювати rollback | ⬜ | — |
| 6 | Чи потребує rollback окремого approval | ⬜ | — |
| 7 | Чи можна відновлювати ARCHIVED Robot’а | ⬜ | — |
| 8 | Семантика READ / CALL / WRITE / TRIGGER | ⬜ | відкладено за D46 — до першого способу викликати іншого робота |
| 9 | Чи може TEST викликати PROD виключно через Orchestrator API | 💡 | §4.6: TEST Robot не повинен обходити Orchestrator; наш D5 — усе через оркестратор. Деталі — за D46, разом із викликом робота |
| 10 | Умови автоматичного SUSPENDED | ⬜ | — |
| 11 | RUNNING Execution після DISABLED | ⬜ | можна взяти модель ALLOW_TO_FINISH / TERMINATE з §4.1.14 |
| 12 | RUNNING Execution після SUSPENDED | ⬜ | те саме |
| 13 | RUNNING Execution після REVOKED | ⬜ | те саме |
| 14 | Фінальна RBAC-матриця | 🟡 | D23, D24, D25 — треба розширити під permissions §4.5 |
| 15 | Scheduling algorithm | 🟡 | D28 — cron; пропозиція D47: черга на машину, PROD першим, FIFO всередині |
| 16 | Concurrency model | 🟡 | пропозиція D49: машина — 1 запуск, PROD-робот — 1 активний на організацію |
| 17 | Starvation prevention | 🟡 | пропозиція D47: у MVP окремо не вирішуємо — ризик низький, є тайм-аут черги |
| 18 | Resource locking | ⬜ | частково закриває D49 |
| 19 | Resource reservation | ⬜ | — |
| 20 | Deployment нової PROD Version без downtime | 🟡 | перемикання вказівника + запуск посилається на конкретну версію: запущені дотягують стару, нові беруть нову |
| 21 | UX lifecycle/version statuses у Studio | 🟡 | 12-screens — перелік екранів, без станів ТЗ |
| 22 | Working Version при одночасній Active Production Version | ✅ | D34: робоча копія не є версією й на Active Production Version не впливає |
| 23 | Модель Version Release State | ✅ | D39 — [18-version-release.md](18-version-release.md) |
| 24 | Чи потрібен DEPRECATED / RETIRED lifecycle | ⬜ | — |
| 25 | TEST Version — окремий authorization object чи тільки Test Period | ✅ | D37: тільки Test Period; authorization версії — лише для PROD |
| 26 | Чи охоплює Test Period нові Versions після approval | ✅ | D37: так — період видається на робота й діє для всіх його версій, зокрема нових |
| 27 | Що відбувається при створенні нової Version під час активного PROD | 💡 | §4.1.10: створення v28 не замінює v27 автоматично |
| 28 | Як Orchestrator визначає Definition для Assistant | ✅ | D4: сценарій у JSONB версії; приходить агенту в `JOB_ASSIGN` (09-protocol) |

**Підсумок:** ✅ 5 · 💡 2 · 🟡 8 · ⬜ 13.

## 6. Порядок перебудови документації

| Крок | Що | Стан |
|------|----|------|
| 1 | ТЗ у репозиторії: PDF + дослівна md-версія | ✅ |
| 2 | Трасування — цей файл | ✅ |
| 3 | Словник у `01-overview` під терміни ТЗ | ⬜ |
| 4 | Модель даних `03-data-model` під §7: `robots`, `robot_versions`, `executions`, `test_periods`, `audit_events`, розширений `approvals` | ✅ |
| 5 | Нові розділи: Lifecycle і TEST/PROD · Approvals і Test Period · Audit · Pre-run checks і черги · BPMN · Custom components | 🟡 Approvals і Test Period ✅ — [17](17-authorization.md); Release State ✅ — [18](18-version-release.md); Audit ✅ — [19](19-audit.md); Політики ✅ — [20](20-policies.md); Черги 🟡 — [21](21-queues.md); решта далі |
| 6 | Перегляд D11, D25, D28, D29, статусів запуску | ⬜ |
| 7 | API й протокол (`13-api`, `09-protocol`) під нову модель | ⬜ |
