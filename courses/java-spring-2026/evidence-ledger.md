# Evidence Ledger — Java Spring Boot Mastery

> Реестр проверяемых претензий на освоение тем курса.
>
> **Level:** R (Reproductive — знакомый паттерн в знакомой ситуации), T (Transfer — паттерн в новой ситуации), C (Constructive — новое решение для новой ситуации).
>
> **Статусы:** ⬜ planned, 🟡 in-progress, ✅ closed, ❌ falsified (claim был, Evidence не подтвердилось — важнейшая категория честности).

---

## Текущий реестр

| ID | Модуль | Claim | Level | Evidence | Дата | Статус |
|----|--------|-------|-------|----------|------|--------|
| 000 | Infra | Двухрепозиторная архитектура настроена, Claude Project и Code работают | R | smd-learning@[SHA0], methodolog@[SHA0] | [YYYY-MM-DD] | ✅ closed |

---

## Метрики (обновлять после каждой мета-сессии)

- **Всего claims:** 8
- **Closed:** 2 (25%)
- **In-progress:** 3
- **Falsified:** 3
- **Claims по уровням:** R=3, T=3, C=0, meta=2

---

| 001 | M0 | Знаю синтаксис records и автогенерируемые методы (equals/hashCode/toString/accessors/canonical ctor) | R | A1.1 | 2026-04-19 | ✅ closed |

| 002 | M0 | Знаю compact constructor для валидации в record | R | A1.1 (освоен в ходе разбора, не применил самостоятельно) | 2026-04-19 | 🟡 in-progress |

| 003 | M0 | Различаю record как nominal tuple / value-based class от JVM value type (Valhalla) | T | A1.1 | 2026-04-19 | 🟡 in-progress |

| 004 | M0 | Удерживаю контракт equals/hashCode семантически (только поля класса, согласованность двух методов) | T | A1.1 | 2026-04-19 | 🟡 in-progress |

| 005 | M0 | Моделирую immutability через final + отсутствие сеттеров как рефлекс, не после замечания | T | A1.1 (первая версия была mutable) | 2026-04-19 | ❌ falsified |

| 006 | M0 [meta] | Есть рефлекс "запустить компилятор / REPL перед словом готово" | — | A1.1 | 2026-04-19 | ❌ falsified |

| 007 | M0 [meta] | Есть рефлекс "fix one → check all" при обнаружении ошибки | — | A1.1 (нашёл в equals, в hashCode — только после указания) | 2026-04-19 | ❌ falsified |
