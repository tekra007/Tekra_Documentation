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

## Відкриті питання

- **D21** — селектори всередині кроків чи окремою бібліотекою елементів
- **D22** — чи оголошує робот свої вхідні параметри

Обидва — у [02-decisions.md](02-decisions.md).
