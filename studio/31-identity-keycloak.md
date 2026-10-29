# 31. Ідентичність і Keycloak — дизайн (E0)

> Задача **E0** з [30-phase-0.md](30-phase-0.md). Під D75 (глобальний акаунт +
> членство), D81 (Keycloak). Складено 2026-10-23. ✅ **Затверджено того ж дня —
> D83**, усі сім пунктів розділу «Що затвердити» — як запропоновано.

## Що вирішуємо

1. Де живуть **ролі й членство** в організації.
2. Як **входить Assistant** (десктоп).
3. Що стає з **наявним входом** і з таблицями `admins`, `users` та всім, що
   на них посилається.

## Що вміє Keycloak — перевірено

| Можливість | Стан | Джерело |
|------------|------|---------|
| **Організації** (Organizations): тенанти в одному realm; людина — в кількох організаціях; членство в токені полем `organization`; своя корпоративна система входу на організацію за доменом пошти | ✅ офіційно з Keycloak 26 (у 25 — preview) | реліз 26.0.0 |
| **Ролі в межах організації** для звичайних користувачів («адмін в А, розробник у Б») | ❌ **немає**. Перенесено за межі 26; у 26.7 очікуються ролі для **керування** організаціями, а не ролі користувачів | issue #30180; блог Keycloak про 26.7 |
| запрошення в організацію (API) | ✅ з 26.5 | javadoc `OrganizationInvitationsResource` |
| вхід через Google (identity brokering) | ✅ вбудовано | Server Admin Guide |
| захист від перебору паролів | ✅ вбудовано (тимчасове й постійне блокування) | Server Admin Guide |
| політики паролів | ✅ вбудовано; точні назви (строк дії, історія) — звірити при налаштуванні | Server Admin Guide |
| обмеження кількості сесій користувача | ✅ автентифікатор «User Session Count Limiter»; **офлайн-токени його обходять** | сторонні інструкції — звірити |
| другий фактор **TOTP** (застосунок-генератор), WebAuthn | ✅ вбудовано | Server Admin Guide |
| другий фактор **кодом на пошту** | ❌ не вбудовано — лише сторонній плагін | форум Keycloak, проєкт `keycloak-2fa-email-authenticator` |
| вхід із десктопа через системний браузер: `http://127.0.0.1` з будь-яким портом | ✅ | документація OIDC Keycloak |
| Device Authorization Grant (вхід на пристрої без браузера) | ✅ | документація OIDC Keycloak |

## Пропозиція

### 1. Keycloak — лише «хто людина». Членство й ролі — у нас

| Де | Що |
|----|----|
| **Keycloak** | акаунт: вхід, пароль, другий фактор, Google, захист від перебору, сесії, скидання пароля |
| **наша БД** | `accounts` — дзеркало акаунта (`keycloak_sub`, пошта, імʼя); `memberships` — акаунт × організація × **роль** (`ADMIN` / `DEVELOPER` / `USER`) |

**Чому так.** Ролей у межах організації Keycloak не вміє, а нам вони потрібні
одразу (D56). Якщо тримати членство в Keycloak, а ролі в нас — два джерела
правди про одне й те саме. Тому **одне джерело — наша БД**.

**Organizations Keycloak у MVP не використовуємо.** Знадобляться пізніше — щоб
організація в спільному Free-середовищі входила через свою корпоративну систему.
У Paid-deployment корпоративна система підключається до всього realm і так.

### 2. Як сервіси дізнаються, хто прийшов

```
 токен Keycloak (sub = акаунт)  +  заголовок «активна організація»
        │
        ▼
 оркестратор / Studio-back:
   1. перевірити підпис, видавця, аудиторію, строк  (публічні ключі Keycloak)
   2. знайти accounts за sub
   3. знайти membership: цей акаунт у цій організації
   4. контекст = (акаунт, організація, роль)  →  права D56, фільтр і RLS (F1, F2)
```

- **Активна організація** — окремий заголовок запиту, який фронт ставить за
  вибором користувача. Сервіс перевіряє його **за членством у БД** — підробити
  не можна: немає членства — 403.
- **Акаунт у нашій БД** зʼявляється при першому вході (за `sub`) або при
  запрошенні.

### 3. Клієнти Keycloak

| Клієнт | Тип | Вхід |
|--------|-----|------|
| `tekra-face` | публічний | code + PKCE |
| `tekra-studio` | публічний | code + PKCE |
| `tekra-assistant` | публічний, десктопний | code + PKCE через системний браузер на `http://127.0.0.1:<порт>` |
| `tekra-orchestrator`, `tekra-studio-back` | ресурси (аудиторії токена) | — |

Один realm на deployment. У спільному Free — один realm на всіх, організації
розрізняються в нашій БД.

### 4. Як входить Assistant

| Що | Як |
|----|----|
| **реєстрація** Assistant і машини (I2, D76, D77) | адмін у Face видає **токен підключення**: адреса оркестратора + організація. Assistant реєструється з ним **один раз** |
| **вхід людини** (attended) | системний браузер → Keycloak → назад на `127.0.0.1`; refresh-токен зберігається через DPAPI — десктоп уже так зберігає облікові дані |
| **WS-канал** | замість логіна, пароля й назви організації — access-токен у першому повідомленні; оркестратор перевіряє його так само, як HTTP |
| **unattended** (D3b, після MVP) | окремі облікові дані самого Assistant або Device Authorization Grant — вирішити разом з unattended |

### 5. Що стає з наявним входом

| Зараз (оркестратор) | Стає |
|---------------------|------|
| вхід адміна email + пароль (`/admins/login`) | Keycloak |
| 2FA кодом на пошту | **TOTP / WebAuthn Keycloak** (вбудовано) — або сторонній плагін email-кодів (див. «Що затвердити») |
| вхід через Google (`routers/google_oauth.py`) | Google у Keycloak |
| скидання пароля | Keycloak |
| одна сесія на адміна (`active_session_id`), простій 30 хв через Redis | налаштування сесій Keycloak; обмеження однієї сесії — опційно (лімітер) |
| блокування за перебір (Redis `attempts` / `ban`) | вбудований захист Keycloak |
| сповіщення «вхід з нової IP» | прибрати або пізніше — через події Keycloak |
| запрошення (magic-link, `accept-invite`) | наш запис `memberships` + акаунт у Keycloak з листом «задайте пароль» |
| паролі юзерів десктопа, історія й строк дії (`user_passwords`, `organizations.default_password_expiration`) | політика паролів Keycloak — **одна на realm**, не на організацію |
| вхід десктопа через WS логіном і паролем | розділ 4 |

Redis лишається для обмеження частоти запитів (FastAPILimiter).

### 6. Таблиці

Прод-даних немає — переносити рядки не потрібно, таблиці змінюємо напряму.

| Зараз | Стає |
|-------|------|
| `admins`, `users` | `accounts` + `memberships` (роль, відділ, статус, хто запросив) |
| `user_passwords` | прибирається — паролі в Keycloak |
| `user_session` (мова, остання активність) | налаштування акаунта |
| налаштування адміна (тема, щільність тощо) | налаштування акаунта |
| `hardware_lock` (в обох таблицях) | прибирається — не використовується |
| `users.is_online` | прибирається — онлайн тепер стан Assistant (D76) |
| `users.pending_review` | лишається — в `memberships` |
| `admin_notifications.recipient_admin_id` | → акаунт |
| `notifications.user_id` | → акаунт |
| `approvals.decided_by_admin_id` | → акаунт |
| `*.invited_by_admin_id` | → `memberships.invited_by_account_id` |
| `user_license.user_id` (місце в ліцензії) | → членство |
| `robots.user_id` | замінюється у фазі 1 |

Точні поля — у S1, в [03-data-model.md](03-data-model.md).

## Що затвердити

| # | Питання | Пропозиція |
|---|---------|------------|
| 1 | Членство й ролі | **у нашій БД**; Keycloak — лише ідентичність; Organizations Keycloak — не в MVP |
| 2 | Другий фактор | **TOTP / WebAuthn** Keycloak замість кодів на пошту — без сторонніх плагінів; або плагін email-кодів, якщо коди на пошту обовʼязкові |
| 3 | Вхід Assistant | **системний браузер** (code + PKCE, `127.0.0.1`) + токен підключення від адміна |
| 4 | Політика паролів | **одна на deployment** (realm), а не на організацію; `default_password_expiration` прибирається |
| 5 | Одна сесія на адміна | не обмежувати — Keycloak сам керує сесіями й простоєм; обмежувач — коли знадобиться |
| 6 | «Вхід з нової IP» | прибрати зараз; повернути пізніше через події Keycloak, якщо знадобиться |
| 7 | Активна організація | заголовок у запиті, перевіряється за членством у БД |

## Реалізація (E1, 2026-10-29)

- Realm `tekra` — файл `Tekra-Orchester/docker/keycloak/import/tekra-realm.json`,
  імпортується при першому старті Keycloak; клієнти — як у розділі 3.
- Перенесено з наявного входу: блок на 10 хв після 3 невдалих спроб, простій
  сесії 30 хв, найдовша сесія 48 год, другий фактор — за бажанням (TOTP).
- Нове: пароль від 12 символів, не логін і не пошта, не 3 останні; access-токен
  5 хв (замість 24 год); самостійної реєстрації немає — лише запрошення.
- Тестові акаунти й організація — лише для локального стеку (окремий файл і
  скрипт `scripts/seed_dev_identity.py`).
- **E2:** оркестратор перевіряє токени й будує контекст «акаунт +
  організація + роль»; `admins` і `users` прибрано (міграція
  `1c8b8561ccb8`), люди — `/members`. Десктоп входить токеном Keycloak у
  першому повідомленні WebSocket.
- **Запрошення:** службовий акаунт `tekra-orchestrator` (лише керування
  користувачами realm) створює людину в Keycloak, Keycloak шле лист
  «задайте пароль» (посилання 72 год, назад — у Face); у нас — членство
  `INVITED`, яке стає `ACTIVE` при першому вході. SMTP — змінні `KC_SMTP_*`
  у realm-файлі, локально — Mailpit.

## Джерела

- [Keycloak 26.0.0 released](https://www.keycloak.org/2024/10/keycloak-2600-released.html) — Organizations офіційно з 26
- [Keycloak Organizations · issue #30180](https://github.com/keycloak/keycloak/issues/30180) — ролі організацій не в 26
- [Fine-Grained Admin Permissions for Organizations](https://www.keycloak.org/2026/05/org-fgap) — у 26.7 — ролі керування організаціями
- [OrganizationInvitationsResource (26.5)](https://www.keycloak.org/docs-api/26.5.1/javadocs/org/keycloak/admin/client/resource/OrganizationInvitationsResource.html) — запрошення
- [Server Administration Guide](https://www.keycloak.org/docs/latest/server_admin/index.html) — Google, захист від перебору, політики паролів, сесії
- [Keycloak OIDC layers](https://www.keycloak.org/securing-apps/oidc-layers) — Device Authorization Grant, `127.0.0.1` з будь-яким портом
- [Форум Keycloak про email OTP](https://forum.keycloak.org/t/we-need-to-implement-mfa-in-keycloak-26-1-3-with-totp-and-email-otp-and-want-to-confirm-if-custom-spis-are-required-for-email-otp-support-and-dual-authentication-selection/29837), [keycloak-2fa-email-authenticator](https://github.com/mesutpiskin/keycloak-2fa-email-authenticator) — коди на пошту лише плагіном
- [Ripple: Keycloak session limiting](https://docs.ripple.com/products/custody/identity-and-access/sso/keycloak-session-limiting) — обмежувач сесій, офлайн-токени
