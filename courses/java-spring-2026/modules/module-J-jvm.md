# Модуль J: JVM Fundamentals
## Рихтеровский уровень Java-платформы

## Онтологическая цель

После модуля учащийся видит JVM не как "среду исполнения", а как **конкретную машину** с правилами загрузки классов, компиляции, управления памятью и оптимизации. Может диагностировать проблемы не из логов приложения, а через инструменты JVM (JFR, heap dumps, `-XX:+PrintCompilation`).

## Предпосылки

- Пройдена сессия A6 в Модуле 0 (heat-map по JVM построена)
- Цвета A6.1–A6.4 определяют глубину проработки каждой сессии в модуле

## Структура (4 сессии + экзамен)

### Сессия 1 — Classloading и executable JAR format

- Parent delegation model, ClassLoader иерархия (Bootstrap → Platform → System → Application)
- Spring Boot `LaunchedURLClassLoader`: как работает `BOOT-INF/classes` и `BOOT-INF/lib`
- `ClassNotFoundException` vs `NoClassDefFoundError` — онтологическая разница
- Class unloading: когда возможно, когда нет
- Практика: написать custom ClassLoader для hot-reload в тестовой среде

### Сессия 2 — Bytecode как артефакт

- `.class` format на уровне структур: constant pool, methods, access flags
- Инструкции JVM: stack-based execution, local variables, operand stack
- `invokevirtual` / `invokeinterface` / `invokestatic` / `invokespecial` / `invokedynamic` — когда какая
- Lambda и `LambdaMetafactory`: почему не анонимный класс
- Практика: прочитать bytecode Spring-bean методов через `javap`, найти proxy-вызовы

### Сессия 3 — Garbage Collection in depth

- Generational hypothesis: почему работает
- G1: regions, mixed collections, humongous objects, когда Full GC
- ZGC и Shenandoah: concurrent collectors, read barriers, sub-ms pauses
- Когда что выбирать: latency SLA, throughput targets, heap size
- JFR + JMC для анализа GC поведения
- Практика: настроить GC для `methodolog`, провести нагрузочный тест, сравнить G1 vs ZGC

### Сессия 4 — JIT и профилирование

- Interpreter → C1 (tier 1-3) → C2 (tier 4) → deoptimization
- Profile-guided optimization: что JIT "узнаёт" о коде
- Inlining, escape analysis, scalar replacement
- JMH как единственный корректный способ micro-benchmark
- Async-profiler, flame graphs: production-grade профилирование
- Практика: профилировать hot path в `methodolog`, найти неожиданную не-inlined функцию, понять причину

### Экзамен модуля

**Формат:** [Симулянт-эксперт] в роли Principal Engineer. Сценарий: production-инцидент с утечкой памяти или деградацией latency. Учащийся проводит диагностику через JVM-инструменты, формулирует гипотезу, подтверждает через данные, предлагает решение. Полный Совет мудрецов атакует после.

## Evidence критерии

- [C] Может прочитать GC лог и назвать: нормальное поведение / деградация / конкретный паттерн проблемы
- [C] Может объяснить код на bytecode-уровне, где это нужно для понимания поведения (особенно proxy, lambda, invokedynamic)
- [T] Выбирает GC по характеристикам нагрузки, не по привычке
- [T] Пишет корректный JMH benchmark для конкретного вопроса производительности
