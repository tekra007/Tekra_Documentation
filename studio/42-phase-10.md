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


## Етап 10B — рішення

| # | Питання | Рішення | Чому |
|---|---------|---------|------|
| 1 | Як захоплюється веб-елемент | `POST /inspector-sessions { provider: "web-cdp", url }`: Assistant відкриває **кероване вікно** Edge (чистий профіль, як у запуску) на `url`; людина ходить сторінками (зокрема входить сама); під курсором — підсвітка елемента; **Ctrl + клік** — захоплення (звичайні кліки працюють як завжди, щоб можна було дійти до потрібної сторінки); вікно закривається. Esc / «Cancel» у Studio / тайм-аут сесії (5 хв) — скасування | D19: Inspector у власному вікні; Ctrl — щоб не плутати навігацію із захопленням |
| 2 | Які стратегії | у порядку стійкості: `role` + доступна назва (aria-label, `<label>`, текст кнопки / посилання), `css` (`#id`, `[data-testid]`, `[name]`, інакше короткий шлях із `:nth-of-type`), `text` (видимий текст до 80 символів, для кнопок / посилань); кожна перевіряється в Assistant на **однозначність** (рівно один збіг) — неоднозначні відкидаються; жодної — помилка «element not unique» | D7: кілька способів; стійкі першими |
| 3 | Знімок (D7a) | `snapshot: { url, title, tag, attributes (без value полів пароля), text (≤200), outer_html (≤2000, без value), rect, captured_at, provider: "web-cdp" }`; зображення — не в MVP | знімок для перезахоплення, не для пошуку; пароль не потрапляє |
| 4 | Адреса | `url` — `http(s)://` або порожньо (`about:blank`); перевіряє оркестратор | не відкривати `file://` / `javascript:` з Studio |
| 5 | Studio-front | «Capture element»: вибір «Window (desktop)» / «Web page» + поле адреси (пам'ятається остання); бібліотека елементів показує вид і стратегії людською мовою (`button «Sign in»`, `#login`, `text «Sign in»`); у формі кроку вибір елемента — лише відповідного виду (веб-крок — веб-елементи) | 12-screens |
| 6 | Таблиця / список | **Ctrl + Shift + клік** — захоплюється найближча таблиця / список, а не клітинка (для `ExtractTable`) | додано під час реалізації: Ctrl + клік по таблиці майже завжди влучає в клітинку |

## Задачі 10B

| ID | Що | Де |
|----|----|----|
| **I1** ✅ | оркестратор: `url` у сесії й у `inspector/start`, перевірка адреси | Orchester |
| **I2** ✅ | Assistant: веб-Inspector на Playwright (підсвітка, Ctrl + клік, стратегії з перевіркою однозначності, знімок), 1.7.0 | Desktop |
| **I3** ✅ | Studio-front: захоплення веб-елемента, вигляд веб-елементів, фільтр за видом у формі кроку | Studio-front |
| **I4** ✅ | живий прогін: захоплення на тестовій сторінці → крок → TEST-запуск | усі |

Як зроблено (10B):

- **Оркестратор** — `url` у `InspectorSessionRequestSchema` (лише `http(s)://`,
  порожньо → `None`) і в `inspector/start`; перевірка можливості провайдера — як була.
- **Assistant 1.7.0** — `PlaywrightInspector` за `WebInspector`: окремий Edge із
  чистим профілем, підсвітка, Ctrl + клік / Ctrl + Shift + клік, стратегії перевіряються
  тим самим пошуком, що й запуск; знімок без значень полів; iframe — помилка.
- **Studio-front** — вибір «Window / Web page» з адресою, вид і стратегії людською
  мовою в бібліотеці, фільтр елементів за видом у формі кроку.

## Живий прогін 10B (2026-11-09)

Робот RBT-31: `OpenBrowser` → `TypeText` → `Click «Sign in»` (елемента `sign_in` ще
немає — перевірка: «unknown element 'sign_in'») → `GetText` → `Log` → `CloseBrowser`;
у бібліотеці ще UIA-елемент `notepad_ok`. Studio-front у headless-браузері, «людина»
у вікні Assistant — Ctrl + клік через Windows UI Automation.

| Що | Результат |
|----|-----------|
| Адреса `javascript:alert(1)` | «Use an address starting with http:// or https://…», Start неактивний |
| Захоплення | Assistant 1.7.0 відкрив Edge на `http://127.0.0.1:8765/`, Ctrl + клік по «Sign in» → у Studio «Captured»: `button «Sign in»`, `#login > button`, адреса сторінки; текстова стратегія відкинута як неоднозначна |
| Add to scenario | `sign_in` у бібліотеці («Page · Sign in · 2 ways», крок s3), помилка перевірки зникла |
| Форма кроку `Click (page)` | вибір елемента: `user`, `welcome`, `sign_in` — без `notepad_ok` (UIA) |
| Run (TEST) | `COMPLETED` 6/6, «Welcome, Petro», `fallbacks_count` 0 |
| Знімок | `url, title, tag, attributes, text, outer_html, rect, captured_at, provider`; `outer_html` — `<button type="submit">Sign in</button>` |
| Після | процесів Edge, запущених Playwright, — 0 |

Не перевірено: Chrome без Edge, збірка exe, захоплення таблиці (Ctrl + Shift + клік) наживо —
лише в тестах `tests/web`.
