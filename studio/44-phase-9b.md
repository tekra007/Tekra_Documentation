# 44. Фаза 9B — Word, Outlook, пошта, PDF: задачі

> Що: наступні за Excel задачі бухгалтерії — договори й акти з шаблону (Word), листи з
> вкладеннями (Outlook або напряму поштою), дані з PDF. Додано 2026-11-10 після фази 9
> ([43-phase-9.md](43-phase-9.md)); після 9B — пауза на обговорення перед фазою 11 (1С / BAS).

## Рішення

| # | Питання | Рішення | Чому |
|---|---------|---------|------|
| 1 | Провайдери | нові: **`word-com`** (Word через COM), **`outlook-com`** (Outlook через COM), **`email`** (пошта напряму: SMTP / IMAP, без Outlook), **`pdf`** (файли PDF, без застосунку). `office-com` лишається Excel | провайдер = можливість Assistant (D17, перевірка №12): Word, Outlook і Excel бувають на машині не разом; пошта й PDF не потребують Office |
| 2 | Word — як Excel (43, №2–3, №9) | власний прихований Word на запуск (`DispatchEx`), закривається наприкінці запуску завжди (без збереження), завис / діалог — процес завершується за PID; виклики COM — у потоці виконання, лише позиційні аргументи; шляхи повні, `normpath`; дескриптор документа — повний шлях або `new:N`; макроси не запускаються | ті самі граблі, що знайшли в Excel |
| 3 | Активності `word-com/…` (min Assistant **1.9.0**) | `OpenDocument {path, password?, readOnly?}` → дескриптор; `CreateDocument {template?}` → дескриптор (з `.dotx` / `.docx`-шаблону або порожній); `FillTemplate {values}` → кількість замін — `{{ключ}}` у документі (текст, колонтитули, написи, таблиці) замінюється значенням `values.ключ`, невідомі ключі в документі лишаються й повертаються попередженням у лог; `ReplaceText {find, replace, matchCase?}` → кількість; `AppendText {text}` (новий абзац у кінці); `GetText` → текст документа; `ReadTables` → список таблиць (рядки — об'єкти за першим рядком, як `ReadRange`); `SaveDocument {path?}` (формат за розширенням: `.docx`, `.docm`, `.doc`, `.dotx`, `.rtf`, `.txt`, `.odt`, `.html`, `.pdf`); `CloseDocument {save?}` (+ необов'язковий `document`) | «договір / акт із шаблону → PDF» — головний сценарій; `{{ключ}}` — той самий вигляд, що й шаблони Tekra |
| 4 | Outlook | **Outlook людини** (профіль, у який вона увійшла) — інакше пошти немає; Assistant не закриває Outlook, якщо той уже працював, і закриває, якщо запускав сам. Без профілю — помилка «Outlook has no mail profile» | COM Outlook — один екземпляр на користувача, своєї скриньки в робота немає |
| 5 | Активності `outlook-com/…` (min 1.9.0) | `SendMail {to, cc?, bcc?, subject, body, html?, attachments?, account?}` (адреси через `;` або список; вкладення — список повних шляхів; `account` — з якої скриньки, типово основна); `ReadMail {folder?, unreadOnly?, from?, subjectContains?, since?, limit?, markAsRead?, saveAttachmentsTo?}` → список листів `{id, from, to, subject, received, body, attachments}` (`attachments` — імена або, з `saveAttachmentsTo`, повні шляхи збережених файлів) | надіслати акт / забрати рахунки з пошти; окремі «позначити» / «зберегти вкладення» — опції `ReadMail`, щоб не тягати id листа між кроками |
| 6 | Пошта напряму (`email/…`, min 1.9.0) | `SendMail {server, security?, login, password, from, to, cc?, bcc?, subject, body, html?, attachments?}` (SMTP); `ReadMail {server, security?, login, password, folder?, unreadOnly?, from?, subjectContains?, since?, limit?, markAsRead?, saveAttachmentsTo?}` (IMAP) → той самий формат листів, що в Outlook. `server` — `host:port`; `security` — `starttls` (типово) / `ssl` / `none`; `login` / `password` — облікові дані робота (D20, маскуються) | працює на машині без Outlook і на сервері; формат листа однаковий — сценарій легко перенести |
| 7 | PDF (`pdf/…`, min 1.9.0) | `ReadText {path, pages?, password?}` → текст (сторінки через порожній рядок); `ReadTables {path, pages?, password?, headers?}` → список таблиць (як `ReadTables` Word); `GetInfo {path, password?}` → `{pages, title, author, created, has_text}`; `MergeFiles {paths, path}`; `ExtractPages {path, pages, saveTo}`. `pages` — `1,3,5-7`; сторінок без тексту (скан) — `has_text: false` і попередження «looks like a scan — OCR is not supported yet». Бібліотеки: **pdfplumber** (текст і таблиці), **pypdf** (злиття, сторінки, пароль) | цифрові PDF (рахунки, виписки) — більшість; OCR сканів — окремо, не в 9B |
| 8 | Можливості | `word-com` / `outlook-com` — якщо в реєстрі `Word.Application` / `Outlook.Application` (без запуску); `email`, `pdf` — завжди | як `office-com` |
| 9 | Перевірки Studio-back | константні `path` / `template` / `output` / `saveAttachmentsTo` — повний шлях (`office_path`, як в Excel); `server` — `host:port` (`email_server`); адреси `to` / `cc` / `bcc` / `from` — схожі на e-mail (`email_address`, лише константи); `pages` — `1,3,5-7` (`pdf_pages`); `word-com`-крок до `OpenDocument` / `CreateDocument` — попередження `document_not_open` | 14-validation |
| 10 | Живий прогін | Word, PDF — на цій машині; `email` — на тестовому поштовому сервері з SMTP + IMAP (GreenMail у Docker, лише для перевірки); `outlook-com` — тести з підставним COM (профілю Outlook на машині розробки немає), наживо — коли з'явиться скринька | 2026-11-10: Outlook без профілю |

## Задачі

| ID | Що | Де |
|----|----|----|
| **Y1** ✅ | каталог `word-com` / `outlook-com` / `email` / `pdf` (Studio-back і Desktop однаково, `catalog_version` 5); перевірки Studio-back | Studio-back, Desktop |
| **Y2** ✅ | Assistant: чотири провайдери, можливості, закриття Word / Outlook, 1.9.0 | Desktop |
| **Y3** ✅ | живий прогін: шаблон договору в Word → PDF → лист із вкладенням (SMTP) → прочитати лист і зберегти вкладення (IMAP) → дані з PDF | усі |

Як зроблено:

- **Каталог** `catalog_version` 5: `word-com/*` (9), `outlook-com/*` (2), `email/*` (2),
  `pdf/*` (5), min 1.9.0; Studio-back і Desktop однакові. **Studio-back** — `office_path`
  для шляхів і списків шляхів, `email_address`, `email_server`, `pdf_pages`, попередження
  `document_not_open`.
- **Assistant 1.9.0** — `WordComProvider` (як Excel: свій прихований Word, сторож діалогів
  `OfficeWatch` — тепер спільний для Excel і Word, завершення за PID), `OutlookComProvider`
  (Outlook людини; закривається, лише якщо запускав робот), `SmtpImapMailProvider`
  (stdlib), `PdfPlumberProvider` (pdfplumber 0.11.9 + pypdf 6.21.4); `word-com` /
  `outlook-com` — за реєстром, `email` / `pdf` — завжди.
- Виявлено на живому Word: порожній пароль Word вважає «немає» і показує діалог (навіть
  прихований стає видимим) → заповнювач замість порожнього; зайнятий файл — теж діалог →
  власна перевірка до відкриття; `Hwnd` у Word немає → PID за унікальним `Caption`.
- Під час прогону вхід `ExtractPages.output` перейменовано на `saveTo`: `output` плутався
  зі «змінною результату» кроку.

## Живий прогін (2026-11-10)

Робот RBT-37, 28 кроків → TEST-запуск в Assistant 1.9.0. Шаблон акта — RTF із `{{contractor}}`
у колонтитулі, тексті й таблиці та `{{signature}}` без значення; пошта — тестовий сервер
GreenMail 2.1.3 у Docker (скриньки `robot`, `accounting`, лише 127.0.0.1).

| Що | Результат |
|----|-----------|
| Перевірка сценарію | без помилок і попереджень, `min_assistant_version` 1.9.0 |
| Запуск | `COMPLETED` 28/28 за 20 с (Excel і Word в одному запуску) |
| Excel → Word | 3 рядки `invoices.xlsx` → 3 акти з шаблону, по 4 заміни (з колонтитулом); WARN «No value for {{signature}} — left in the document as is» |
| Word → PDF | `act Romashka LLC.pdf`, `act Vasylko Ltd.pdf`, `act Kalyna JSC.pdf`; `ReadTables` шаблону — `[{"Item": "Services, November", "Amount": "{{amount}}"}]` |
| PDF | `MergeFiles` → `acts.pdf` (3 сторінки, `has_text: true`); `ExtractPages 2` → текст другого акта: «Contractor: Vasylko Ltd … 310.5» |
| SMTP | лист «Acts for November — акти за листопад» з `acts.pdf` |
| IMAP | лист знайдено (`subjectContains`), вкладення збережене в `inboxcts.pdf` (3 сторінки), `markAsRead` — повторне читання непрочитаних дає `[]` |
| Після | процесів Word / Excel — 0 |

Окремо: тести пошти на GreenMail 4/4, PDF 8/8, справжній Word 7/7.

Не перевірено: `outlook-com` наживо (на машині розробки немає профілю Outlook — лише тести з
підставним COM; діалог безпеки Outlook «програма надсилає лист» не обробляється),
`pdf/ReadTables` у живому запуску (лише тести), зібраний exe. Тести Excel на неактивованому
Office зрідка падають на збереженні в PDF («The remote procedure call failed») — середовище,
не код (так само на коді фази 9).

