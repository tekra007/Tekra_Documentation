# TEKRA RPA

## Єдина deployment-модель платформи

> **Автор:** Oleksii Kulinskyi, TEKRA · **Дата:** 10/7/2026
>
> **Статус:** ⚠️ **пропозиція, не затверджено** (уточнено 2026-10-13). Канон
> лишається ТЗ [`tz-lifecycle.md`](tz-lifecycle.md) (D32); рішення під цей
> документ не приймаються, доки його не затверджено. Розбір —
> [29-deployment-model.md](../29-deployment-model.md).
>
> Оригінал: `TEKRA_Studio_Базова_модель_розгортання1.pdf` — у репозиторій
> покласти як [`deployment-model.pdf`](deployment-model.pdf). Цей файл — дослівне
> перенесення тексту; схеми перемальовані текстом. У спірному випадку звіряти з
> PDF.

---

**Єдина deployment-модель платформи** - з двома режимами розгортання для клієнтів, але з однією продуктовою архітектурою:

```
                    ┌──────────────────────────┐
                    │       MARKETPLACE        │
                    │       GLOBAL / ONE       │
                    │                          │
                    │  Components              │
                    │  Automations / Robots    │
                    │  Processes / Templates   │
                    │  Connectors              │
                    └────────────┬─────────────┘
                                 │
                    distribute / install / update
                                 │
            ┌────────────────────┴───────────────────┐
            ▼                                        ▼
   ┌──────────────────────┐              ┌──────────────────────┐
   │     FREE / TRIAL     │              │  PAID / ENTERPRISE   │
   │  SHARED ENVIRONMENT  │              │ ISOLATED DEPLOYMENT  │
   └──────────┬───────────┘              └──────────┬───────────┘
        ┌─────┼─────┐                     ┌─────────┴─────────┐
        ▼     ▼     ▼                     ▼                   ▼
   Tenant A Tenant B Tenant C        Customer A          Customer B
        └─────┼─────┘                     │                   │
              ▼                           ▼                   ▼
      ┌───────────────┐          ┌──────────────┐    ┌──────────────┐
      │ Shared Studio │          │    Studio    │    │    Studio    │
      │ Shared Orch.  │          │      +       │    │      +       │
      │ Shared Infra  │          │ Orchestrator │    │ Orchestrator │
      └───────┬───────┘          │  Customer A  │    │  Customer B  │
              │                  └──────┬───────┘    └──────┬───────┘
        ┌─────┴─────┐               ┌───┴───┐           ┌───┴───┐
        ▼           ▼               ▼       ▼           ▼       ▼
    Assistant   Assistant     Assistant Assistant Assistant Assistant
    Machine A   Machine B       PC-01     PC-02     PC-01    Server
```

## 1. Marketplace - один і глобальний

Marketplace **не прив'язаний до конкретного клієнта**.

Він є єдиним каталогом:

- компонентів;
- Composite Components;
- Automation / Robots;
- Business Tasks;
- Processes / шаблонів;
- Connectors;
- інших артефактів, які ви вирішите розповсюджувати.

Клієнт знаходить артефакт у Marketplace і встановлює/імпортує його у своє середовище.

## 2. Free / Trial - shared

Для безкоштовного доступу **не створюємо окремий deployment на кожного користувача**. Маємо:

```
                 FREE / TRIAL PLATFORM
                          │
        ┌─────────────────┼─────────────────┐
     Tenant A          Tenant B          Tenant C
        └─────────────────┼─────────────────┘
                          │
                    Shared Studio
                 Shared Orchestrator
                Shared Infrastructure
```

Але кожен Tenant **логічно ізольований**:

- свої користувачі;
- свої ролі;
- свої процеси;
- свої Business Tasks;
- свої Automations;
- свої компоненти/налаштування;
- свої credentials;
- свої execution;
- свої логи;
- свої дані.

**Tenant A не бачить Tenant B.**

При цьому фізично вони можуть працювати на спільному cloud infrastructure.

Free/Trial просто має обмеження:

- ліміти execution
- ліміти concurrent jobs
- ліміти storage
- ліміти кількість Assistant
- ліміти доступні функції
- ...

## 3. Paid / Enterprise - isolated

Коли клієнт переходить на платний рівень, він отримує **власний deployment**.

```
Customer A
│
├── Studio
├── Orchestrator
├── Database
├── Queue / Cache
├── Configuration
├── Secrets / Credentials
└── Logs / Execution data
```

А інший клієнт:

```
Customer B
│
├── Studio
├── Orchestrator
├── Database
├── Queue / Cache
├── Configuration
├── Secrets / Credentials
└── Logs / Execution data
```

Фізично це все може знаходитися:

```
AWS
│
├── Customer A deployment
├── Customer B deployment
├── Customer C deployment
└── ...
```

або в Azure / GCP / іншій інфраструктурі.

Тобто **окремий deployment ≠ окремий фізичний сервер**.

## 4. Studio + Orchestrator

Важливо!!! Архітектурне розділення!

Для користувача: **Customer Deployment = Studio + Orchestrator**

Але технічно це не обов'язково один Docker container.

Наприклад:

```
Customer A Deployment
│
├── Studio
│     ├── container
│     └── container ...
│
├── Orchestrator
│     ├── API container
│     ├── Scheduler container
│     ├── Worker container
│     └── ...
│
├── DB
└── supporting services
```

Тобто **ізольований deployment - це одиниця розгортання клієнта**, а не обов'язково один контейнер. Docker/Kubernetes дозволять потім масштабувати його без зміни продуктової моделі.

## 5. Assistants - окремий рівень

Assistant **не входить у deployment Studio/Orchestrator як звичайний container**.

Він встановлюється туди, де фактично має виконуватися автоматизація:

```
Customer A Deployment
        │
        ├───────────────────┐
        ▼                   ▼
    Assistant           Assistant
   Windows PC        Windows Server
        │                   │
        ▼                   ▼
      Apps                Apps
      BAS                 Browser
      Excel               Files
      etc.                etc.
```

Assistant реєструється у відповідному Orchestrator.

Наприклад:

```
Assistant #27
   ↓
Customer A
   ↓
PROD
   ↓
Machine "Accounting-PC-01"
```

## 6. Перехід: Free → Paid

Це має бути **не міграція в інший продукт**, а зміна deployment model.

```
   FREE / TRIAL
        │
     Tenant A
        │
    "Upgrade"
        │
        ▼
┌─────────────────────┐
│ Customer A          │
│ isolated deployment │
└─────────────────────┘
```

Тобто клієнт зберігає:

- акаунт;
- свої процеси;
- свої Automation;
- свої компоненти;
- свої налаштування;
- свої дані - в межах того, що дозволяє тариф/політика міграції.

А інфраструктура переходить із: **Shared → Isolated.**

## 7. Спрощена загальна модель

**Архітектурний принцип**: платформа складається з єдиного глобального Marketplace та клієнтських середовищ, які можуть працювати у **shared multi-tenant режимі** для Free/Trial або як **ізольовані deployment'і** для **платних та Enterprise-клієнтів**; кожне клієнтське середовище містить **Studio** та **Orchestrator**, а **Assistant** встановлюється окремо на машинах, де виконується автоматизація.

І якщо зовсім коротко, пропонована базова deployment-архітектура:

```
                     PLATFORM
                         │
          ┌──────────────┴──────────────┐
     MARKETPLACE                CUSTOMER ENVIRONMENTS
     ONE / GLOBAL                       │
                               ┌────────┴────────┐
                             FREE              PAID
                            SHARED           ISOLATED
                               │                 │
                          Many Tenants      One Customer
                               │                 │
                         Studio + Orch.    Studio + Orch.
                               └────────┬────────┘
                                        │
                                    ASSISTANTS
                                        │
                                  REAL MACHINES
```
