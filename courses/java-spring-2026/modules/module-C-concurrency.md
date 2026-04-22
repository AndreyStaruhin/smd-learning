# Модуль C: Concurrency & Memory Model
## Безопасная multithreading-разработка как дисциплина

## Онтологическая цель

После модуля учащийся **не может** написать race condition, не заметив этого. Видит код через happens-before: где гарантия есть, где нет, что её даёт. Различает три разных concurrency-парадигмы (threads + locks / Virtual Threads + structured / reactive) и осознанно выбирает.

## Предпосылки

- Пройдена сессия A7 в Модуле 0
- Желательно — пройден Модуль J (bytecode для понимания volatile/synchronized на уровне барьеров памяти)

## Структура (4 сессии + экзамен)

### Сессия 1 — Java Memory Model как ментальная модель

- JMM как формальная спецификация (JLS §17)
- Happens-before как частичный порядок: что именно он гарантирует
- Reordering: compile-time vs CPU-level (StoreLoad барьеры и т.п.)
- Semantics: volatile, synchronized, final, Thread.start/join, ThreadLocal
- Safe publication: 4 способа корректно опубликовать объект между потоками
- Практика: анализ конкретных коротких фрагментов кода на "что может произойти"

### Сессия 2 — java.util.concurrent как инструментарий

- AQS (AbstractQueuedSynchronizer): основа ReentrantLock, Semaphore, CountDownLatch
- `ConcurrentHashMap` internals: lock striping, computeIfAbsent atomicity
- `LongAdder` vs `AtomicLong`: когда contention убивает scalability
- Executor framework: `ThreadPoolExecutor`, work-stealing в `ForkJoinPool`
- `CompletableFuture` и composition
- Практика: дизайн thread-safe кэша с TTL — собрать из concurrent-примитивов без synchronized

### Сессия 3 — Virtual Threads и Structured Concurrency

- Project Loom: virtual threads как M:N scheduling
- Carrier threads, mount/unmount, pinning (synchronized, JNI, некоторые native блокировки)
- Что меняется в Spring Boot 3 при `spring.threads.virtual.enabled=true`
- Scoped Values (Java 25) как замена ThreadLocal
- StructuredTaskScope (preview в Java 25): cancellation propagation, fork/join для concurrent tasks
- Практика: переписать блокирующий endpoint с Executor на Virtual Threads, сравнить throughput и memory

### Сессия 4 — Три парадигмы concurrency: сравнение

- Threads + shared state + locks (классика)
- Virtual Threads + blocking code (Loom-стиль)
- Reactive streams + backpressure (Reactor/RxJava)
- Когда каждая уместна: характер нагрузки, объём параллелизма, команда
- `@Async`, `@Scheduled`, reactive transactions: как Spring обслуживает каждую парадигму
- Практика: один и тот же use case (агрегация из 5 микросервисов) — три имплементации, замеры

### Экзамен модуля

**Формат:** задача на code review. Дано 5 фрагментов: race condition, broken DCL, Virtual Thread pinning, неправильный `@Async`-контракт, memory leak через ThreadLocal. Учащийся находит, объясняет через JMM, предлагает фикс. Совет мудрецов атакует.

## Evidence критерии

- [C] Видит race conditions в коде рефлекторно, не после подсказки
- [C] Может обосновать выбор concurrency-парадигмы через характеристики нагрузки
- [T] Корректно использует `CompletableFuture` composition (включая exception handling)
- [T] Понимает, когда Virtual Threads дают выигрыш, когда нет (CPU-bound задачи)
