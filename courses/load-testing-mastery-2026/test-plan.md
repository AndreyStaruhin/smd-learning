# План нагрузочного тестирования сервиса digitalprofile.individuals

> Документ для обсуждения на встрече с командой (разработка + QA).
> Цель встречи — согласовать scope, ответить на отмеченные вопросы и по итогу зафиксировать план перед тем, как писать Gatling-скрипты.

## 1. Контекст и мотивация

После серии задач последнего релизного цикла (в частности DO-11665 — проброс `customerUuid` и валидация) у нас появилась новая колонка в `esia_operations`, новая проверка в `CbEsiaService.validateCustomerUuid`, новое поле в Gate-запросе и ещё несколько мёрджей из master. Параллельно вносились структурные изменения в обработку очередей (разделение `queue-person-executor` / `queue-org-executor`). На этом фоне важно получить опорные метрики производительности одной ноды — без них невозможно ни принимать решения о масштабировании, ни отличать регресс от нормы.

Ключевая архитектурная особенность, из-за которой стандартные HTTP-нагрузочные тесты недостаточны: сервис работает **асинхронно через PostgreSQL-очередь** (таблица `tasks`, выборка через `SELECT FOR UPDATE SKIP LOCKED`). На REST-запрос клиент получает ответ **до** того, как реальная работа сделана — задача уходит в очередь и обрабатывается воркерами (`PersonQueueTaskHandler`, `OrgQueueTaskHandler`). Поэтому только один Gatling-сценарий не покрывает нагрузку целиком.

## 2. Цель теста (capacity)

Найти предел пропускной способности одной инстанс-ноды по трём поверхностям:

1. **REST-слой** — максимум RPS на `GET /rest/api/v1/cb`, при котором error rate остаётся < 1% и p99 < согласованного SLA.
2. **Очередь ФЛ** — максимум задач в секунду, которые воркеры успевают обрабатывать без роста backlog (сейчас `queue.executor.threadPoolSize=10`).
3. **БД** — не становится ли PostgreSQL первым узким местом на `getNextPerson` или на INSERT в `tasks`/`history`.

**Не входит в scope** этой итерации: мульти-инстансная конфигурация, soak-тест 24+ часа, нагрузочный тест реверс-СМЭВ, полный end-to-end с реальным логином пользователя в ЕСИА.

**Вопросы к обсуждению:**
- Какой наш целевой SLA по p95/p99 на `GET /cb`? Есть ли требование бизнеса/ЦБ?
- Допустимо ли в этой итерации не проверять ЮЛ-очередь (`OrgQueueTaskHandler`) — или нужно покрыть тестом оба типа задач?

## 3. Инструменты и архитектура стенда

### Распределение по машинам

Принцип: **нагружающий и нагружаемый на разных хостах** — иначе Gatling и сервис конкурируют за CPU/RAM и результаты искажаются.

| Хост | Что крутится | Зачем |
|---|---|---|
| **Локальная VM** (Linux, `docker-compose`) | сервис `digitalprofile.individuals`, PostgreSQL, WireMock, Prometheus, Grafana | Изолированная среда: можно снимать снапшоты, пересоздавать с нуля, не трогая прод-стенды. Prometheus/Grafana рядом с сервисом — короткий путь для метрик, низкая нагрузка на сеть |
| **Тестовая машина** | Gatling (разработка сценариев и прогоны), браузер для Grafana | Нагружающий — отдельно, ест CPU при больших RPS. Здесь же пишем скрипты, храним историю прогонов |
| **Внешние сервисы** | реальный ТР ЕСИА | Как в сценарии A |

Схема взаимодействия:

```
[Test machine]                   [Local VM]                         [Internet]
  Gatling  ──HTTP──────────▶  nginx/tomcat (dp-service)
                                    │                                  ТР ЕСИА
                                    ├──SOAP─▶ WireMock (СМЭВ adapter)
                                    ├──HTTP─▶ WireMock (Gate-out)
                                    └──JDBC─▶ PostgreSQL
                                    
                              Prometheus ──scrape──▶ /actuator/prometheus
                              Grafana    ──query───▶ Prometheus
  Браузер  ──HTTP──────────▶ Grafana UI  (графики в реальном времени)
```

### Стек инструментов

- **Gatling** (тестовая машина) — синхронный сценарий A. Hosted на отдельной машине, чтобы не смешивать нагрузку с сервисом.
- **WireMock** (на VM рядом с сервисом) — моки для адаптера СМЭВ (`SmevOptions.AdapterUri`) и Gate-out (`Gates.AbsGateUri`). Отвечают фиксированно + задержка ~150 мс.
- **PostgreSQL** (на VM) — отдельная чистая БД, можно TRUNCATE перед каждым прогоном.
- **Spring Actuator + Micrometer Prometheus registry** (в самом сервисе) — экспортит метрики на `/actuator/prometheus`. Нужно добавить зависимость `micrometer-registry-prometheus` в `core/pom.xml` (рядом с уже подключённым `spring-boot-starter-actuator`).
- **Prometheus** (на VM, Docker) — скрепит `/actuator/prometheus` каждые 10–15 сек, хранит timeseries за прогон.
- **Grafana** (на VM, Docker) — визуализация. Используем готовый дашборд "JVM (Micrometer)" ID 4701 или JVM Overview.

### Что нужно внести в код сервиса (отдельным PR, до старта тестов)

1. В `core/pom.xml` добавить:
   ```xml
   <dependency>
       <groupId>io.micrometer</groupId>
       <artifactId>micrometer-registry-prometheus</artifactId>
   </dependency>
   ```
2. В `application.yaml` стенда (на VM, не в master) — открыть эндпоинты:
   ```yaml
   management:
     endpoints:
       web:
         exposure:
           include: health,info,metrics,prometheus
     metrics:
       tags:
         application: digitalprofile-individuals
   ```
   После этого появится `GET /actuator/prometheus` с метриками в формате Prometheus.

### Кто и что снимает с Actuator

Actuator — просто HTTP endpoint, сам в файл ничего не пишет. Разделение ролей:

| Роль | Задача | Когда |
|---|---|---|
| DevOps / автор стенда | Поднять Prometheus+Grafana в Docker на VM; добавить Prometheus scrape-job на `http://<vm>:5698/actuator/prometheus`; импортировать базовые дашборды | Один раз до начала тестов |
| Разработчик сервиса | PR с `micrometer-registry-prometheus` в pom и открытыми actuator endpoints | Один раз до начала тестов |
| Performance-инженер (тот, кто гоняет тест) | Во время прогона смотрит Grafana в реальном времени; после прогона экспортирует графики / PDF / CSV; пишет вывод в раздел «Результаты» этого документа | Каждый прогон |

**Что вообще снимается** (готовые метрики Micrometer — ничего писать не надо):
- `jvm_memory_used_bytes{area="heap"}` — heap.
- `jvm_gc_pause_seconds_sum/count` — GC паузы.
- `process_cpu_usage` — CPU.
- `tomcat_threads_busy_threads`, `tomcat_threads_current_threads` — пул Tomcat.
- `hikaricp_connections_active`, `hikaricp_connections_pending` — БД-коннекшены.
- `http_server_requests_seconds_{count,sum,max}` с тегами `uri`, `status` — per-endpoint latency и RPS.

**Вопросы к обсуждению:**
- Кто владеет WireMock-конфигом? Есть ли уже готовые stub’ы для адаптера/Gate в проекте или делаем с нуля?
- Кто собирает VM: разработчик сам, DevOps или QA? Предлагаю один docker-compose файл с сервисом + PG + WireMock + Prom + Grafana, положить в `docs/load-test/` рядом с этим планом.
- Сеть между тестовой машиной и VM — это тот же сегмент LAN? Latency до VM не должен мешать замерам (10 мс в LAN — норма, 50 мс через VPN — уже искажает low-level latency).

## 4. Изоляция от внешних систем

**Согласованный вариант**: реальный ТР ЕСИА для сценария A + WireMock для адаптера СМЭВ и Gate-out.

**Обоснование:** реальный ТР ЕСИА даёт честный сигнал по стадии инициации (`requestPreliminaryAccess`), а моки СМЭВ/Gate делают результаты повторяемыми. Иначе шумы от чужих сервисов заглушили бы собственные ограничения.

**Риски:**
- ТР ЕСИА может rate-limit’ить тестовый стенд. Потолок может быть виден на ТР ЕСИА, а не у нас.
- `TrEsiaOptions.SessionTtlMinutes=30` — если шаги одной сессии Gatling растягиваются дольше, операция в нашей БД expire’ит, `handleSuccessRedirect` отредиректит на failure.

**Вопросы к обсуждению:**
- Нужно запросить у владельцев ТР ЕСИА разрешённый RPS на стенде. Кто это сделает?
- Готовы ли мы к тому, что первые запуски упрутся в ТР ЕСИА, а не в наш сервис? Возможно, имеет смысл параллельно делать прогон со stub’ом ТР ЕСИА для сравнения.

## 5. Сценарий A — синхронный REST-слой (Gatling)

Одна виртуальная сессия Gatling имитирует полный цикл синхронного взаимодействия:

1. **`GET /rest/api/v1/cb?purpose=CREDIT_REPORT&action=ALL_ACTIONS_TO_DATA&bankId=<uuid>&customerUuid=<v4-uuid>`**
   Ожидаем 302 Location на ТР ЕСИА. Из Location-URL извлекаем параметр (`sid` или `oprId`) — это идентификатор нашей операции в БД (см. `CbEsiaService.initiateEsiaAccess`, который сохраняет `EsiaOperationEntity` с `oprId`, возвращённым из ТР ЕСИА).
2. **Имитация callback от ТР ЕСИА**: `POST /rest/api/v1/cb/RegisterEsiaClientParams` с body
   ```json
   { "oprId": "<oprId>", "cpInfo": { "oid": "1002505368", "accessTokenHash": null } }
   ```
   Ожидаем 200 OK. Это обновляет `EsiaOperationEntity` в БД (`callbackReceived=true`).
3. **`GET /rest/api/v1/cb/success?oprId=<oprId>`**
   Это вызывает `handleSuccessRedirect` → отправка SMEV-запроса в адаптер (WireMock) → **создаётся задача в очереди `tasks`**. Ожидаем 302 на `Redirects.SuccessUri`.

**Почему callback имитируется:** настоящий callback приходит из ТР ЕСИА после того, как пользователь ввёл логин/пароль на страничке ЕСИА. Автоматизация этого — только через Selenium, что сильно усложняет тест и делает его нестабильным. Мы сознательно упрощаем: callback в сценарии A подделан, но это не меняет нагрузку на наш код, т.к. `/RegisterEsiaClientParams` обрабатывается одинаково независимо от источника.

**Нагрузочный профиль Gatling:**

| Этап | Users / RPS | Длительность | Цель |
|---|---|---|---|
| Warm-up | 1 → 5 | 2 мин | Прогрев JIT, Hikari-пула, кеша MapStruct |
| Baseline | 5 | 5 мин | Опорные p50/p95/p99 |
| Ramp-up | 5 → 50 ступенями по +5 RPS | каждая ступень 3 мин | Поиск точки деградации |
| Plateau на найденном пределе | X | 10 мин | Удостовериться, что деградация стабильна |

**Критерий остановки ramp-up**: p99 GET `/cb` > 3 сек **ИЛИ** error rate > 1% на двух соседних ступенях.

**Вопросы к обсуждению:**
- SLA p99 = 3 сек — достаточно ли строго? (Сравнить с .NET-версией, если известны её цифры.)
- Имитация callback: достаточно, или QA настаивает на Selenium для реалистичности?
- Структура `EsiaCpDataResult` в теле callback — достаточно ли минимального `oid`, или нужны все поля, которые приходят от реального ТР ЕСИА?

## 6. Сценарий B — изолированная пропускная способность очереди

Цель: чистая скорость воркеров ФЛ, без HTTP-шума.

**Подготовка:**

1. `TRUNCATE tasks, history, esia_operations RESTART IDENTITY`.
2. SQL-скрипт (или Liquibase seed) вставляет N готовых задач в `tasks` со стадией `GetResponse`. Шаблон `jsonData` — реальный JSON `GetResponseStageData`, снятый из лога прод/тест-прогона; размножаем с разными `bankId`/`correlationId`/`originalMessageId`.
3. Останавливаем сервис, запускаем с `queue.executor.mode=person` — чтобы ЮЛ-пул не конкурировал.
4. Адаптер СМЭВ и Gate-out — WireMock с задержкой 150 мс.

**Что замеряем (повторяем SELECT’ы раз в секунду):**

```sql
-- Throughput: сколько завершилось с начала прогона
SELECT count(*) FROM history WHERE updated > :start_ts AND status = 'AckPerformed';

-- End-to-end time на задачу (среднее)
SELECT avg(extract(epoch from (updated - created)))
FROM history WHERE status = 'AckPerformed' AND updated > :start_ts;

-- Backlog
SELECT count(*) FROM tasks WHERE running = false;

-- Stuck (зависшие)
SELECT count(*) FROM tasks
WHERE running = true AND status_updated < now() - interval '1 minute';
```

**Батчи:**

| N задач | Цель |
|---|---|
| 100 | Sanity-check. Ожидаем throughput ≈ (threadPoolSize / latency_одной_задачи) = 10 / 0.2 сек ≈ 50 задач/сек |
| 1 000 | Baseline, БД под full-load воркеров |
| 10 000 | Поиск точки, где `getNextPerson` начинает деградировать |
| 50 000 | Стресс: память, размер таблиц, VACUUM |

**Критерий деградации:** время на задачу растёт быстрее линейного; backlog не уменьшается; появляются retry (`retryTransientError` в `history`).

**Вопросы к обсуждению:**
- Достаточно ли покрыть только стадию `GetResponse`, или шаблонов должно быть несколько (`SendRequest`, `AckResponse`, плюс `CustomerPermissionListRequest`)?
- Делаем ли аналогичный сценарий B’ для ЮЛ (`OrgQueueTaskHandler`) в этой же итерации?
- На каких объёмах БД (`tasks`+`history`) обычно живёт прод? Если 50 000 — уже нереалистично много, сокращаем до реалистичного.

## 7. Метрики — что и откуда снимаем

### Gatling (сценарий A)
- Response time p50/p95/p99 отдельно для каждого из трёх запросов.
- Error rate (суммарный и по endpoint).
- Фактический throughput (req/s).

### Prometheus + Grafana (основной канал метрик)

Настройка один раз (см. раздел 3 — архитектура стенда). Дальше всё идёт автоматически:

- **Scrape**: Prometheus опрашивает `http://<vm>:5698/actuator/prometheus` каждые 10–15 сек и хранит timeseries за всё время прогона.
- **Что смотрим в Grafana** (готовый дашборд "JVM (Micrometer)" + пара панелей на специфичные метрики):
  - Heap: `jvm_memory_used_bytes{area="heap"}`.
  - GC: `rate(jvm_gc_pause_seconds_sum[1m])` — сколько времени JVM провела в GC.
  - CPU: `process_cpu_usage`.
  - Tomcat: `tomcat_threads_busy_threads / tomcat_threads_current_threads`.
  - Hikari: `hikaricp_connections_active`, `hikaricp_connections_pending` — если pending > 0 устойчиво, БД-коннекшены закончились.
  - Per-endpoint HTTP latency: `histogram_quantile(0.99, rate(http_server_requests_seconds_bucket[1m]))` с группировкой по `uri`.
- **После прогона**: performance-инженер делает скриншот или экспорт графиков, вкладывает в раздел «Результаты» этого документа.

Никаких ручных `curl`/CSV/скриптов — Micrometer сам публикует все нужные метрики.

### PostgreSQL
- `pg_stat_activity` — число активных соединений ≤ `hikari.maximum-pool-size=15`.
- `pg_stat_statements` (если расширение установлено на стенде) — какие SQL медленные. Первые подозреваемые: `NATIVE_QUERY_GET_NEXT_PERSON`, `NATIVE_QUERY_CREATE_TASK`, `NATIVE_QUERY_COMPLETE`.
- `pg_locks` — нет ли подвисших блокировок от SKIP LOCKED.
- Размер `tasks`/`history` до и после (pg_total_relation_size).

### OS
- CPU, RSS JVM, load average — `top` / `htop`.

### Логи сервиса
- `grep -c ERROR` и `grep -c 'retryTransientError'` в `logs/digitalprofile.log` — индикаторы перегрузки.

**Вопросы к обсуждению:**
- Кто будет снимать SQL-метрики во время теста — скрипт или вручную? Можно ли автоматизировать простым shell-лупом.
- Стоит ли добавить в логирование тайминги этапов обработки задачи (например, время между стадиями в `ScenarioEventService`) — чтобы после теста точно знать, где ушло время?

## 8. Подготовка окружения (чеклист)

### Локальная VM (стенд)
- [ ] `docker-compose.yml` с контейнерами: сервис digitalprofile, PostgreSQL, WireMock, Prometheus, Grafana. Положить рядом с этим планом в `docs/load-test/`.
- [ ] Prometheus scrape-job: `http://dp-service:5698/actuator/prometheus`, интервал 15 сек.
- [ ] Grafana provisioned datasource (Prometheus) + дашборд "JVM (Micrometer)" (ID 4701) + собственная панель с `http_server_requests_seconds` и Hikari.
- [ ] Отдельная БД; скрипт `seed_tasks.sql` для сценария B и `reset.sql` (TRUNCATE) между прогонами.
- [ ] WireMock stub'ы для `SendRequestRequest`/`GetResponseRequest` СМЭВ-адаптера и Gate-out endpoint'а, с задержкой ~150 мс.
- [ ] `application.yaml` на VM: `SmevOptions.AdapterUri` и `Gates.AbsGateUri` → URL WireMock; `management.endpoints.web.exposure.include=health,info,metrics,prometheus`.
- [ ] В `core/pom.xml` добавлен `micrometer-registry-prometheus` (отдельный PR, до теста).

### Тестовая машина
- [ ] Gatling установлен, базовый проект со сценариями A и B склонирован.
- [ ] Тестовая машина и VM в одной LAN (latency < 5 мс). Зафиксировать `ping`-результат в документе как поправку к p99.
- [ ] Таймауты Gatling подкручены так, чтобы не уходить в повторы по TCP.

### Общее
- [ ] У ТР ЕСИА запрошен разрешённый RPS на стенде; зафиксирован в этом документе.
- [ ] Открытый доступ к Grafana с тестовой машины (проверяется до начала теста).

## 9. Алгоритм ramp-up (универсальный)

1. Baseline — 3–5 минут на минимальной нагрузке. Записали опорные цифры.
2. +1 ступень (+5 RPS для A; +батч для B). Держим 3 мин.
3. Проверили все пороги (p99, error rate, backlog, CPU, GC pause). Если все в норме — шаг 2.
4. Если хотя бы один порог пересечён — откатились на предыдущую ступень, держим 10 мин (deflake).
5. **Предел = последняя стабильная ступень.**
6. Фиксируем в этом же документе: предел, что сломалось первым, какие метрики поднялись.

## 10. Ожидаемые узкие места (гипотезы для проверки)

Перечень на дискуссию. Не утверждения, а кандидаты — тест их проверит.

1. **Пул воркеров ФЛ** — `queue.executor.threadPoolSize=10` в `application.yaml`. При сценарии B >> 10 одновременных задач очередь упирается в число потоков × длительность одной задачи.
2. **`SELECT FOR UPDATE SKIP LOCKED`** на больших объёмах `tasks` — PostgreSQL-блокировки могут становиться дороже, чем сама обработка.
3. **Hikari pool** — `maximum-pool-size=15`. Если воркер ждёт коннект, растёт `hikaricp.connections.pending`.
4. **GC paуы** — под большим потоком JSON-ser/deser (MapStruct + Jackson) возможны видимые паузы. Мониторим `jvm.gc.pause`.
5. **Gate connection pool** — `gate.connection.httpMaxConnTotal=200`. Маловероятно bottleneck на наших объёмах, но проверим.

## 11. Риски и неясности (единым списком)

- ТР ЕСИА может rate-limit-нуть тестовый стенд и исказить результаты сценария A.
- `SessionTtlMinutes=30` — если шаги одной Gatling-сессии дольше 30 мин, операция expire, шаг 3 пойдёт на failure URI. Держим одну сессию ≤ 5 мин.
- JIT warm-up: первые 30–60 сек каждого прогона — прогрев, метрики оттуда некорректны. Всегда имеем warm-up-фазу.
- Накопление строк в `tasks`/`history` и влияние VACUUM: после 50k задач могут пойти autovacuum-паузы, которые будут читаться как деградация. Смотрим на метрики параллельно с `pg_stat_all_tables`.

## 12. Что делаем после прогона

1. В этот же документ добавляем раздел «Результаты <дата>» с цифрами и тем, что сломалось первым.
2. Заводим JIRA-тикет на устранение первого bottleneck (по гипотезам — это пул воркеров).
3. После внесения изменений — повторный прогон по тем же скриптам, сравнительный отчёт.

## Приложение A. Критичные файлы проекта

- `core/pom.xml:44` — `spring-boot-starter-actuator` уже подключён.
- `core/src/main/resources/application.yaml` — настройки `queue.executor`, `smev.pulling.executor`, `settings.center-mode`, Hikari.
- `core/src/main/java/ru/modernsys/dp/configuration/queue/QueueThreadPoolExecutorProperties.java` — `threadPoolSize`, `mode` (person / org / all).
- `core/src/main/java/ru/modernsys/dp/logic/queue/PersonQueueTaskHandler.java`, `OrgQueueTaskHandler.java` — воркеры.
- `queue/src/main/java/ru/modernsys/dp/queue/repository/TaskRepository.java` — SQL-запросы очереди (`NATIVE_QUERY_GET_NEXT_PERSON`, `NATIVE_QUERY_GET_NEXT_ORG`, `NATIVE_QUERY_CREATE_TASK`, `NATIVE_QUERY_COMPLETE`, `NATIVE_QUERY_RETRY_TRANSIENT_ERROR`).
- `core/src/main/java/ru/modernsys/dp/api/cb/CbEsiaController.java` — REST endpoints для сценария A.
- `core/src/main/java/ru/modernsys/dp/logic/tresia/CbEsiaService.java` — сохранение `EsiaOperationEntity`, создание задачи в `handleSuccessRedirect`, валидация customerUuid.
