# 42. Фаза 10 — `web-cdp`, браузер: задачі

> Що: третій провайдер (D17) — сторінки в браузері (CRM, портали, кабінети).
> Assistant керує **власним** браузером (D19), не Chrome людини. Фаза 9 (`office-com`)
> відкладена — [10-roadmap.md](10-roadmap.md). Проєктування — [04-approach.md](04-approach.md),
> [06-activities.md](06-activities.md), [05-selectors.md](05-selectors.md).

## Етапи

| Етап | Що | Чому так |
|------|----|----------|
| **10A** | виконання: формат веб-елементів у контракті, активності `web-cdp` у каталозі, провайдер в Assistant (Playwright + системний Edge), перевірки | без виконавця Inspector нема що перевіряти |
| **10B** | Inspector для веба (Assistant відкриває кероване вікно, людина клікає елемент → стратегії), Studio-front: захоплення веб-елемента, форма активностей | другий Inspector — окремий шматок |

## Етап 10A — рішення

| # | Питання | Рішення | Чому |
|---|---------|---------|------|
| 1 | Бібліотека й браузер | **Playwright 1.64.1** (sync API у потоці виконання) з **системним Microsoft Edge** (`channel="msedge"`), якщо його немає — Google Chrome (`channel="chrome"`); власних браузерів Playwright не завантажуємо. Assistant оголошує можливість `web-cdp`, лише коли знайшов Edge або Chrome | D19 (A); Edge є на кожній Windows 10/11 і оновлюється сам — без ~150 МБ свого Chromium у збірці |
| 2 | Життя браузера | один браузер на запуск, **чистий профіль** (без сесій людини, D19); видимий (headed); закривається наприкінці запуску завжди — навіть без `CloseBrowser`, і при помилці / Stop | робот логіниться сам (облікові дані D20); забутий браузер не лишається вночі |
| 3 | Формат веб-елемента | ті самі `elements` (D21), нові види стратегій: `{kind: "css", selector}`, `{kind: "role", role, name?}`, `{kind: "text", text}`; пробуються по черзі, як в UIA (D7); не перша спрацювала — `fallbacks_count` (D7a, D52). Один елемент — або UIA, або веб, не змішано. Iframe — не в MVP | 04-approach: «у браузера свій формат»; role/text — стійкіші за CSS, CSS — точніший |
| 4 | Активності (`web-cdp/…`, min Assistant 1.6.0) | `OpenBrowser {url?}`, `Navigate {url}`, `Click {target}`, `TypeText {target, text}` (облікові дані можна, D20), `GetText {target}` → output, `SelectOption {target, value}`, `WaitForElement {target, appears, timeout}`, `ExtractTable {target}` → список рядків (ключі — заголовки таблиці), `CloseBrowser` | 06-activities (назви — як у `desktop-uia`) |
| 5 | Перевірки (Studio-back) | `target` веб-активності — елемент із веб-стратегіями, UIA-активності — з UIA (`element_kind` помилка); веб-активність до `OpenBrowser` — попередження; URL — `http(s)://` або шаблон | 14-validation |
| 6 | Тайм-аути | очікування елемента — як у UIA (`timeout`, типово 10 с); навігація — 30 с | однакова поведінка провайдерів |
| 7 | Безпека | паролі (D20) маскуються в логах як завжди; Playwright без розширень, без `--remote-debugging` назовні (браузер запускає й тримає сам Assistant) | D19 B відкинуто саме через відкритий порт |

## Задачі 10A

| ID | Що | Де |
|----|----|----|
| **W1** ✅ | контракт: види стратегій `css` / `role` / `text`; каталог `web-cdp` (Studio-back і Desktop однаково); перевірки Studio-back | Orchester, Studio-back, Desktop |
| **W2** ✅ | Assistant: провайдер `web-cdp` на Playwright + Edge, активності, можливість `web-cdp`, закриття браузера, 1.6.0 | Desktop |
| **W3** ✅ | живий прогін: робот із веб-кроками (логін на тестовій сторінці, читання таблиці) → TEST-запуск в Assistant | усі |

Як зроблено (10A):

- **Контракт** — `CssStrategy`, `RoleStrategy`, `TextStrategy` поруч з UIA (`SelectorStrategy`),
  дискримінатор `kind`; `Element.kind`; змішаний елемент — помилка.
- **Каталог** — 9 активностей `web-cdp/*` (min 1.6.0), `catalog_version` 3, Studio-back і
  Desktop однакові. **Studio-back** — помилка `element_kind` (вид елемента не той, що в
  провайдера), попередження `browser_not_open`.
- **Assistant 1.6.0** — `PlaywrightWeb` (Playwright 1.64.1, системний Edge / Chrome, чистий
  профіль, закриття браузера наприкінці запуску завжди), можливість `web-cdp` — лише коли
  браузер знайдено; `fallbacks_count` для веб-елементів.

## Живий прогін 10A (2026-11-08)

Тестова сторінка на `http://127.0.0.1:8765/` (форма входу, список, таблиця), робот RBT-29 з
Studio-back → версія 29.1 → TEST-запуск в Assistant 1.6.0 (видимий Edge).

| Що | Результат |
|----|-----------|
| Перевірка сценарію | без помилок, `min_assistant_version` 1.6.0 |
| Запуск | `COMPLETED` 11/11: введення в поля (role «User» / css), вибір «South», клік «Sign in» (role), `GetText` → «Welcome, Petro», `ExtractTable` → 2 рядки, ForEach → «Romashka LLC = 1250.00», «Vasylko Ltd = 310.50» |
| Резервна стратегія | перша (`#no-such-thing`) не знайшла — друга знайшла: WARN «found by fallback strategy 2», `fallbacks_count` = 1 |
| Після запуску | процесів Edge, запущених Playwright, — 0 |
| Studio-front | сторінка RBT-29 відкривається, веб-елементи в бібліотеці, лог запуску видно (опис веб-стратегій — 10B) |

Не перевірено: збірка exe з драйвером Playwright (PyInstaller), Chrome без Edge.

