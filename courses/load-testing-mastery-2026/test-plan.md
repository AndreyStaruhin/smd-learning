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

## 2.1 Подход к мониторингу: RED + USE + backlog dynamics

Чтобы не утонуть в метриках Micrometer, опираемся на два канонических паттерна и одну специфичную для нашей архитектуры пару величин.

### Паттерн RED (для сервисов)

[Tom Wilkie, Weaveworks] — измеряется **на уровне каждого вызываемого сервиса/слоя**:

| Буква | Что | Где у нас брать |
|---|---|---|
| **R**ate | RPS / задач в секунду | Gatling (HTTP), `rate(http_server_requests_seconds_count[1m])`, `rate(dp_queue_tasks_processed_total[1m])` |
| **E**rrors | Доля неуспешных | Gatling error rate, `rate(http_server_requests_seconds_count{status=~"5.."}[1m])`, `rate(dp_queue_tasks_failed_total[1m])` |
| **D**uration | p50/p95/p99 latency | Gatling histogram, `histogram_quantile(0.99, rate(http_server_requests_seconds_bucket[1m]))`, `dp_queue_task_processing_seconds` |

RED применяем **дважды**: к синхронному REST-слою (HTTP-обращения банка) и к асинхронному слою (воркеры очереди как «сервис» с собственными RPS/Errors/Duration).

### Паттерн USE (для ресурсов)

[Brendan Gregg] — для **каждого ограничивающего ресурса**:

| Буква | Что | Ресурсы и метрики |
|---|---|---|
| **U**tilization | % времени, ресурс занят | CPU: `process_cpu_usage`. Heap: `jvm_memory_used_bytes / jvm_memory_max_bytes`. Tomcat: `tomcat_threads_busy_threads / tomcat_threads_config_max`. Hikari: `hikaricp_connections_active / hikaricp_connections_max`. Воркеры очереди: `busyWorkers / threadPoolSize` (см. раздел 7). |
| **S**aturation | Глубина очереди ожидания | Tomcat: `tomcat_threads_busy_threads` после превышения `max`. Hikari: `hikaricp_connections_pending`. БД: `pg_stat_activity.wait_event_type='Lock'`. Очередь задач: backlog (см. ниже). |
| **E**rrors | Ошибки самого ресурса | GC: `jvm_gc_pause_seconds` сверх SLA; Hikari: `hikaricp_connections_timeout_total`; БД: `pg_stat_database.deadlocks`; воркеры: `retryTransientError` count. |

### Backlog dynamics (специфично для нашей очереди)

Поскольку обработка асинхронная и идёт через PostgreSQL-очередь, классических RED/USE недостаточно — нужно явно отслеживать **динамику накопления**:

```
incoming_rate  = rate(dp_queue_tasks_incoming_total[1m])
processing_rate = rate(dp_queue_tasks_processed_total[1m])
backlog_growth  = incoming_rate − processing_rate     ← Prometheus-выражение
```

Интерпретация:

| Знак | Что значит |
|---|---|
| `backlog_growth > 0` устойчиво | Воркеры не успевают, очередь растёт. **Сигнал предела пропускной способности**. |
| `backlog_growth ≈ 0` | Установился steady state на текущей нагрузке. |
| `backlog_growth < 0` | Очередь рассасывается (есть запас по worker-ам). |

Дополнительно держим **gauge** `dp_queue_tasks_backlog_size` (текущий размер) — для тревог при выходе за абсолютный порог (например, > 1000 задач при `threadPoolSize=10`).

> 💬 А ты знал, что Little's Law — `L = λ × W` (среднее число задач в системе = arrival rate × среднее время в системе) — даёт независимую оценку? Если измерили `processing_rate` и `processing_duration` (через Timer), то теоретически `backlog ≈ processing_rate × processing_duration`. Расхождение на порядки между формулой и фактом — признак, что задачи где-то «застряли».

### Привязка существующих метрик плана к паттернам

| Раздел документа | RED | USE | Backlog |
|---|:---:|:---:|:---:|
| §5 (Сценарий A, Gatling) | ✅ R+E+D на HTTP | — | — |
| §6 (Сценарий B, очередь) | ✅ R+E+D на воркерах | ✅ U+S на пуле воркеров и БД | ✅ через SQL-замеры |
| §7 (Prometheus/Grafana) | ✅ R+E+D на HTTP | ✅ U+S на JVM/Tomcat/Hikari | ⚠️ нужен PR с кастомными метриками |

**Вопросы к обсуждению:**
- Согласны ли на RED+USE как канонический набор для отчёта? Это тот язык, на котором SRE-команда смотрит сервисы — упростит передачу результатов после теста.
- Какие пороги ставим как «предел» по `backlog_growth`? Предлагаю: `> 0` устойчиво более 2 минут на ступени = достигли предела пропускной способности воркеров.

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

## 6.1 Сценарий C — комбинированная нагрузка (REST → очередь → воркеры)

**Зачем нужен этот сценарий:** A покрывает только REST-слой и не нагружает воркеры (запросы успевают пройти HTTP-фазу до того, как очередь успевает забиться). B изолированно меряет воркеров, но без HTTP-конкуренции за CPU/Hikari. Сценарий C — это **единственный сценарий, в котором асинхронная часть нагружается так же, как в проде**: HTTP-трафик от Gatling параллельно генерирует задачи, воркеры их разгребают, обе подсистемы делят один CPU и один Hikari-пул.

**Конфигурация:**

- Gatling гонит сценарий A (полный цикл REST), но **без `truncate` между прогонами** — задачи копятся.
- Воркеры включены (`queue.executor.threadPoolSize=10`, режим `person`).
- WireMock с задержкой 150 мс на адаптер и Gate-out (как в B).
- Все кастомные метрики (см. §7) включены и отображаются на одном дашборде.

**Профиль:**

| Этап | Gatling RPS на `/cb` | Длительность | Цель |
|---|---|---|---|
| Warm-up | 1 | 2 мин | Прогрев + создание стартового backlog'а ~50 задач |
| Steady low | 5 | 5 мин | Проверить, что `backlog_growth ≈ 0` — система держит. Опорные RED/USE. |
| Steady mid | 15 | 10 мин | Поиск точки, где `backlog_growth > 0` начинает расти устойчиво |
| Burst | rampUp 5 → 30 за 1 мин | 1 мин ramp + 5 мин держим | Тест на short-burst: способна ли очередь рассосать всплеск за разумное время? |
| Recovery | обратно к 5 | 5 мин | Время рассасывания backlog'а после burst — это и есть «time to recover» |

**Критерии остановки:**
- `backlog_growth > 0` устойчиво более 2 минут на ступени steady mid → нашли предел воркеров.
- `dp_queue_tasks_backlog_size > 1000` (10× threadPoolSize × «нормальный» latency) → уходим на ступень вниз.
- p99 HTTP `/cb` > SLA или error rate > 1% → ушли в HTTP-предел раньше воркеров (значит, узкое место — REST-слой, а не очередь).

**Что хотим узнать:**

1. **Какая подсистема упирается первой** при честной смешанной нагрузке: HTTP, воркеры, БД?
2. **Time to recover** после burst’а: сколько секунд воркерам нужно, чтобы вернуть backlog в 0 после кратковременного 6× по нагрузке.
3. **Конкуренция за Hikari**: при росте `hikaricp_connections_pending > 0` отстают и Tomcat-потоки, и воркеры — где это видно раньше.

**Вопросы к обсуждению:**
- Реалистичный профиль burst'а — какой? +1 ступень (+5 RPS) умеренно консервативен; в проде бывают «просыпания» больше.
- Нужно ли в этой же итерации делать вариант C+permissions-after-rest (когда включена настройка `needSendPermissionsRequestAfterRestScenario` и на каждый REST-цикл создаётся **две** задачи в очереди вместо одной)?

## 7. Метрики — что и откуда снимаем

### Gatling (сценарий A)
- Response time p50/p95/p99 отдельно для каждого из трёх запросов.
- Error rate (суммарный и по endpoint).
- Фактический throughput (req/s).

### Prometheus + Grafana (основной канал метрик)

Настройка один раз (см. раздел 3 — архитектура стенда). Дальше всё идёт автоматически:

- **Scrape**: Prometheus опрашивает `http://<vm>:5698/actuator/prometheus` каждые 10–15 сек и хранит timeseries за всё время прогона.
- **Что смотрим в Grafana** (готовый дашборд "JVM (Micrometer)" + панели по нашим RED/USE/backlog).

#### Out-of-the-box метрики (USE + RED для HTTP)

| Метрика | Что отражает | Паттерн |
|---|---|---|
| `jvm_memory_used_bytes{area="heap"}` | Heap | USE/Util |
| `rate(jvm_gc_pause_seconds_sum[1m])` | GC паузы | USE/Errors |
| `process_cpu_usage` | CPU | USE/Util |
| `tomcat_threads_busy_threads / tomcat_threads_config_max` | Tomcat threadpool | USE/Util |
| `tomcat_threads_busy_threads >= max` устойчиво | Очередь Tomcat-acceptor | USE/Saturation |
| `hikaricp_connections_active` / `_max` | БД-пул | USE/Util |
| `hikaricp_connections_pending` | Ожидание коннекта | USE/Saturation |
| `hikaricp_connections_timeout_total` | Не дождались | USE/Errors |
| `rate(http_server_requests_seconds_count[1m])` by `uri` | HTTP RPS | RED/Rate |
| `rate(http_server_requests_seconds_count{status=~"5.."}[1m])` | HTTP errors | RED/Errors |
| `histogram_quantile(0.99, rate(http_server_requests_seconds_bucket[1m]))` by `uri` | HTTP p99 | RED/Duration |

#### Кастомные метрики Micrometer (асинхронная часть — RED + backlog dynamics)

Эти метрики **нужно добавить в код отдельным PR до начала тестов**. Без них наблюдать за асинхронной частью можно только через ручные SQL-запросы.

| Метрика | Тип | Точка инструментирования | Назначение |
|---|---|---|---|
| `dp_queue_tasks_incoming_total{taskType, taskStage}` | `Counter` | `ScenarioEventService.handleStartScenarioEvent()` (строка 63) — после `publisher.publishEvent(new TaskReadyToCreateEvent(...))` | RED/Rate воркеров (входной поток) |
| `dp_queue_tasks_processed_total{taskType, taskStage, result}` | `Counter` (теги `result=complete\|fail\|transient_error`) | `ScenarioEventService.publishResultEvent()` (строка 142) — в `switch` по `stageStatus` | RED/Rate (выход) + RED/Errors |
| `dp_queue_task_processing_seconds{taskType, taskStage}` | `Timer` | Обернуть `scenario.executeStage(...)` в `ScenarioEventService.handleTaskReceivedEvent()` (строка 128) | RED/Duration воркеров |
| `dp_queue_tasks_backlog_size{taskType}` | `Gauge` | `@Scheduled(fixedDelay=10s)` в новом `QueueMetricsCollector`, через `TaskRepository.countByRunningFalse(taskType)` | USE/Saturation очереди |
| `dp_queue_tasks_stuck_total{taskType}` | `Gauge` | Тот же `QueueMetricsCollector`, SQL: `running=true AND status_updated < now()-1m` | USE/Errors очереди |
| `dp_queue_workers_busy{taskType}` | `Gauge` | Из `QueueThreadPoolExecutor.getActiveCount()` | USE/Util воркеров |

**Имена тегов** держим в нижнем snake/kebab без специфичных Java-енумных значений: тег `taskType=person|org`, `taskStage=send_request|get_response|...`, `result=complete|fail|transient_error`.

**Размер cardinality**: для `correlationId`/`bankId` теги **не делать** — это кардинальный взрыв в Prometheus. Bank-фильтрация нужна только для прода с alert'ами, а не для нагрузочного теста.

#### Ключевые Prometheus-выражения для дашборда

Положить в Grafana отдельный дашборд "DP Queue — RED & Backlog":

```promql
# Backlog dynamics — главный график теста
sum(rate(dp_queue_tasks_incoming_total[1m]))
  - sum(rate(dp_queue_tasks_processed_total[1m]))

# Backlog absolute size
sum(dp_queue_tasks_backlog_size) by (taskType)

# Worker utilization (USE/Util)
sum(dp_queue_workers_busy) by (taskType)
  / on(taskType) group_left()
  sum(dp_queue_workers_pool_size) by (taskType)

# Worker p99 (RED/Duration)
histogram_quantile(0.99, sum(rate(dp_queue_task_processing_seconds_bucket[1m])) by (le, taskType))

# Worker error rate (RED/Errors)
sum(rate(dp_queue_tasks_processed_total{result=~"fail|transient_error"}[1m])) by (taskType)
  / sum(rate(dp_queue_tasks_processed_total[1m])) by (taskType)
```

**Скелет PR с метриками** (ориентир для разработчика):

```java
@Component
@RequiredArgsConstructor
class QueueMetrics {
    private final MeterRegistry registry;

    void recordIncoming(TaskType type, TaskStage stage) {
        Counter.builder("dp.queue.tasks.incoming")
                .tag("taskType", type.name().toLowerCase())
                .tag("taskStage", stage.name().toLowerCase())
                .register(registry).increment();
    }
    // recordProcessed(...), Timer.record(...) — по аналогии
}
```

И инжект `QueueMetrics` в `ScenarioEventService` — вызовы `recordIncoming(...)` после публикации `TaskReadyToCreateEvent` и `recordProcessed(...)` в `publishResultEvent` ветках `switch`.

**После прогона**: performance-инженер делает скриншот или экспорт графиков, вкладывает в раздел «Результаты» этого документа.

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
- [ ] PR с кастомными метриками (`dp.queue.tasks.incoming`, `_processed`, `_processing_seconds`, `_backlog_size`, `_stuck`, `_workers_busy`) смерджен в ветку стенда. Места инструментирования см. §7. Без этого PR сценарий C неинформативен.
- [ ] Дашборд "DP Queue — RED & Backlog" импортирован в Grafana (JSON-экспорт положить в `docs/load-test/grafana/`).
- [ ] Подтверждено, что метрики `dp_queue_tasks_*` появляются на `/actuator/prometheus` после пары пробных запросов.

## 9. Алгоритм ramp-up (универсальный)

1. Baseline — 3–5 минут на минимальной нагрузке. Записали опорные цифры.
2. +1 ступень (+5 RPS для A; +батч для B). Держим 3 мин.
3. Проверили все пороги (p99, error rate, backlog, CPU, GC pause). Если все в норме — шаг 2.
4. Если хотя бы один порог пересечён — откатились на предыдущую ступень, держим 10 мин (deflake).
5. **Предел = последняя стабильная ступень.**
6. Фиксируем в этом же документе: предел, что сломалось первым, какие метрики поднялись.

## 10. Ожидаемые узкие места (гипотезы для проверки)

Перечень на дискуссию. Не утверждения, а кандидаты — тест их проверит. Каждая гипотеза привязана к метрике из §2.1, по которой мы её подтвердим/опровергнем.

| # | Гипотеза | Сигнальная метрика | Паттерн |
|---|---|---|---|
| 1 | Пул воркеров ФЛ упрётся при сценарии B/C: при `>10` параллельных задач очередь растёт быстрее, чем обрабатывается. `queue.executor.threadPoolSize=10` в `application.yaml`. | `backlog_growth > 0` устойчиво при `dp_queue_workers_busy / pool_size = 1.0` | USE/Saturation + Backlog |
| 2 | `SELECT FOR UPDATE SKIP LOCKED` дорожает на больших объёмах `tasks`. PostgreSQL-блокировки могут становиться дороже самой обработки. | `dp_queue_task_processing_seconds` p99 растёт **до** того, как утилизация воркеров упёрлась. Дополнительно — `pg_locks` count. | RED/Duration + USE/Saturation БД |
| 3 | Hikari pool — `maximum-pool-size=15`. Если воркер ждёт коннект, очередь и REST-слой делят пул. | `hikaricp_connections_pending > 0` устойчиво. Проверка: при росте C-нагрузки (REST + воркеры) pending растёт раньше, чем в B. | USE/Saturation |
| 4 | GC паузы под потоком JSON ser/deser (MapStruct + Jackson). | `rate(jvm_gc_pause_seconds_sum[1m]) > 0.1` (10% времени в GC) | USE/Errors |
| 5 | Gate connection pool — `gate.connection.httpMaxConnTotal=200`. Маловероятно на наших объёмах. | Косвенно — рост `dp_queue_task_processing_seconds` без роста CPU/heap. | RED/Duration |
| 6 | **Конкуренция HTTP ↔ воркеры за CPU** в сценарии C. В B и A по отдельности упор может не воспроизвестись. | Падение `process_cpu_usage` headroom при одновременном росте HTTP RPS и `dp_queue_workers_busy`. | USE/Util |
| 7 | **Time to recover** после burst'а в сценарии C превышает разумные минуты. Это уже не bottleneck, а характеристика устойчивости. | После завершения burst — время, за которое `dp_queue_tasks_backlog_size` возвращается к baseline. | Backlog |

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
- **`core/src/main/java/ru/modernsys/dp/scenarios/ScenarioEventService.java`** — точки инструментирования кастомных метрик асинхронной части:
  - `handleStartScenarioEvent()` (line 63) — `dp.queue.tasks.incoming` после публикации `TaskReadyToCreateEvent`.
  - `handleTaskReceivedEvent()` (line 95) — `dp.queue.task.processing.seconds` Timer вокруг `scenario.executeStage(...)`.
  - `publishResultEvent()` (line 142) — `dp.queue.tasks.processed` с тегом `result` в `switch` по `stageStatus` (`COMPLETE`, `FAIL`, `TRANSIENT_ERROR`).
