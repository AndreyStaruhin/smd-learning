# Модуль 0: Аудит онтологических пробелов
## Жёсткий ассессмент — "без обид и по-чесноку"

---

## Цель модуля

Честно измерить **текущее состояние** по всем ключевым темам курса через выполнение конкретных задач в коде. Результат — `heat-map.md` с цветовой маркировкой, определяющая приоритизацию модулей.

---

## Правила ассессмента

**Для пользователя:**
- Не гуглить, не смотреть StackOverflow, не спрашивать у Claude "как сделать". Это диагностика, не экзамен.
- "Не знаю" — **самый ценный сигнал**, не позор.
- Если на задачу уходит больше 20 минут активного мышления без сдвига — "не справился, время вышло". Это тоже сигнал.

**Для Claude:**
- Позиция [Симулянт-эксперт] (tech lead, техническое собеседование).
- После каждой задачи — оценка по трём метрикам: (1) работает/нет, (2) онтологическое понимание: понимает/заучил/не знает, (3) обоснование: осознанное/интуитивное/отсутствует.
- Не подсказывай до оценки. После — короткий комментарий, что упущено.

---

## Инфраструктура ассессмента

Перед первой сессией в репозитории `methodolog`:

1. Создать папку `assessment/` с подпапками `A1/`, `A2/`, `A3/`, `A4/`, `A5/`.
2. В корне `methodolog/` — минимальный `pom.xml` с parent `spring-boot-starter-parent:3.2.x` и модулем `assessment`.
3. Установить JDK 21 (или 17 если 21 недоступен), Maven 3.9+, Docker для PostgreSQL через docker-compose.

Конкретный скелет строится **в задаче A4.5** как часть ассессмента — это намеренно.

Пути к файлам ассессмента:
- Код → `methodolog/assessment/A1/...`, `assessment/A2/...` и т.д.
- Записи сессий → `smd-learning/courses/java-spring-2026/reflection-log.md` (через Claude Code).
- Evidence → `smd-learning/courses/java-spring-2026/evidence-ledger.md`.

---

# Сессия A1 — Java Core + современные фичи + Generics (2 часа)

Проверяем: владение современной Java (Records, Sealed, Pattern Matching, Streams) и — главное — **генериками на онтологическом уровне**.

## A1.1 [R] — Records vs classic class (10 мин)

> В `methodolog/assessment/A1/PairVariants.java` реализуй неизменяемую пару `(String, Integer)` двумя способами: (а) классом в стиле Java 8 с final полями, конструктором, геттерами, equals, hashCode, toString; (б) record-ом. В комментарии покажи, какие методы record предоставил автоматически и что меняется, если нужна валидация в конструкторе.

**Критерии:** работает? понимает nominal tuple с автоматическим equals по структуре? знает compact constructor?

## A1.2 [T] — Sealed classes для domain union (15 мин)

> В `assessment/A1/GameState.java` смоделируй domain union: `Draft(LocalDate plannedStart)`, `Active(LocalDateTime startedAt, int participantsCount)`, `Finished(LocalDateTime endedAt, String summary)`, `Cancelled(String reason)`. Используй `sealed interface` + record implementors. Напиши метод `String describe(GameState state)` через switch expression с pattern matching.

**Критерии:** работает? понимает compile-time exhaustiveness? понимает преимущество перед enum+instanceof?

## A1.3 [T] — Stream API в глубину (15 мин)

> Дан `List<Game>` с полями `name, status, createdBy`. Без явных циклов:
> 1. `Map<Class<?>, List<String>>` — имена игр по типу состояния.
> 2. Количество игр каждого автора в активном состоянии.
> 3. Автор с максимумом созданных игр.

**Критерии:** работает? понимает разницу collect/reduce, intermediate/terminal, lazy evaluation?

## A1.4 [C] — Generics и проблема `new T()` (20 мин)

> В `assessment/A1/Registry.java` — `Registry<T>` с `register(String, T)`, `T get(String)`, `List<T> getAll()`. Добавь `T createDefault()` через no-arg constructor. Компилятор не позволит `new T()` — почему? Решение двумя способами, объяснения в комментариях.

**Критерии:** ключевой тест. Понимает type erasure в runtime? Знает Class<T> pattern и Supplier<T>?

## A1.5 [C] — Type erasure и граничные случаи (20 мин)

> В `assessment/A1/ErasureProblems.java` — два перегруженных метода `void process(List<String>)` и `void process(List<Integer>)`. Компилятор ругается — почему? Два исправления, сравни. Затем `<T> T[] createArray(int size)` — почему не рад? Объясни и обойди.

**Критерии:** ядро теста. Bridge methods, heap pollution, reifiable vs non-reifiable? Знает `@SafeVarargs`, `Array.newInstance()`?

## Оценка A1

```
- Modern Java (records, sealed, pattern matching): 🟢/🟡/🟠/🔴
- Stream API: 🟢/🟡/🟠/🔴
- Generics (базовые): 🟢/🟡/🟠/🔴
- Type erasure онтология: 🟢/🟡/🟠/🔴
- Обоснование решений: сильное / среднее / слабое
```

---

# Сессия A2 — Spring Core + IoC + Bean Lifecycle + AOP (2 часа)

## A2.1 [R] — Bean scopes (15 мин)

> В `assessment/A2/ScopesDemo/` — Spring Boot приложение с `SingletonCounter`, `PrototypeCounter`, `RequestCounter`. В `@RestController` внедри все, на каждый GET `/counters` инкрементируй и верни значения. Запусти, сделай 5 запросов, объясни.

**Критерии:** понимает, что prototype inject в singleton = один раз? Знает `ObjectFactory` / `@Lookup` для обхода?

## A2.2 [T] — Bean lifecycle hooks (15 мин)

> `LifecycleAwareBean` реализует `InitializingBean`, `DisposableBean`, имеет `@PostConstruct`, `@PreDestroy`, `@Autowired` setter, конструктор. Лог каждого вызова. Запусти, объясни **точный** порядок.

**Критерии:** понимает фазы: instantiation → DI → Aware → BPP.before → init hooks → BPP.after → ready?

## A2.3 [T] — Circular dependency (15 мин)

> `ServiceA` и `ServiceB` внедряются друг в друга через **constructor injection**. Запусти, получи ошибку, объясни. Замени на setter или `@Lazy` — объясни, почему работает и какие риски.

**Критерии:** понимает, почему constructor требует полной инициализации? Знает про early bean reference?

## A2.4 [C] — AOP и self-invocation (25 мин)

> Аннотация `@Timed`, `@Aspect` с `@Around`. `serviceMethodA` вызывает `this.serviceMethodB()` (оба `@Timed`). `serviceMethodB` не замерится. Почему? Два способа обхода.

**Критерии:** критический тест. Понимает proxy-based AOP? Знает `AopContext.currentProxy()`, AspectJ LTW?

## A2.5 [C] — BeanFactoryPostProcessor vs BeanPostProcessor (15 мин)

> Сценарий, где нужен BFPP, а BPP не подойдёт. И наоборот. Аргументация в `assessment/A2/BeanPostProcessorsCompared.md`.

**Критерии:** понимает: BFPP до создания (BeanDefinitions), BPP после (объекты)?

## Оценка A2
```
- IoC и scopes: 🟢/🟡/🟠/🔴
- Bean lifecycle: 🟢/🟡/🟠/🔴
- Dependency resolution: 🟢/🟡/🟠/🔴
- AOP (proxy онтология): 🟢/🟡/🟠/🔴
- Extension points (BFPP/BPP): 🟢/🟡/🟠/🔴
```

---

# Сессия A3 — Spring Boot + Web + Persistence (2 часа)

## A3.1 [R] — REST CRUD (15 мин)

> `GameController` с CRUD для `Game (id, name, status, createdBy)`. Обработка 404 через `@ResponseStatus` и через `ResponseEntity`. Оба варианта.

## A3.2 [T] — Кастомная валидация (15 мин)

> `@NotEmptyIfActive` на уровне класса Game: если `status == ACTIVE`, то `name` не пустой. Через `ConstraintValidator`. Применяй на POST через `@Valid`.

**Критерии:** field-level vs class-level валидация? Группы валидации?

## A3.3 [T] — ControllerAdvice и exception mapping (10 мин)

> `@ControllerAdvice` с `@ExceptionHandler` для `EntityNotFoundException` → 404, `ValidationException` → 400 с телом. Как Spring выбирает handler?

## A3.4 [C] — Spring Data JPA query derivation (20 мин)

> `GameRepository` с `findByStatusAndCreatedBy(...)` и `findByNameContainingIgnoreCaseOrderByCreatedAtDesc(...)`. Включи SQL-логирование, покажи SQL. Откуда Spring Data "знает"?

**Критерии:** ключевой тест. Понимает parsing грамматики → Criteria API? Плюсы/минусы vs `@Query` vs Criteria?

## A3.5 [C] — @Transactional propagation (25 мин)

> `ServiceA.methodA()` (`@Transactional`) вызывает `ServiceA.methodB()` (`@Transactional(REQUIRES_NEW)`) через `this`. Затем ServiceB инжектится в A, вызов через `serviceB.methodB()`. Сколько транзакций в каждом случае? Debug-лог `org.springframework.transaction=DEBUG`. Последствия ошибки в methodB?

**Критерии:** критический тест. AOP-прокси и self-invocation? Семантика propagation levels?

## Оценка A3
```
- REST basics: 🟢/🟡/🟠/🔴
- Validation: 🟢/🟡/🟠/🔴
- Exception handling: 🟢/🟡/🟠/🔴
- Spring Data JPA (онтология): 🟢/🟡/🟠/🔴
- Transactional propagation: 🟢/🟡/🟠/🔴
- SQL observability habit: 🟢/🟡/🟠/🔴
```

---

# Сессия A4 — Testing + Maven + Codegen (2 часа)

## A4.1 [R] — Unit test с Mockito (10 мин)

> Unit test для `GameService.createGame()` с mocked `GameRepository`. Верификация вызова.

## A4.2 [T] — Slice test (15 мин)

> `@WebMvcTest(GameController.class)` с `MockMvc`. POST с валидным/невалидным телом. Почему быстрее `@SpringBootTest`? Что инжектится, что нет?

**Критерии:** "срезы" контекста? Знает `@DataJpaTest`, `@JsonTest`?

## A4.3 [T] — Testcontainers (15 мин)

> Integration test с PostgreSQL через Testcontainers. Что до запуска теста?

## A4.4 [C] — Maven archaeology (25 мин)

> `mvn dependency:tree` на реальном проекте. Найди транзитивную зависимость, приходящую по нескольким путям. `mvn dependency:tree -Dverbose` — какая версия победила? `mvn help:effective-pom` — что даёт? Внеси конфликт версий, разреши через `<dependencyManagement>` и `<exclusions>`, сравни.

**Критерии:** ключевой тест. nearest-wins? BOM? Разница `<dependency>` vs `<dependencyManagement>`?

## A4.5 [C] — XSD → Java → Domain через MapStruct (35 мин)

> XSD:
> ```xml
> <xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema">
>   <xs:element name="GameScenario">
>     <xs:complexType><xs:sequence>
>       <xs:element name="title" type="xs:string"/>
>       <xs:element name="participantsMin" type="xs:int"/>
>       <xs:element name="participantsMax" type="xs:int"/>
>       <xs:element name="stages" minOccurs="1" maxOccurs="unbounded">
>         <xs:complexType><xs:sequence>
>           <xs:element name="name" type="xs:string"/>
>           <xs:element name="durationMinutes" type="xs:int"/>
>         </xs:sequence></xs:complexType>
>       </xs:element>
>     </xs:sequence></xs:complexType>
>   </xs:element>
> </xs:schema>
> ```
>
> Задачи (в рамках реального multi-module Maven проекта methodolog, который нужно инициализировать сейчас):
> 1. Сделать parent POM с дочерним `methodolog-integration`.
> 2. Подключить jaxb2-maven-plugin к нему, генерация в `target/generated-sources/jaxb/`.
> 3. `mvn generate-sources`. Найти классы. В какой фазе?
> 4. Domain: record `GameTemplate`, не один-к-одному с JAXB-моделью.
> 5. `@Mapper` из MapStruct: JAXB → domain.
> 6. Тест: десериализация XML → JAXB → domain.

**Критерии:** работает end-to-end? Понимает compile-time codegen vs runtime reflection? Почему MapStruct, а не BeanUtils?

## Оценка A4
```
- Testing (unit, slice, integration): 🟢/🟡/🟠/🔴
- Testcontainers: 🟢/🟡/🟠/🔴
- Maven (lifecycle, phases, plugins): 🟢/🟡/🟠/🔴
- Maven dependency management: 🟢/🟡/🟠/🔴
- Code generation (JAXB, MapStruct): 🟢/🟡/🟠/🔴
```

---

# Сессия A5 — Operations + Security + миграция 2→3 (2 часа)

## A5.1 [R] — Profiles и externalized config (10 мин)

> Три профиля: `dev` (PostgreSQL localhost), `test` (H2), `prod` (PostgreSQL + env URL). `application-{profile}.yml`. Запусти с разными.

**Критерии:** precedence (default → profile → env → command-line)?

## A5.2 [T] — Actuator (15 мин)

> Включи. Открой `/health`, `/metrics`, `/env`, `/configprops`. Что доступно без auth в Boot 3? Кастомный HealthIndicator для внешнего HTTP-сервиса.

## A5.3 [T] — Spring Security + method security (20 мин)

> HTTP Basic, in-memory `admin:admin`, `user:user`. `/games/**` — authenticated; POST/DELETE — только ADMIN. `@PreAuthorize("hasRole('ADMIN')")` на сервис-методе. Как взаимодействуют URL-based и method security?

**Критерии:** понимает filter chain? Знает `SecurityFilterChain` (новый API, без `WebSecurityConfigurerAdapter`)?

## A5.4 [C] — Миграция Spring Boot 2 → 3 (25 мин)

> Старый код (даю):
> ```java
> @Configuration
> @EnableWebSecurity
> public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
>     @Override
>     protected void configure(HttpSecurity http) throws Exception {
>         http.authorizeRequests()
>             .antMatchers("/admin/**").hasRole("ADMIN")
>             .anyRequest().authenticated()
>             .and().httpBasic();
>     }
> }
> 
> import javax.persistence.*;
> @Entity
> public class Game { /* ... */ }
> 
> // application.properties
> management.endpoints.web.exposure.include=*
> spring.security.user.name=admin
> ```
>
> 1. Перечисли **все** ломающие изменения.
> 2. Выполни миграцию вручную. Скомпилируй.
> 3. OpenRewrite recipe для той же миграции. Сравни.

**Критерии:** понимает Jakarta EE namespace как **фундаментальный** сдвиг (все транзитивные либы)? Micrometer 1.10+?

## A5.5 [C] — GraalVM native hints (20 мин)

> В `assessment/A5/NativeHints.md`:
> 1. Почему reflection (Spring Data, Jackson) проблематична для native?
> 2. Какие классы зарегистрировать для reflection в methodolog?
> 3. `@RegisterReflectionForBinding` и `RuntimeHints` в Boot 3?
> 4. Ограничения native (dynamic class loading, build-time vs run-time init)?
>
> Попробуй `mvn -Pnative native:compile` (если GraalVM установлен).

**Критерии:** понимает сдвиг dynamic JVM → AOT? Когда native нужен (serverless, scale-to-zero), когда overkill?

## Оценка A5
```
- Profiles и config: 🟢/🟡/🟠/🔴
- Actuator: 🟢/🟡/🟠/🔴
- Spring Security (новый API): 🟢/🟡/🟠/🔴
- Spring Boot 2→3 migration: 🟢/🟡/🟠/🔴
- GraalVM native: 🟢/🟡/🟠/🔴
```

---

# Финальная сессия — Heat-Map Construction (60-90 мин)

**Формат:** мета-сессия, Claude только в [Методолог].

## Процедура

**Шаг 1. Агрегация (20 мин).** Перевод 25 задач в цвета 🟢/🟡/🟠/🔴.

**Шаг 2. Группировка по темам (15 мин).** 25 задач → 15-18 тематических областей, каждая получает усреднённый цвет.

**Шаг 3. Приоритизация модулей (20 мин).**
- 🔴 — критический приоритет (P0)
- 🟠 — приоритет 1 (основная масса)
- 🟡 — приоритет 2 (быстрее, с акцентом на онтологию)
- 🟢 — минимальный (только обновление под SB3)

**Шаг 4. Фиксация в `heat-map.md`** (через Claude Code в `smd-learning/courses/java-spring-2026/heat-map.md`).

Формат:
```markdown
# Heat Map — [дата]

## Диагностика по темам

| Тема | Уровень | Приоритет | Комментарий |
|------|---------|-----------|-------------|
| Modern Java (17→21) | 🟢 | Low | Только обновить на 21 |
| Generics + Type Erasure | 🟠 | P1 | "Сахар" — сломать |
| Spring IoC / Bean Lifecycle | 🟡 | P1 | Онтологии нет |
| Spring AOP / self-invocation | 🟠 | P1 | Критичный onto-пробел |
| Spring Data JPA | 🟡 | P1 | + SQL observability |
| Transactional propagation | 🔴 | **P0** | В первом модуле после 0 |
| Testing (unit, slice) | 🟢 | Low | - |
| Testcontainers | 🟠 | P1 | - |
| Maven (базовое) | 🟡 | P1 | - |
| Maven (dependency mgmt) | 🟠 | P1 | Корпоративный контекст |
| JAXB / codegen | 🔴 | **P0** | Болевая точка |
| Spring Security (новый API) | 🟠 | P1 | - |
| Spring Boot 2→3 migration | 🟠 | P1 | - |
| GraalVM native | 🔴 | P2 | До модуля 11 |
| Observability | 🟡 | P2 | - |

## Рекомендованный порядок модулей
1. **Модуль G** (Generics)
2. **Модуль X** (Codegen + XSD)
3. **Модуль 2** (Spring Core / IoC)
4. **Модуль M** (Maven)
5. **Модуль 5** (Persistence с SQL observability)
6. **Модуль 4** (Web)
7. **Модуль 7** (Security)
8. **Модуль 9 + 10** (Observability + События)
9. **Модуль 11** (Cloud-native + GraalVM)

## Сквозные темы (не модули — пронизывают всё)
- **AOP/Proxy** — proxy-механика, self-invocation, аспекты (по вызову)
- **Testing** — привычка: unit → slice → integration в каждом модуле (по вызову)
- **SB3-специфика** — Virtual Threads, HTTP Interfaces, Problem Details, Observation API, новый Security API (по вызову)
- **Clean Code / Clean Architecture** ⚡ — слои, DIP, границы, naming, SRP (всегда на дежурстве)

Проработка через **Совет мудрецов** (code review + экзамен комиссией). Страж Clean Architecture активен постоянно — в парном программировании и при проблематизации кода.

## План на следующие 2 недели
- Модуль G, 4 сессии: 1/4, 2/4, 3/4, экзамен
- Первая мета-сессия после модуля: дата [...]
```

**Шаг 5. Первая мета-запись** (через Claude Code в `meta-sessions/ms-00-kickoff.md`): общее впечатление, неожиданности, настрой.

---

# Закрытие Модуля 0

Закрыт, когда:
- ✅ 5 сессий ассессмента пройдены
- ✅ `heat-map.md` построена
- ✅ Приоритизированный порядок модулей составлен
- ✅ Первая мета-запись в `meta-sessions/`
- ✅ 25 записей Evidence в ledger
- ✅ 6 записей в reflection-log (5 сессий + финал)

С этого момента — основной курс по вашей индивидуальной программе.
