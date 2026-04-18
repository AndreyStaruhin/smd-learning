# SMD Learning — Методологический корпус

Персональный репозиторий методологически-дисциплинированных курсов по системо-мыследеятельностному подходу с опорой на First Principles Framework (FPF).

**Приватный репозиторий.** Содержит личные рефлексии, evidence-артефакты, схемы мышления.

---

## Структура

```
smd-learning/
├── README.md                   ← этот файл
├── CLAUDE.md                   ← контекст для Claude Code в этом репо
└── courses/                    ← все курсы
    └── java-spring-2026/       ← активный курс
        ├── course-mechanics.md
        ├── project-systemprompt.md
        ├── active-module.md    ← активный модуль в каждый момент
        ├── modules/            ← все модули курса (архив + новые)
        │   └── module-0-audit.md
        ├── claude-code-playbook.md
        ├── reflection-log.md
        ├── evidence-ledger.md
        ├── heat-map.md         ← появится после M0
        ├── onto-map.md         ← появится позже
        ├── meta-sessions/      ← мета-сессии раз в 2 недели
        └── schemes/            ← построенные схемы (mermaid, svg)
```

## Текущий активный курс

**`courses/java-spring-2026/`** — Java + Spring Boot 3 как способ мышления.

Старт: [дата].

Связан со сквозным проектным репо [**methodolog**](../methodolog).

## Принципы работы с репо

1. **Это — источник правды** для методологической работы. Всё, что не в коммите — не случилось.
2. **Claude Code пишет, Claude Project читает** через GitHub connector.
3. **Коммиты как Speech Acts** — каждый осмысленный шаг курса имеет свой коммит с говорящим message.
4. **Артефакты — живые**: reflection-log, evidence-ledger, onto-map эволюционируют.
5. **Privacy boundary**: этот репо **не** идёт в public/portfolio/demo. Личная интеллектуальная собственность.

## Коммит-конвенция

Для артефактов курса:
- `session #[N]: [тема]` — запись в reflection-log
- `evidence: claim #[ID] [status]` — изменение в ledger
- `meta-session #[NN]: [инсайт]` — мета-запись
- `switch active module: M[X] -> M[Y]` — переключение модуля
- `scheme: [name] v[N]` — новая/обновлённая схема

Для методологических документов:
- `methodology: [что и почему изменено]` — правки после мета-сессий

## Масштабирование

Когда начнётся второй курс — появится `courses/[новый-курс]/` рядом, с такой же внутренней структурой. Методологический корпус накапливается, не мигрирует.
