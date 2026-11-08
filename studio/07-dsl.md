# 7. DSL — як виглядає сценарій

> ⚠️ Чернетка. Тут зводяться докупи D1 (JSON-DSL), D6 (активності) і D7
> (селектори). Формат зміниться після першого реального робота.

## Що таке крок

Сценарій — це список кроків. Кожен крок виглядає однаково:

```json
{
  "id": "s3",
  "activity": "desktop-uia/Click",
  "label": "Натиснути «Провести»",
  "inputs": { "target": "el_post_button" },
  "output": null
}
```

| Поле | Навіщо |
|------|--------|
| `id` | стабільний ідентифікатор кроку. За ним логи прив'язуються до кроку і Studio підсвічує потрібний рядок |
| `activity` | яка це активність |
| `label` | те, що бачить юзер. Він може перейменувати, на виконання не впливає |
| `inputs` | параметри кроку |
| `output` | у яку змінну покласти результат (якщо активність щось повертає) |

### Ім'я активності містить провайдера

```
flow/If
flow/ForEach
desktop-uia/Click
web-cdp/Click
office-com/ReadRange
```

`Click` існує і в десктопі, і в браузері — це різні активності з різними
параметрами. Префікс провайдера прибирає плутанину і дає Executor'у зрозуміти,
кому передати крок.

## Вкладеність

`ForEach`, `If` і `TryCatch` містять інші кроки всередині себе:

```json
{
  "id": "s2",
  "activity": "flow/ForEach",
  "inputs": { "items": "{{rows}}", "itemVar": "row" },
  "body": [ { "...крок..." }, { "...крок..." } ]
}
```

| Активність | Поля з вкладеними кроками |
|-----------|---------------------------|
| `flow/ForEach` | `body` |
| `flow/If` | `then`, `else` |
| `flow/TryCatch` | `try`, `catch` |

## Умови без мови виразів

За D6b виразів немає. Тому умова — це **структура**, а не рядок:

```json
{
  "activity": "flow/If",
  "inputs": {
    "left": "{{row.amount}}",
    "operator": "greater_than",
    "right": "1000"
  },
  "then": [ ... ],
  "else": [ ... ]
}
```

Studio показує це трьома полями, юзер нічого не пише руками.

## Значення полів

За D6b значення — константа, посилання на змінну або шаблон:

```
"A2:D100"                     константа
"{{rows}}"                    змінна
"звіт_{{today}}.xlsx"         шаблон
```

## Повний приклад: «Excel → 1С»

Той самий канонічний сценарій з [06-activities.md](06-activities.md), тепер у JSON:

```json
{
  "schemaVersion": 1,
  "parameters": [
    { "name": "file", "type": "string", "label": "Файл замовлень" }
  ],
  "elements": {
    "el_create":  { "provider": "desktop-uia", "strategies": [ "…" ] },
    "el_partner": { "provider": "desktop-uia", "strategies": [ "…" ] },
    "el_amount":  { "provider": "desktop-uia", "strategies": [ "…" ] },
    "el_post":    { "provider": "desktop-uia", "strategies": [ "…" ] }
  },
  "steps": [
    { "id": "s1", "activity": "office-com/OpenWorkbook",
      "inputs": { "path": "{{file}}" }, "output": "wb" },

    { "id": "s2", "activity": "office-com/ReadRange",
      "inputs": { "workbook": "{{wb}}", "sheet": "Аркуш1", "range": "A2:D100" },
      "output": "rows" },

    { "id": "s3", "activity": "flow/ForEach",
      "inputs": { "items": "{{rows}}", "itemVar": "row" },
      "body": [

        { "id": "s4", "activity": "flow/TryCatch",
          "try": [
            { "id": "s5", "activity": "desktop-uia/AttachWindow",
              "inputs": { "title": "1С:Підприємство*" }, "output": "win" },

            { "id": "s6", "activity": "desktop-uia/Click",
              "inputs": { "target": "el_create" } },

            { "id": "s7", "activity": "desktop-uia/TypeText",
              "inputs": { "target": "el_partner", "text": "{{row.contractor}}" } },

            { "id": "s8", "activity": "desktop-uia/TypeText",
              "inputs": { "target": "el_amount", "text": "{{row.amount}}" } },

            { "id": "s9", "activity": "desktop-uia/Click",
              "inputs": { "target": "el_post" } }
          ],
          "catch": [
            { "id": "s10", "activity": "flow/Log",
              "inputs": { "level": "ERROR",
                          "message": "Не вдалось провести {{row.contractor}}" } }
          ] }
      ] },

    { "id": "s11", "activity": "office-com/CloseWorkbook",
      "inputs": { "workbook": "{{wb}}" } }
  ]
}
```

Зверни увагу: кроки посилаються на елементи **по ідентифікатору**
(`"target": "el_post"`), а самі описи елементів лежать окремо в `elements`.
Чому саме так — питання D21.

## Формат фази 1 — на ньому живе тонка нитка (J3)

> 2026-11-06. Код — `RobotDefinition` у `src/contracts/definition.py` (однаковий в
> оркестраторі й Studio-back). Чернетка вище лишається напрямком; поки працює лише
> те, що нижче.

```json
{
  "steps": [
    {
      "id": "s1",
      "activity": "desktop-uia/Click",
      "label": "Натиснути OK",
      "inputs": { "target": "ok_button" }
    }
  ],
  "elements": {
    "ok_button": {
      "strategies": [
        {
          "kind": "uia",
          "path": [
            { "control": "Window", "name": "Tekra test*" },
            { "control": "Button", "automation_id": "okButton" }
          ]
        }
      ]
    }
  }
}
```

- `steps` — щонайменше один; `id` унікальні; жодних полів, яких немає у форматі.
- `elements` — бібліотека елементів робота (D21); `inputs.target` посилається на
  ключ бібліотеки, і такий ключ мусить існувати.
- Селектор — список способів (D7), у фазі 1 лише `kind: uia`: шлях від вікна до
  елемента; перший вузол — вікно, останній — сам елемент, проміжні можна
  пропускати (D7b). `name` з `*` наприкінці — «починається з»; `automation_id` — AutomationId (не `id`: не затіняє вбудоване ім'я Python і не плутається з `id` кроку).
- Які активності існують — справа каталогу Studio-back (D2). У фазі 1 —
  лише `desktop-uia/Click`.
- **Хеш** (`definition_hash`) — SHA-256 від JSON із відсортованими ключами, без
  пробілів, UTF-8; одна функція в `contracts` для обох сервісів (28, «Цілісність
  версії»).

## Формат фази 2 — логіка й повний `desktop-uia` (P1)

> 2026-11-07, рішення — [34-phase-2.md](34-phase-2.md), «Відкриті питання». Формат
> фази 1 лишається валідним: це той самий формат без вкладених блоків.

### Вкладені блоки

Крок може мати списки кроків — **лише** у своєї активності:

| Активність | Блоки | Обовʼязкові |
|-----------|-------|-------------|
| `flow/If` | `then`, `else` | `then` |
| `flow/ForEach` | `body` | `body` |
| `flow/TryCatch` | `try`, `catch` | `try` |

- `id` кроків унікальні **в усьому дереві**; `inputs.target` перевіряється на будь-якій
  глибині. Блок у чужої активності (`then` у `Click`) — сценарій невалідний.
- Глибина вкладеності — не більше `DEFINITION_MAX_DEPTH` (16) рівнів.
- Порожній блок дозволено (`"else": []`).

### Змінні й шаблони (D6b)

- Ім'я змінної — `[A-Za-z_][A-Za-z0-9_]*`. Змінна з'являється з `output` кроку,
  `itemVar` циклу (лише в його тілі) і `errorVar` у `catch`.
- Шаблони підставляються в **усі рядкові входи** (і в рядки всередині списків /
  обʼєктів), крім назв змінних (`itemVar`, `errorVar`). `target` — **статичний**
  ключ бібліотеки елементів, не шаблон (D21: видно, які кроки зачепить зміна
  елемента; сценарій із невідомим ключем невалідний ще до запуску).
- Значення **цілком** `"{{rows}}"` — сама змінна **з її типом** (список, обʼєкт,
  число). Шаблон **у тексті** `"звіт_{{n}}.txt"` — рядок: рядкові значення як є,
  решта — як JSON (`1000`, `true`, `null`, `[1,2]`).
- `{{row.amount}}` — поле обʼєкта, `{{rows.0}}` — елемент списку за індексом.
- **Невідома змінна, поле чи індекс — помилка кроку** з назвою. Екранування `{{`
  у фазі 2 немає.

### Числа в умовах

Рядок вважається числом, якщо після видалення пробілів (звичайних і
нерозривних — так 1С показує `1 000,50`) він є числом; кома — десяткова, якщо
крапки немає. `true` / `false` числами не є.

### Активності фази 2

Спільне для активностей `desktop-uia` з `target`: необовʼязковий `timeout` —
секунди очікування елемента (типово 10, не більше 600).

| Активність | `inputs` | `output` |
|-----------|----------|----------|
| `flow/SetVariable` | `value` — будь-яке JSON-значення або шаблон | **обовʼязковий** — ім'я змінної |
| `flow/If` | `left`, `operator`, `right` (немає для `is_empty` / `is_not_empty`) | — |
| `flow/ForEach` | `items` — список (зазвичай `"{{rows}}"`), `itemVar` — ім'я змінної елемента | — |
| `flow/TryCatch` | `errorVar` — куди покласти текст помилки (типово `error`) | — |
| `flow/Delay` | `seconds` — число, не більше 3600 | — |
| `flow/Log` | `message`; `level` — `INFO` (типово) / `WARN` / `ERROR` | — |
| `desktop-uia/AttachWindow` | `target` — елемент, чий шлях — саме вікно; дочекатися, розгорнути, вивести на передній план | — |
| `desktop-uia/Click` | `target` | — |
| `desktop-uia/TypeText` | `target`, `text`; `clear` — спершу очистити (типово `true`) | — |
| `desktop-uia/GetText` | `target` | текст елемента |
| `desktop-uia/SelectItem` | `target` (список / випадайка), `item` — текст пункту | — |
| `desktop-uia/WaitForElement` | `target`; `state` — `appears` (типово) / `disappears` | — |
| `desktop-uia/PressKeys` | `keys` — список комбінацій, напр. `["Ctrl+S", "Enter"]`; `target` — необовʼязково: спершу дати йому фокус | — |

**Оператори `flow/If`:** `equals` / `not_equals` — числа як числа, інакше як текст;
`greater_than` / `less_than` — лише числа (інакше помилка кроку); `contains` —
підрядок, а для списку — чи є такий елемент; `is_empty` / `is_not_empty` —
відсутнє значення (`null`), порожній або з одних пробілів рядок, порожній
список / обʼєкт.

**Клавіші `PressKeys`:** комбінація — назви через `+`, без пробілів, регістр не
важливий: `Ctrl`, `Alt`, `Shift`, `Win` + одна клавіша: `A`–`Z`, `0`–`9`,
`F1`–`F12`, `Enter`, `Tab`, `Esc`, `Space`, `Backspace`, `Delete`, `Insert`,
`Home`, `End`, `PageUp`, `PageDown`, `Up`, `Down`, `Left`, `Right`.

**`desktop-uia/Screenshot`** — у фазі 3, разом зі сховищем знімків.

### Помилки, `TryCatch` і лічильники

- Помилка кроку в `try` → рядок `WARN` «Step s7 failed, handled by s4: …», текст — у
  змінній `errorVar`, далі виконується `catch`, потім — крок після `TryCatch`.
  Запуск, де всі помилки зловлено, — `COMPLETED`.
- Помилка поза `try` (або в `catch`) — `ERROR` і `FAILED`, як у фазі 1. Стоп
  (Assistant закривається) `TryCatch` не ловить — `CANCELLED`.
- `flow/Log` з `level: ERROR` — рядок `ERROR` від автора сценарію; запуск не
  зупиняє, але рахується в `errors_count`.
- `steps_total` — кроків у дереві сценарію; `steps_done` — **різних** кроків, які
  хоч раз завершились успішно.

### Приклад — канонічний сценарій на Блокноті

```json
{
  "steps": [
    { "id": "s1", "activity": "flow/SetVariable",
      "inputs": { "value": [ { "who": "Іван", "amount": "1 200,50" },
                             { "who": "Марія", "amount": "300" } ] },
      "output": "rows" },
    { "id": "s2", "activity": "desktop-uia/AttachWindow", "inputs": { "target": "notepad" } },
    { "id": "s3", "activity": "flow/ForEach",
      "inputs": { "items": "{{rows}}", "itemVar": "row" },
      "body": [
        { "id": "s4", "activity": "flow/TryCatch",
          "try": [
            { "id": "s5", "activity": "desktop-uia/TypeText",
              "inputs": { "target": "editor", "text": "{{row.who}}: {{row.amount}}\n", "clear": false } },
            { "id": "s6", "activity": "flow/If",
              "inputs": { "left": "{{row.amount}}", "operator": "greater_than", "right": "1000" },
              "then": [
                { "id": "s7", "activity": "flow/Log",
                  "inputs": { "message": "Велика сума: {{row.who}}" } } ] }
          ],
          "catch": [
            { "id": "s8", "activity": "flow/Log",
              "inputs": { "level": "WARN", "message": "Не вийшло для {{row.who}}: {{error}}" } } ] }
      ] },
    { "id": "s9", "activity": "desktop-uia/GetText", "inputs": { "target": "editor" }, "output": "typed" },
    { "id": "s10", "activity": "flow/Log", "inputs": { "message": "У Блокноті: {{typed}}" } }
  ],
  "elements": {
    "notepad": { "strategies": [ { "kind": "uia", "path": [
      { "control": "Window", "name": "*Notepad" } ] } ] },
    "editor": { "strategies": [ { "kind": "uia", "path": [
      { "control": "Window", "name": "*Notepad" },
      { "control": "Document" } ] } ] }
  }
}
```

**Елемент бібліотеки** (з фази 3): `strategies` — обовʼязково; `label` — як
елемент назвала людина / Inspector; `snapshot` — знімок захоплення (D7a): процес,
властивості елемента й предків, межі, патерни, `image` (PNG data URL ≤ 64 KiB).
Знімок при виконанні не використовується.

**`class_name`** у вузлі шляху (з фази 2) — клас вікна чи елемента (`Notepad`,
`CabinetWClass`, `QPushButton`): на відміну від заголовка не залежить від мови
Windows і відкритого документа; можна разом з `name` і `automation_id`.

**`*` у `name`** (з фази 2): наприкінці — «починається з», на початку —
«закінчується на» (заголовок вікна зазвичай `<документ> - <програма>`), з обох
боків — «містить». Точні шляхи елементів Блокнота (Windows 10 — `Edit`,
Windows 11 — `Document`) уточнюються в P4 на справжньому вікні.

## Фаза 4 — каталог, облікові дані, перевірка (E1–E7)

**Каталог як дані.** Які активності є, їхні входи (`type`, `required`,
`required_when`, `default`, `options`), `output` і блоки — у
`Tekra-Studio-back/src/catalog/activities.json` (`GET /activities`); копія того ж
файлу — у десктопі, і контрактний тест там звіряє її з реалізацією. З каталогу
Studio будує форму кроку й перевіряє сценарій; `min_assistant_version` версії —
найбільша з активностей сценарію та фіч (`credentials` — `1.2.0`).

**Облікові дані робота (D20).** Вхід типу `text_or_secret` (зараз — `text` у
`desktop-uia/TypeText`) приймає або текст, або посилання:

```json
"text": { "credential": "eed261f6-…", "part": "password" }
```

`part` — `username` або `password`. У сценарії лише id; значення Assistant
отримує в `execution/assign.credentials` і в логах показує `***`.

**Перевірка (14-validation).** `ScenarioValidator` у Studio-back: структура й
типи входів за каталогом, обов'язкові поля (з урахуванням `required_when`),
блоки, унікальні id кроків, посилання на елементи бібліотеки, змінна оголошена до
використання (`output`, `itemVar`, `errorVar`), формат клавіш; оркестраторне —
облікові дані існують, провайдер і версії Assistant (`orchestrator.scenario_check`).
Помилки блокують «Зберегти», попередження (крок без назви, елемент з однією
стратегією, давно знятий елемент) — ні.

**Редактор (Studio-front).** Дерево кроків з блоками (`then` / `else` / `body` /
`try` / `catch`), «+ Add step» у кінці кожного блоку, ↑ / ↓ / Delete; форма з
каталогу; елементи — з бібліотеки; облікові дані — випадайкою (назва й логін, не
пароль); панель перевірки веде до кроку. Сирий JSON — вкладка «Advanced (JSON)»;
обидві вкладки редагують той самий текст робочої копії.

## Відкриті питання

- **D21** — селектори всередині кроків чи окремою бібліотекою елементів
- **D22** — чи оголошує робот свої вхідні параметри

Обидва — у [02-decisions.md](02-decisions.md).

## Фаза 5 — вхідні параметри (D22)

> 2026-11-08, [37-phase-5.md](37-phase-5.md). Код — `Parameter` у `src/contracts/definition.py`.

```json
{
  "parameters": [
    { "name": "file", "title": "Orders file", "type": "text", "required": true },
    { "name": "limit", "type": "number", "default": 10 }
  ],
  "steps": [ … "{{file}}" … ],
  "elements": { … }
}
```

- `name` — ім'я змінної (`[A-Za-z_][A-Za-z0-9_]*`), унікальне; `type` — `text` /
  `number` / `boolean` (за замовчуванням `text`); `default` — того самого типу.
- Значення дає запуск (`POST /executions { input }`), оркестратор перевіряє й
  підставляє `default`; на старті кожен параметр — змінна. Необов'язковий без
  значення й без `default` — `null`.
- Studio вважає параметри змінними з першого кроку; невикористаний — попередження
  (`field = "parameters.<name>"`). Версія з параметрами вимагає Assistant 1.3.0
  (фіча каталогу `parameters`).
- Паролів у параметрах немає — лише облікові дані (D20).
- У Studio — блок «Parameters» над деревом кроків і «Run with» перед кнопкою Run.

## Крок-компонент (фаза 8A, [41-phase-8](41-phase-8.md))

```json
{ "id": "s1", "activity": "flow/Component", "label": "Login",
  "inputs": { "who": "{{user}}" }, "output": "greeting",
  "component": { "id": "<uuid>", "number": 7, "version": 3,
                 "name": "Login to BAS", "definition": { "parameters": [], "steps": [], "elements": {}, "result": "greeting" } } }
```

У робочій копії — лише `id` / `number` / `version`; `definition` (вміст версії компонента)
Studio-back кладе при «Save» (D64). Усередині — власна область змінних: лише параметри
всередину, лише `result` назовні; елементи — компонента; кроки в логах — `s1/c1`.
