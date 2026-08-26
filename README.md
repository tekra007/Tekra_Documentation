# Tekra — документація workspace

Крос-репозиторійна документація для трьох проєктів Tekra. Живе окремим
репозиторієм, бо не належить жодному з них.

У воркспейсі лежить як `d:/tekra/Tekra_Documentation/`, поруч із кодовими
репозиторіями.

## Для агентів і розробників

1. Прочитайте **[`AGENTS.md`](AGENTS.md)** — загальні правила для всіх проєктів.
   Канонічна версія лежить **тут**; у корені воркспейсу — лише вказівник.
2. Відкрийте **`<Проєкт>/docs/FOR-AGENTS.md`** обраного репозиторію — правила
   саме для FastAPI, Angular або Desktop.
3. Для крос-репозиторійних тем — розділи «Проєктування» нижче.

## Проєкти

Словник назв у чатах: **десктоп** → `Tekra-Desktop/`; **оркестратор** →
`Tekra-Orchester/`; **фронтенд** / **оркестратор фейс** / **фейс** / **Angular**
→ `Tekra-Orchester-Face/`. Повна таблиця — в [`AGENTS.md`](AGENTS.md).

| Репозиторій | Документація для агентів |
|-------------|--------------------------|
| `Tekra-Orchester` | [docs/FOR-AGENTS.md](https://github.com/tekra007/Tekra-Orchester/blob/main/docs/FOR-AGENTS.md) |
| `Tekra-Orchester-Face` | [docs/FOR-AGENTS.md](https://github.com/tekra007/Tekra-Orchester-Face/blob/main/docs/FOR-AGENTS.md) · еталон структури — [example-project-for-agents](https://github.com/tekra007/Tekra-Orchester-Face/tree/main/docs/example-project-for-agents) |
| `Tekra-Desktop` | [docs/FOR-AGENTS.md](https://github.com/tekra007/Tekra-Desktop/blob/main/docs/FOR-AGENTS.md) |

## Проєктування

| Розділ | Про що |
|--------|--------|
| [`studio/`](studio/README.md) | **Tekra Studio** — веб-редактор роботів, Inspector, виконання. Рішення D1–D30 |

## Правила для цього репозиторію

- посилання **всередині** репозиторію — відносні;
- посилання на код і документи **інших репозиторіїв** — повні GitHub-URL,
  інакше після клонування вони ведуть у нікуди;
- нотатки тримаємо **короткими**; великі обговорення фіксуємо резюме на 3–5
  рядків або посиланням на issue / PR.
