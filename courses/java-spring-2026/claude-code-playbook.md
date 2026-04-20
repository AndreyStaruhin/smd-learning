# Claude Code Playbook
## Сценарии использования в рамках методологии

---

## Базовый принцип разделения

**Claude Project (chat)** = пространство **мышления**. Позиционные диалоги, проблематизация, онтология, рефлексия. Читает репозитории через GitHub connector.

**Claude Code** = пространство **действия**. Реальный код, реальные файлы, реальные коммиты. Работает локально с двумя репозиториями.

Не размазывайте функции: концепция → Project, действие с файлами → Code.

---

## Установка и первичная настройка

### Установка

- **JetBrains plugin** — если работаете в IntelliJ IDEA (наиболее естественно для Java)
- **Terminal** — универсально, удобно для Maven-команд
- **VS Code plugin** — если предпочитаете VS Code

### Работа с двумя репозиториями

Claude Code работает с тем репозиторием, в котором вы сейчас находитесь (открыто в IDE / `cd` в терминале). В нашей архитектуре два репо:

- `~/projects/smd-learning/` — методологические артефакты
- `~/projects/methodolog/` — проектный код

**Практический приём:** держите IntelliJ открытым в `methodolog` (основная работа), а в отдельной вкладке терминала — `cd ~/projects/smd-learning`. Когда нужно записать рефлексию — быстро переключаетесь, Claude Code дописывает, коммит, обратно.

Каждый репозиторий имеет свой `CLAUDE.md` в корне — контекстный файл, который Claude Code читает автоматически. Он задаёт "правила поведения" для этого репо.

---

## Сценарий 1. Bytecode Inspection

**Когда:** при изучении generics (type erasure), AOP (proxy-механика), records.

**Цель:** увидеть, во что реально превращается Java-код — это разрушает "магию" и формирует точную онтологию.

**Типовой запрос:**

```
В methodolog/assessment/A1/Registry.java скомпилируй generic-класс.
Запусти javap -v на target/classes/assessment/A1/Registry.class.
Найди в выводе bridge methods. Объясни их роль для JVM.
Покажи, где T стёрт в Object (или в upper bound).
```

Claude Code:
1. `mvn compile -pl :assessment`
2. Находит `.class` в `target/classes/...`
3. `javap -v -p path/to/Registry.class`
4. Парсит вывод, выделяет bridge methods, Signature attribute
5. Объясняет, связывая с теорией

**Вариация — AOP proxy:**

```
Запусти приложение с -Dspring.aop.proxy-target-class=false и с true.
Через reflection в runtime выведи имя класса прокси-объекта нашего сервиса.
Сравни. Объясни JDK dynamic proxy vs CGLIB.
```

---

## Сценарий 2. Maven Archaeology

**Когда:** Модуль M, отладка dependency conflicts, анализ проекта компании.

**2а. Dependency tree с конфликтами:**
```
mvn dependency:tree -Dverbose на реакторе methodolog.
Найди все случаи dependency mediation (nearest-wins).
Особенно jackson, slf4j, logback.
Предложи, стоит ли зафиксировать версии в <dependencyManagement>.
```

**2б. Effective POM:**
```
mvn help:effective-pom -pl :methodolog-core -Doutput=effective-pom.xml
Покажи, как parent POM + BOM вместе определяют версии.
Найди зависимость, версия которой не указана в нашем pom,
но присутствует в effective благодаря BOM.
```

**2в. Plugin timeline:**
```
mvn help:describe -Dplugin=maven-jaxb2-plugin -Ddetail=true
Покажи goals. К какой фазе привязан `generate`?
Затем mvn verify -X | grep "\[INFO\] --- " — реальная последовательность плагинов.
```

**2г. Dependency analysis:**
```
mvn dependency:analyze на methodolog-api.
Used but undeclared, Declared but unused. Обсуди каждый.
```

---

## Сценарий 3. Hibernate SQL Tracing

**Когда:** Модуль 5 — постоянно. Основной инструмент против "непредсказуемого SQL".

**Базовая настройка** в `application-test.yml`:
```yaml
spring:
  jpa:
    properties:
      hibernate:
        format_sql: true
        use_sql_comments: true
        generate_statistics: true
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE
    org.hibernate.stat: DEBUG
```

Для полной трассировки — **p6spy** (dependency + URL `jdbc:p6spy:postgresql://...`).

**3а. N+1 detection:**
```
GameRepository.findAllWithStages() — Game с коллекцией stages (OneToMany).
Integration test:
1. Вставить 5 игр с 3 этапами каждая.
2. Вызвать findAllWithStages().
3. Через Hibernate Statistics посчитать SQL.
4. Assert: либо один запрос (JOIN FETCH), либо N+1 → FAIL.

Покажи разницу SQL: LAZY без fetch, @EntityGraph, JOIN FETCH в JPQL.
```

**3б. Query plan analysis:**
```
Возьми SQL из лога для findByStatusAndCreatedByOrderByCreatedAtDesc.
Выполни через EXPLAIN ANALYZE в PostgreSQL.
Seq scan? Предложи индекс через Flyway миграцию.
Перезапусти, покажи изменение плана.
```

**3в. Transaction boundaries:**
```
DEBUG на org.springframework.transaction.
OrderController.createOrder() → сервис создаёт Order → вызывает AuditService.logAction (REQUIRES_NEW).
Покажи логи: создание outer, suspend outer, создание inner, commit inner, resume outer, commit outer.
Границы через SQL (BEGIN/COMMIT).
```

---

## Сценарий 4. Code Review — Совет мудрецов

**Когда:** раз в неделю на весь накопленный код или точечно на PR/ветку.

**Механика:** четыре эксперта-линзы, каждый из которых оценивает код через свою сквозную тему. Каждый даёт 1–2 проблематизирующих замечания. Разногласия между экспертами — ценность, не сглаживай.

```
Code review файлов, изменённых в последнем коммите (git diff HEAD~1).
Четыре отзыва от Совета мудрецов:

1. "Страж AOP/Proxy":
   Где здесь proxy? Self-invocation? Что реально вызывается в runtime?
   Аспекты применяются к тому, к чему ожидается?
   Транзакционные границы — через proxy или нет?

2. "Страж Testing":
   Как это проверяется? Какой уровень теста нужен (unit/slice/integration)?
   Тестируется поведение или реализация?
   Что сломается при рефакторинге? Где хрупкие моки?

3. "Страж SB3":
   Это решение из Spring Boot 2 мира или используются возможности SB3?
   Virtual threads, HTTP interfaces, Problem Details, Observation API —
   есть ли SB3-идиоматичный путь? Что изменилось в этой области?

4. "Страж Clean Architecture":
   Где границы слоёв? Кто от кого зависит?
   Domain знает про фреймворк? Нарушается ли DIP?
   Если завтра заменить persistence — сколько файлов тронешь?
   Naming, SRP, размер методов — Clean Code.

В конце — какое замечание наиболее приоритетное и почему.
```

**Вариация — фокусный review:**
```
Code review только через линзу "Страж Testing".
Покажи: где тестов нет, где тест хрупкий, где уровень теста неверный.
```

---

## Сценарий 5. Экзамен модуля — защита перед комиссией

**Когда:** в конце каждого модуля. Блокирует переход.

**Механика:** задача от senior tech lead, 60 минут на решение, затем последовательная защита перед Советом мудрецов. Каждый эксперт задаёт 1–2 вопроса из своей линзы, учащийся защищается, потом следующий. Как защита курсовой в МФТИ.

```
Экзамен Модуля [X].

Фаза 1 — Постановка (5 мин):
Твоя роль: senior tech lead на onboarding нового инженера.
Я 2 дня на проекте, ты даёшь задачу из бэклога.
[Задача из контекста модуля, индустриально-реалистичная]

Фаза 2 — Решение (60 мин):
Работаю в Claude Code. Ты молчишь. Могу спросить по требованиям, не по технике.

Фаза 3 — Защита перед комиссией (20-25 мин):
Последовательно, по одному эксперту:

1. Страж AOP/Proxy (3-5 мин):
   Вопросы о proxy-механике, self-invocation, аспектах в решении.
   Я отвечаю, защищаюсь.

2. Страж Testing (3-5 мин):
   Вопросы о тестовой стратегии, уровнях тестов, покрытии.
   Я отвечаю, защищаюсь.

3. Страж SB3 (3-5 мин):
   Вопросы об идиоматичности решения для Spring Boot 3.
   Я отвечаю, защищаюсь.

4. Страж Clean Architecture (3-5 мин):
   Вопросы о слоях, зависимостях, DIP, naming, SRP.
   Я отвечаю, защищаюсь.

Фаза 4 — Вердикт (5 мин):
Общая оценка комиссии. Каждый эксперт — pass/concern.
Итог: Passed at Constructive / Passed at Transfer / Not passed.

Планку не снижай. Evidence = Constructive.
```

После экзамена — запись в evidence-ledger и reflection-log через Claude Code.

---

## Сценарий 6. OpenRewrite миграции

**Когда:** Модуль 0 (ассессмент 2→3), потом как инструмент рефакторинга.

```
В methodolog/assessment/A5/legacy старый Spring Boot 2 код.

1. Добавь openrewrite-maven-plugin + recipe
   org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_2
2. mvn rewrite:dryRun — покажи diff.
3. mvn rewrite:run — применить.
4. Покажи diff. Что автоматизировано, что руками?
5. Напиши кастомный recipe: все @Autowired на полях → constructor injection.
```

---

## Сценарий 7. Парное программирование

**Когда:** этап мыследействия СМД-сессии.

Правила (уже должны быть в methodolog/CLAUDE.md):
```
"Парное программирование":
- Я driver, ты navigator
- Не пиши код за меня, если не попросил
- Уточни замысел перед началом
- По ходу — один-два комментария за раз
- Напоминай о краях: null? пустая коллекция? тест?
- Застрял — не решай: "что пробовал? какая гипотеза?"
- Только "напиши" — пишешь
- В конце — один финальный комментарий качества сеанса
```

---

## Сценарий 8. Schema Building (интеграция с Project)

**Когда:** построение ключевых онто-схем.

```
Эксперимент для схемы "IoC lifecycle":

LifecycleTrace, имплементирующий ВСЕ hooks:
- BeanNameAware, BeanFactoryAware, ApplicationContextAware
- InitializingBean, DisposableBean
- @PostConstruct, @PreDestroy
- BeanPostProcessor (отдельный, логирующий before/after)
- SmartInitializingSingleton
- ApplicationListener<ContextRefreshedEvent>

Каждый hook: лог "step X: Y called".

Запусти Spring Boot. Покажи точный порядок логов.
Это — фактическая схема. Возьму в Claude Project для визуализации.
```

Затем в Claude Project:
```
Наблюдаемая последовательность (прикладываю лог).
Построй интерактивную схему жизненного цикла bean, где каждый этап
соответствует наблюдению из лога.
```

Схема получает **эмпирическую основу** — FPF Evidence-дисциплина.

---

## Сценарий 9. Запись в артефакты методологического репо

**Важно:** именно Claude Code пишет в `smd-learning`, не Claude Project.

**9а. Запись в reflection-log:**
```
cd ~/projects/smd-learning
Добавь в courses/java-spring-2026/reflection-log.md новую запись
В НАЧАЛО (новые записи сверху) по шаблону:

## Сессия #[N] | [дата] | Модуль [X] | Тема: [...]
### Контекст
[из нашего сеанса]
### Позиции в сессии
[какие были в ходе сессии Claude Project]
### Сдвиг содержания
[формулировка из чата Claude Project]
### Сдвиг процесса
[формулировка]
### Открытые вопросы
- [ ] ...
### Evidence-претензия
[коммит в methodolog]
### Артефакты
- code: methodolog@[SHA]

Коммит: "session #[N]: [краткое название темы]"
```

**9б. Запись в evidence-ledger:**
```
Добавь строку в evidence-ledger.md:
| [новый ID] | M[X] | [claim] | [R/T/C] | [evidence link] | [дата] | 🟡 in-progress |

Коммит: "evidence: claim #[ID] in-progress"
```

**9в. Переключение активного модуля:**
```
Закрываю Модуль [X], начинаю Модуль [Y].
1. Скопируй содержимое modules/module-[Y]-[name].md в active-module.md
2. Коммит: "switch active module: M[X] -> M[Y]"
```

**9г. Создание meta-session записи:**
```
Создай файл meta-sessions/ms-[NN]-[YYYY-MM-DD].md.
Содержимое (передаю в теле запроса):
[полный текст мета-сессии из чата Claude Project]

Коммит: "meta-session #[NN]: [главный инсайт]"
```

---

## Что НЕ делать с Claude Code

- ❌ Не проси объяснять онтологию — это работа Claude Project.
- ❌ Не используй как ChatGPT ("напиши класс X") — используй как парного senior-а.
- ❌ Не соглашайся на "рабочее решение" без запуска и проверки.
- ❌ Не забывай коммитить — git log = часть reflection-log.
- ❌ Не путай обычную разработку и мыследействие в СМД-сессии (во втором — правила парного программирования).

---

## Быстрая шпаргалка путей

```
Код assessment:           methodolog/assessment/A1/...
Код проекта:              methodolog/methodolog-core/...
Reflection-log:           smd-learning/courses/java-spring-2026/reflection-log.md
Evidence-ledger:          smd-learning/courses/java-spring-2026/evidence-ledger.md
Heat-map:                 smd-learning/courses/java-spring-2026/heat-map.md
Active module:            smd-learning/courses/java-spring-2026/active-module.md
Schemes:                  smd-learning/courses/java-spring-2026/schemes/
Meta-sessions:            smd-learning/courses/java-spring-2026/meta-sessions/
```
