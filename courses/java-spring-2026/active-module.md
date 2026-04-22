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

# Сессия A6 — JVM Fundamentals (2 часа)

Диагностика онтологии JVM как платформы. Не Spring, не фреймворки — сама виртуальная машина. Рихтеровский уровень: что реально происходит под абстракцией "Java кода".

## A6.1 [R] — Classloading и class identity (15 мин)

> В `assessment/A6/ClassLoaderDemo/` загрузи один и тот же class-файл двумя разными URLClassLoader (один parent-first, второй — child-first с явным exclude). Создай экземпляр каждого класса, попробуй `instanceof` и cast из одного в другой. Что получаешь? Почему `ClassCastException` при одинаковых fully qualified names?

**Критерии:** понимает, что class identity = `(ClassLoader, FQN)`, не только FQN? Знает parent delegation model? Может объяснить, почему Spring Boot executable JAR использует кастомный `LaunchedURLClassLoader`?

## A6.2 [T] — Bytecode inspection (20 мин)

> Простой класс `Calc` с методами `add(int, int)`, `addStrings(String, String) { return a + b; }`, `callPrint(List<String> items)`. Сделай `javac Calc.java && javap -c -p Calc`. Предскажи до запуска:
> 1. Как "a" + "b" компилируется в bytecode — один `ldc` или последовательность? (Java 9+ vs раньше)
> 2. Какая invoke* инструкция для `items.forEach(System.out::println)`?
> 3. Что такое `invokedynamic` и почему он нужен для lambda?
>
> Сверь с реальным выводом javap. Объясни расхождения.

**Критерии:** различает `invokevirtual` / `invokeinterface` / `invokestatic` / `invokespecial` / `invokedynamic`? Знает, что lambda — это не анонимный класс (был в ранних превью), а `LambdaMetafactory` через `invokedynamic`?

## A6.3 [C] — GC диагностика (25 мин)

> В `assessment/A6/LeakyApp/` — Spring Boot приложение с намеренной утечкой (статическая `Map<String, byte[]>`, в которую endpoint кладёт записи без удаления). Запусти с `-Xmx256m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=./dump.hprof -Xlog:gc*:gc.log`. Нагрузи через curl в цикле. Дождись OOM.
>
> Задачи:
> 1. Проанализируй `gc.log` — какие события видишь? Young GC, Mixed, Full GC?
> 2. Открой `dump.hprof` в Eclipse MAT (или VisualVM). Найди dominator tree, определи leak suspect.
> 3. Сравни поведение на G1 (default в J17+) vs ZGC (`-XX:+UseZGC`). В чём разница по latency и throughput?
> 4. Что такое "Stop-The-World" и почему в ZGC его практически нет?

**Критерии:** ключевой тест. Понимает поколения (young/old, в G1 — regions)? Знает, когда Full GC — это проблема vs норма? Различает throughput-oriented и latency-oriented сборщики?

## A6.4 [C] — JIT и micro-benchmark (20 мин)

> В `assessment/A6/JitDemo/` напиши "наивный" benchmark:
> ```java
> long t = System.nanoTime();
> for (int i = 0; i < 100_000_000; i++) { result = compute(i); }
> long dt = System.nanoTime() - t;
> ```
> Запусти 3 раза подряд в одном JVM. Результаты будут разными. Почему?
>
> Затем:
> 1. Напиши то же через JMH. Сравни результат.
> 2. Запусти с `-XX:+PrintCompilation -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining`. Что видишь?
> 3. Почему JVM может **полностью удалить** твой цикл, если result не используется? Что такое dead code elimination в JIT?

**Критерии:** понимает разницу interpreted → C1 → C2? Знает про warm-up, profile-guided compilation, inlining, escape analysis? Понимает, почему без JMH micro-benchmarks почти всегда врут?

## Оценка A6
```
- Classloading model: 🟢/🟡/🟠/🔴
- Bytecode literacy: 🟢/🟡/🟠/🔴
- GC onthology: 🟢/🟡/🟠/🔴
- JIT awareness: 🟢/🟡/🟠/🔴
- Profiling tools reflex: 🟢/🟡/🟠/🔴
```

---

# Сессия A7 — Concurrency & Memory Model (2 часа)

Диагностика: безопасная multithreading-разработка как дисциплина, не знание API. Без этого модуля senior — это senior по названию.

## A7.1 [R] — Race condition и "исправления" (15 мин)

> `assessment/A7/CounterRace.java`: два потока инкрементируют общий `int counter` миллион раз каждый. Ожидаем 2_000_000, получаем меньше. Почему? Исправь **тремя способами**:
> 1. `synchronized`
> 2. `AtomicInteger`
> 3. `volatile int`
>
> Запусти каждый. Который не работает и почему?

**Критерии:** ключевой тест. Понимает, что volatile даёт видимость, но НЕ атомарность compound-операций? Знает CAS в AtomicInteger? Понимает стоимость каждого варианта (contention, cache line bouncing)?

## A7.2 [T] — Happens-before и reordering (20 мин)

> `assessment/A7/HappensBefore.java`:
> ```java
> int x = 0, y = 0;
> boolean ready = false;
> 
> Thread writer: x = 42; y = 100; ready = true;
> Thread reader: if (ready) { print(x + y); }
> ```
> Что может увидеть reader? 142? 0? 42? 100? Все варианты? Почему?
>
> Затем:
> 1. Сделай `ready` volatile. Что изменилось по JMM?
> 2. Замени на `AtomicBoolean.set/get`. Что гарантируется?
> 3. Используй `synchronized` блоки. Какая разница?

**Критерии:** ядро теста. Понимает happens-before как частичный порядок? Знает, что volatile write → volatile read создаёт HB-edge? Понимает reordering (compile-time и runtime)?

## A7.3 [T] — Double-checked locking и initialization (15 мин)

> Классический broken DCL singleton:
> ```java
> private static Instance instance;
> public static Instance get() {
>   if (instance == null) {
>     synchronized(Class.class) {
>       if (instance == null) instance = new Instance();
>     }
>   }
>   return instance;
> }
> ```
> 1. Почему это broken до Java 5?
> 2. Что исправит `volatile` на поле? Почему это работает только с Java 5+?
> 3. Holder class idiom (статический вложенный класс) — почему предпочтительнее DCL?
> 4. Что происходит с record vs class при инициализации?

**Критерии:** понимает safe publication как концепт? Знает JMM изменения в Java 5 (new memory model JSR-133)? Понимает, почему holder pattern использует class initialization lock, который даёт HB бесплатно?

## A7.4 [C] — Virtual Threads и pinning (30 мин)

> В `assessment/A7/VirtualThreadsPinning/` — Spring Boot с `Tomcat` в Virtual Threads mode (SB3). Endpoint, который делает I/O в `synchronized(lock) { slowIO(); }`.
>
> Задачи:
> 1. Запусти с `-Djdk.tracePinnedThreads=full`. Нагрузи. Что видишь в логе?
> 2. Замени `synchronized` на `ReentrantLock`. Pin-нинг ушёл? Почему?
> 3. Объясни механику: что такое carrier thread, что такое mount/unmount, почему `synchronized` pin-ит, а `ReentrantLock` — нет.
> 4. В каких ещё случаях Virtual Thread pin-ится? (JNI, некоторые native блокировки)
> 5. Какое это имеет значение для твоего реального кода? Где в methodolog есть потенциальные pin-ы?

**Критерии:** понимает Virtual Threads как M:N scheduling с carrier threads? Различает preemption и cooperative unmount? Знает, что это — не про "быстрее", а про "дешевле держать thread заблокированным на I/O"?

## Оценка A7
```
- Race conditions recognition: 🟢/🟡/🟠/🔴
- JMM и happens-before: 🟢/🟡/🟠/🔴
- Safe publication: 🟢/🟡/🟠/🔴
- java.util.concurrent fluency: 🟢/🟡/🟠/🔴
- Virtual Threads onthology: 🟢/🟡/🟠/🔴
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
| JVM classloading | ? | ? | Зависит от A6.1 |
| Bytecode literacy | ? | ? | Зависит от A6.2 |
| GC tuning | ? | ? | Зависит от A6.3 |
| JIT awareness | ? | ? | Зависит от A6.4 |
| JMM / happens-before | ? | ? | Зависит от A7.2 |
| Safe publication | ? | ? | Зависит от A7.3 |
| Virtual Threads | ? | ? | Зависит от A7.4 |

## Рекомендованный порядок модулей
1. **Модуль G** (Generics)
2. **Модуль J** (JVM Fundamentals) — NEW
3. **Модуль C** (Concurrency & Memory Model) — NEW
4. **Модуль X** (Codegen + XSD)
5. **Модуль 2** (Spring Core / IoC)
6. **Модуль M** (Maven)
7. **Модуль 5** (Persistence с SQL observability)
8. **Модуль 4** (Web — с DispatcherServlet internals)
9. **Модуль 6** (AOP — с CGLIB/JDK proxy механикой)
10. **Модуль 7** (Security — с SecurityFilterChain и ThreadLocal SecurityContext)
11. **Модуль 8** (Testing — с ContextCache и @Transactional rollback механикой)
12. **Модуль 9 + 10** (Observability + События — с Micrometer Observation API)
13. **Модуль 11** (Cloud-native + GraalVM)
14. **Финальная сессия** (Reactive/WebFlux — парадигмальный взгляд сбоку)

## Сквозные темы (не модули — пронизывают всё)
- **AOP/Proxy** — proxy-механика, self-invocation, аспекты (по вызову)
- **Testing** — привычка: unit → slice → integration в каждом модуле (по вызову)
- **SB3-специфика** — Virtual Threads, HTTP Interfaces, Problem Details, Observation API, новый Security API (по вызову)
- **Clean Code / Clean Architecture** ⚡ — слои, DIP, границы, naming, SRP (всегда на дежурстве)

Проработка через **Совет мудрецов** (code review + экзамен комиссией). Страж Clean Architecture активен постоянно — в парном программировании и при проблематизации кода.

## План на следующие 2 недели
- Модуль G, 4 сессии: 1/4, 2/4, 3/4, экзамен
- Первая мета-сессия после модуля: дата [...]
- Далее: Модуль J (JVM) → Модуль C (Concurrency) — основано на heat-map по A6 и A7
```

**Шаг 5. Первая мета-запись** (через Claude Code в `meta-sessions/ms-00-kickoff.md`): общее впечатление, неожиданности, настрой.

---

# Закрытие Модуля 0

Закрыт, когда:
- ✅ 7 сессий ассессмента пройдены (A1–A5 + A6 + A7)
- ✅ `heat-map.md` построена с покрытием JVM и concurrency зон
- ✅ Приоритизированный порядок модулей составлен (с учётом новых MJ и MC)
- ✅ Первая мета-запись в `meta-sessions/`
- ✅ 34 записи Evidence в ledger (25 из A1–A5 + 9 из A6+A7)
- ✅ 8 записей в reflection-log (7 сессий + финал)

С этого момента — основной курс по вашей индивидуальной программе.
