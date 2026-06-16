+++
title = "Prometheus + Grafana + Alertmanager"
date = 1993-02-17
draft = false
tags = ["prometheus", "grafana", "alertmanager", "monitoring", "devops", "observability"]
categories = ["monitoring"]
description = "Повний розбір стеку Prometheus/Grafana/Alertmanager: як влаштована база даних часових рядів, типи метрик (Counter, Gauge, Histogram, Summary), PromQL функції rate/increase/irate, побудова дашбордів і алертів — з реальними прикладами кожного рядка."
+++

---

## 1. Архітектура стеку: хто що робить

```
[ Targets ]  ←── scrape ──  [ Prometheus ]  ──→  [ Grafana ]
                                   │
                              [ Alertmanager ]  ──→  Slack / PagerDuty / Email
```

| Компонент | Роль | Протокол |
|---|---|---|
| **Prometheus** | Збір, зберігання, обчислення | HTTP pull (scrape) |
| **Grafana** | Візуалізація, дашборди | HTTP API до Prometheus |
| **Alertmanager** | Роутинг, дедублікація, мовчання алертів | HTTP від Prometheus |
| **Exporter** | Конвертує метрики у Prometheus-формат | HTTP `/metrics` endpoint |

**Ключовий принцип — pull model**: Prometheus сам ходить до таргетів кожні N секунд (за замовчуванням 15 секунд). Це протилежність push-моделі (StatsD, InfluxDB Telegraf). Pull дає:
- централізований контроль над конфігурацією scrape
- легкий health-check (якщо таргет недоступний — Prometheus це знає)
- простіше troubleshoot мережевих проблем

---

## 2. Що таке метрика в Prometheus: ключ-значення зсередини

### 2.1 Формат exposition (текстовий)

Кожен `/metrics` endpoint повертає plain text:

```
# HELP http_requests_total Total HTTP requests received
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200",handler="/api/users"} 1847
http_requests_total{method="POST",status="201",handler="/api/users"} 312
http_requests_total{method="GET",status="404",handler="/api/unknown"} 23
```

Розбір по рядках:

```
# HELP http_requests_total Total HTTP requests received
^      ^                   ^
│      │                   └── Опис метрики (опціонально, але обов'язково для хорошого коду)
│      └── Ім'я метрики
└── Директива HELP

# TYPE http_requests_total counter
^      ^                   ^
│      │                   └── Тип метрики: counter | gauge | histogram | summary | untyped
│      └── Ім'я метрики
└── Директива TYPE

http_requests_total{method="GET",status="200",handler="/api/users"} 1847
^                   ^                                                ^
│                   │                                                └── Поточне значення (float64)
│                   └── Labels: ключ-значення пари — саме вони утворюють унікальний time series
└── Metric name
```

### 2.2 Як Prometheus зберігає дані: TSDB

Prometheus використовує власну **Time Series Database (TSDB)**, оптимізовану для append-only запису.

**Ідентифікація time series** — це пара:
```
metric_name + sorted set of labels = унікальний series
```

Приклад: наступні три рядки — це **три різних time series**:
```
http_requests_total{method="GET",  status="200"} 1847
http_requests_total{method="POST", status="200"} 312
http_requests_total{method="GET",  status="404"} 23
```

**Що зберігається на диску:**

Prometheus зберігає дані у блоках (blocks), кожен блок охоплює 2 години:
```
/prometheus/data/
├── 01GK4XY.../          ← block (2h chunk)
│   ├── chunks/          ← стиснуті дані (XOR delta encoding)
│   ├── index            ← inverted index label→series
│   └── meta.json
├── 01GK6AB.../
└── wal/                 ← Write-Ahead Log (останні ~2 години в пам'яті)
```

**WAL (Write-Ahead Log)**: свіжі дані спочатку пишуться в WAL (пам'ять + диск), потім кожні 2 години компактуються у block. Це захищає від втрати даних при падінні.

**Cardinality** — головна причина OOM у Prometheus:
```
# Кількість унікальних time series = кросс-добуток унікальних label values
# Наприклад:
# method: GET, POST, PUT, DELETE = 4 значення
# status: 200, 201, 404, 500 = 4 значення  
# handler: 50 різних ендпоінтів
# Разом: 4 × 4 × 50 = 800 time series для одної метрики
```

**Погана практика** — динамічні labels з високою кардинальністю:
```
# НІКОЛИ так не робіть:
http_requests_total{user_id="12345"} 1   # user_id може мати мільйони значень!
http_requests_total{request_id="abc-123-xyz"} 1  # унікальний на кожен запит — катастрофа
```

---

## 3. Типи метрик: Counter, Gauge, Histogram, Summary

### 3.1 Counter — лічильник, що тільки зростає

**Визначення**: монотонно зростаюче значення. Скидається тільки при рестарті процесу.

**Конвенція**: суфікс `_total` у назві.

```
# HELP http_requests_total Total number of HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 182947
http_requests_total{method="GET",status="500"} 47
```

**Що відбувається при рестарті**: значення скидається до 0. Саме тому **ніколи не використовуйте сирий counter у графіках** — ви побачите вертикальне падіння при кожному деплої.

**Правильне використання** — завжди через `rate()` або `increase()`:

```promql
# Кількість запитів на секунду (за останні 5 хвилин)
rate(http_requests_total[5m])

# Кількість запитів за останні 5 хвилин (абсолютна кількість)
increase(http_requests_total[5m])
```

**Де застосовується**:
- кількість HTTP запитів
- кількість помилок
- кількість оброблених завдань
- байти отримані/надіслані (node_network_receive_bytes_total)

---

### 3.2 Gauge — поточне значення

**Визначення**: значення, яке може зростати і зменшуватись довільно. Відображає поточний стан.

```
# HELP node_memory_MemAvailable_bytes Available memory in bytes
# TYPE node_memory_MemAvailable_bytes gauge
node_memory_MemAvailable_bytes 2.147e+09

# HELP go_goroutines Number of goroutines
# TYPE go_goroutines gauge
go_goroutines 42

# HELP node_load1 1-minute load average
# TYPE node_load1 gauge
node_load1 0.85
```

**Ключова різниця від Counter**: gauge **можна використовувати напряму** без `rate()`.

```promql
# Пряме використання — правильно для gauge:
node_memory_MemAvailable_bytes
node_load1
go_goroutines

# Обчислення % використання пам'яті:
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100

# Тренд gauge за час (похідна):
deriv(node_load1[10m])   # чи зростає навантаження?
predict_linear(node_disk_free_bytes[1h], 4*3600)  # прогноз місця на диску через 4 години
```

**Де застосовується**:
- використання CPU, пам'яті, диску
- кількість активних з'єднань
- кількість горутин/потоків
- температура
- довжина черги

---

### 3.3 Histogram — розподіл значень

**Визначення**: автоматично групує спостереження у попередньо визначені bucket-и (відра). Дозволяє обчислювати **перцентилі** на стороні Prometheus/Grafana.

**Histogram генерує 3 метрики автоматично:**

```
# Для метрики http_request_duration_seconds histogram:

# 1. _bucket — кількість запитів, що вклалися у ≤ N секунд
http_request_duration_seconds_bucket{le="0.005"} 24
http_request_duration_seconds_bucket{le="0.01"}  36
http_request_duration_seconds_bucket{le="0.025"} 89
http_request_duration_seconds_bucket{le="0.05"}  120
http_request_duration_seconds_bucket{le="0.1"}   142
http_request_duration_seconds_bucket{le="0.25"}  155
http_request_duration_seconds_bucket{le="0.5"}   158
http_request_duration_seconds_bucket{le="1"}     159
http_request_duration_seconds_bucket{le="2.5"}   160
http_request_duration_seconds_bucket{le="+Inf"}  160  ← завжди є, = total count

# 2. _count — загальна кількість спостережень
http_request_duration_seconds_count 160

# 3. _sum — сума всіх спостережень
http_request_duration_seconds_sum 12.34
```

**Як читати bucket-и**: кожен bucket є **кумулятивним** (le = "less or equal"). `le="0.1"` означає "скільки запитів тривали ≤ 100ms".

**Обчислення перцентилів у PromQL:**

```promql
# p50 (медіана) латентності за останні 5 хвилин
histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))

# p95 — 95% запитів вклалися у це значення
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# p99 — "довгий хвіст", критично для SLA
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# Середнє значення (з суми та кількості):
rate(http_request_duration_seconds_sum[5m]) / rate(http_request_duration_seconds_count[5m])
```

**Чому важливо правильно задати bucket-и:**

Bucket-и задаються при оголошенні метрики у коді. Якщо ваші запити тривають від 1ms до 10s, але bucket-и задані для 0..1s — перцентилі будуть неточними. Загальне правило: bucket-и мають покривати 95%+ очікуваних значень з достатньою деталізацією.

```python
# Приклад у Python (prometheus_client):
from prometheus_client import Histogram

REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    buckets=[.005, .01, .025, .05, .1, .25, .5, 1.0, 2.5, 5.0, 10.0]
)
```

**Де застосовується**: латентність запитів, розмір payload, час виконання черги.

---

### 3.4 Summary — клієнтські перцентилі

**Визначення**: схожий на Histogram, але перцентилі **обчислюються на стороні клієнта** (в самому застосунку), а не в Prometheus.

```
# HELP go_gc_duration_seconds A summary of GC invocation durations
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 4.9351e-05
go_gc_duration_seconds{quantile="0.25"} 7.5552e-05
go_gc_duration_seconds{quantile="0.5"} 0.000116
go_gc_duration_seconds{quantile="0.75"} 0.000181
go_gc_duration_seconds{quantile="0.99"} 0.001445
go_gc_duration_seconds_sum 0.35
go_gc_duration_seconds_count 2693
```

**Порівняння Histogram vs Summary:**

| Аспект | Histogram | Summary |
|---|---|---|
| Де рахуються квантилі | Сервер (Prometheus) | Клієнт (застосунок) |
| Агрегація across instances | ✅ Можливо | ❌ Неможливо |
| Точність квантилів | Залежить від bucket-ів | Висока |
| Навантаження на клієнт | Мінімальне | Помірне |
| Рекомендація 2024 | **Використовувати** | Тільки якщо точність критична |

**Критична різниця — агрегація:**

```promql
# Histogram — можна агрегувати по всіх інстансах:
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
)

# Summary — НЕ можна агрегувати квантилі по інстансах:
# avg(go_gc_duration_seconds{quantile="0.99"}) — математично НЕКОРЕКТНО!
# Середнє перцентилів ≠ перцентиль всієї вибірки
```

---

## 4. PromQL: rate, increase, irate та інші функції

### 4.1 rate() — швидкість зміни counter

```promql
rate(http_requests_total[5m])
```

**Що робить**: обчислює середню швидкість зміни counter за вказаний часовий інтервал.

**Алгоритм під капотом**:
1. Знаходить усі scrape-точки у вікні `[5m]`
2. Обробляє reset-и counter (якщо значення впало — значить був рестарт)
3. Обчислює: `(last_value - first_value) / duration_in_seconds`
4. Результат: значення **в одиницях на секунду**

```promql
# rate(http_requests_total[5m]) = 2.3
# Означає: в середньому 2.3 запити/секунду за останні 5 хвилин

# Для отримання запитів/хвилину:
rate(http_requests_total[5m]) * 60

# Відсоток помилок:
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) * 100
```

**Правило вибору інтервалу**: інтервал `[Xm]` має бути мінімум у **4 рази більший** за scrape interval.
- scrape_interval = 15s → мінімальний інтервал `[1m]`, рекомендований `[5m]`
- scrape_interval = 60s → мінімальний `[4m]`, рекомендований `[10m]`

Занадто малий інтервал → нестабільні результати (мало точок для обчислення).

---

### 4.2 increase() — абсолютне збільшення за інтервал

```promql
increase(http_requests_total[1h])
```

**Що робить**: скільки разів counter збільшився за вказаний час.

**Математика**: `increase(m[d]) = rate(m[d]) * duration_in_seconds`

```promql
# Кількість запитів за останню годину:
increase(http_requests_total[1h])

# Кількість 5xx помилок за останні 24 години:
increase(http_requests_total{status=~"5.."}[24h])

# Кількість failed jobs за останній день:
increase(batch_jobs_failed_total[24h])
```

**Важливо**: `increase()` також обробляє reset-и counter, тобто якщо процес перезапустився — це не псує результат.

---

### 4.3 irate() — миттєва швидкість (остання пара точок)

```promql
irate(http_requests_total[5m])
```

**Що робить**: обчислює rate тільки **між двома останніми** scrape-точками у вікні.

**Різниця від rate():**

```
Часові точки:  t=0   t=15s   t=30s   t=45s   t=60s   t=75s
Значення:      100   115     130     160     161     163

rate([5m]) = середнє за всі точки = плавний результат
irate([5m]) = (163 - 161) / 15 = 0.13/s  (тільки останні 2 точки)
```

| Функція | Переваги | Недоліки |
|---|---|---|
| `rate()` | Плавна, стабільна, добре для алертів | Може "замилювати" піки |
| `irate()` | Реагує на різкі піки | Нестабільна, шумна; погана для алертів |

**Коли використовувати irate()**: тільки для графіків реального часу, де потрібно бачити піки. **Ніколи** у правилах алертів.

---

### 4.4 Інші важливі функції

```promql
# --- Агрегаційні оператори ---

# Сума по всіх інстансах:
sum(rate(http_requests_total[5m]))

# Сума, зберігаючи тільки label "job":
sum by (job) (rate(http_requests_total[5m]))

# Сума, прибираючи тільки label "instance":
sum without (instance) (rate(http_requests_total[5m]))

# Максимум по всіх інстансах:
max by (instance) (node_load1)

# Топ-5 найзавантаженіших хостів:
topk(5, node_load1)

# --- Функції часу ---

# Скільки секунд тому відбувся останній scrape (для виявлення "мертвих" таргетів):
time() - node_boot_time_seconds  # uptime в секундах

# Прогноз вільного місця (лінійна екстраполяція):
predict_linear(node_filesystem_free_bytes{mountpoint="/"}[6h], 24*3600)
# "Якщо тренд останніх 6h продовжиться — скільки байт буде через 24h"

# --- Зміна у часі ---

# Різниця між поточним і N секунд тому (для gauge):
delta(node_load1[10m])

# Похідна (trend):
deriv(node_load1[10m])

# --- Відбір по значенню ---

# Тільки інстанси з load > 2.0:
node_load1 > 2.0

# Тільки інстанси де є помилки (rate > 0):
rate(http_requests_total{status=~"5.."}[5m]) > 0

# Умовний вираз — якщо error rate > 0.01, повернути значення, інакше nothing:
rate(http_requests_total{status=~"5.."}[5m]) > 0.01
```

---

## 5. Label matching та Vector matching

```promql
# Бінарні операції між метриками вимагають збігу labels

# Відсоток вільної пам'яті (обидві метрики мають label "instance"):
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100

# Якщо labels не збігаються — потрібен ignoring або on:
# Помилки на запит (метрики мають різні sets labels):
rate(http_errors_total[5m]) / ignoring(error_type) rate(http_requests_total[5m])

# group_left — many-to-one join (додати label з іншої метрики):
rate(http_requests_total[5m])
* on(instance) group_left(datacenter)
node_meta_info  # метрика з додатковим label "datacenter"
```

---

## 6. Grafana: побудова дашбордів

### 6.1 Типи панелей і коли які використовувати

| Тип панелі | Використання |
|---|---|
| **Time series** | Будь-які метрики у часі (CPU, latency, RPS) |
| **Stat** | Одне поточне значення (uptime, поточний RPS) |
| **Gauge** | Значення у відсотках з кольоровими зонами |
| **Bar chart** | Порівняння між сутностями (топ хостів) |
| **Heatmap** | Розподіл histogram по часу |
| **Table** | Детальні дані (список алертів, топ помилок) |
| **Logs** | Loki logs |

### 6.2 Приклад: HTTP RPS дашборд

**Панель: Requests per second by status**
```promql
# Query A — успішні запити:
sum by (status) (rate(http_requests_total{job="my-api"}[5m]))

# Legend: {{status}}
# Visualization: Time series, Fill opacity: 20
```

**Панель: Error rate %**
```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
* 100

# Unit: Percent (0-100)
# Thresholds: 0=green, 1=yellow, 5=red
```

**Панель: p50/p95/p99 latency**
```promql
# Query A — p50:
histogram_quantile(0.50, sum by (le) (rate(http_request_duration_seconds_bucket{job="my-api"}[5m])))

# Query B — p95:
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket{job="my-api"}[5m])))

# Query C — p99:
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{job="my-api"}[5m])))

# Legend: p{{quantile_label}}   (або задати вручну: p50, p95, p99)
# Unit: seconds (s)
```

### 6.3 Template variables для динамічних дашбордів

```yaml
# У Settings → Variables:
# Variable 1:
Name: instance
Type: Query
Query: label_values(node_cpu_seconds_total, instance)
# Тепер у запитах використовуємо: {instance="$instance"}

# Variable 2:
Name: interval
Type: Interval
Values: 1m,5m,10m,30m,1h
Default: 5m
# Використовуємо: rate(metric[$interval])
```

```promql
# З variables — динамічний запит:
rate(http_requests_total{instance="$instance", job="$job"}[$interval])
```

### 6.4 Heatmap для Histogram

Heatmap — найкращий спосіб візуалізувати розподіл латентності у часі:

```promql
# Query для Heatmap панелі:
sum(rate(http_request_duration_seconds_bucket{job="my-api"}[$interval])) by (le)

# Format: Heatmap
# Bucket bound: Upper (тому що le = less-or-equal, тобто верхня межа)
```

---

## 7. Alertmanager: маршрутизація і дедублікація

### 7.1 Правила алертів у Prometheus

```yaml
# /etc/prometheus/rules/api.yml
groups:
  - name: api.alerts
    interval: 1m  # як часто перевіряти (default: global evaluation_interval)
    rules:

      # Алерт на high error rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m]))
          * 100 > 5
        for: 5m
        # ^ "for" — алерт спрацьовує тільки якщо умова TRUE протягом 5 хвилин безперервно.
        # Без "for": будь-який спайк → негайний алерт (багато false positives)
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High error rate on {{ $labels.job }}"
          description: |
            Error rate is {{ $value | printf "%.2f" }}% 
            for job {{ $labels.job }} on instance {{ $labels.instance }}.
            Current threshold: 5%

      # Алерт на latency
      - alert: HighLatencyP99
        expr: |
          histogram_quantile(0.99,
            sum by (le, job) (rate(http_request_duration_seconds_bucket[5m]))
          ) > 1.0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "p99 latency > 1s for {{ $labels.job }}"
          description: "p99 = {{ $value | printf \"%.3f\" }}s"

      # Алерт на відсутність метрик (dead exporter)
      - alert: ExporterDown
        expr: up == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Exporter down: {{ $labels.instance }}"
```

**Цикл стану алерту:**
```
INACTIVE → PENDING → FIRING → RESOLVED
           ^          ^
           │          └── умова true > for duration
           └── умова стала true
```

### 7.2 Конфігурація Alertmanager

```yaml
# /etc/alertmanager/alertmanager.yml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@company.com'
  slack_api_url: 'https://hooks.slack.com/services/T.../B.../xxx'

# Шаблони повідомлень
templates:
  - '/etc/alertmanager/templates/*.tmpl'

# Дерево маршрутизації
route:
  receiver: 'default-receiver'  # fallback
  group_by: ['alertname', 'job']
  # ^ групувати алерти з однаковим alertname+job в одне повідомлення
  group_wait: 30s
  # ^ чекати 30s перед відправкою першої нотифікації (щоб зібрати групу)
  group_interval: 5m
  # ^ мінімальний інтервал між повідомленнями для тієї ж групи
  repeat_interval: 4h
  # ^ якщо алерт не resolved — повторювати кожні 4h

  routes:
    # Critical → Slack #alerts-critical + PagerDuty
    - match:
        severity: critical
      receiver: 'critical-receiver'
      continue: false  # не передавати далі по дереву

    # Warning → Slack #alerts-warning тільки в робочий час
    - match:
        severity: warning
      receiver: 'warning-receiver'
      active_time_intervals:
        - business_hours

    # Конкретна команда
    - match_re:
        team: ^(backend|api)$
      receiver: 'backend-team'

time_intervals:
  - name: business_hours
    time_intervals:
      - weekdays: ['monday:friday']
        times:
          - start_time: '09:00'
            end_time: '18:00'
        location: 'Europe/Kyiv'

receivers:
  - name: 'default-receiver'
    slack_configs:
      - channel: '#alerts-general'
        send_resolved: true
        title: '{{ template "slack.title" . }}'
        text: '{{ template "slack.text" . }}'

  - name: 'critical-receiver'
    slack_configs:
      - channel: '#alerts-critical'
        send_resolved: true
    pagerduty_configs:
      - service_key: '<YOUR_PD_KEY>'

  - name: 'backend-team'
    slack_configs:
      - channel: '#backend-alerts'

inhibit_rules:
  # Якщо instance Down — не посилати інші алерти для цього instance
  - source_match:
      alertname: InstanceDown
    target_match_re:
      alertname: .+
    equal: ['instance']
```

### 7.3 Silence — заглушення алертів

```bash
# CLI через amtool:
amtool silence add \
  --alertmanager.url=http://alertmanager:9093 \
  --duration=2h \
  --comment="Planned maintenance window" \
  alertname=HighErrorRate \
  job=my-api

# Переглянути активні silences:
amtool silence query --alertmanager.url=http://alertmanager:9093
```

---

## 8. Prometheus конфігурація: scrape та service discovery

```yaml
# /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s         # як часто збирати метрики
  evaluation_interval: 15s     # як часто обчислювати alerting rules
  scrape_timeout: 10s          # timeout для одного scrape

# Правила та алерти
rule_files:
  - "/etc/prometheus/rules/*.yml"

# Alertmanager
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

scrape_configs:
  # --- Prometheus сам себе ---
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # --- Node Exporter ---
  - job_name: 'node'
    static_configs:
      - targets:
          - 'server1.example.com:9100'
          - 'server2.example.com:9100'
    relabel_configs:
      # Замінити instance label на короткий hostname:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__address__]
        regex: '([^:]+):.*'
        target_label: instance
        replacement: '$1'

  # --- HAProxy exporter ---
  - job_name: 'haproxy'
    static_configs:
      - targets: ['haproxy-host:8405']
    labels:
      datacenter: 'nj'

  # --- File-based service discovery (динамічне оновлення без рестарту) ---
  - job_name: 'dynamic-targets'
    file_sd_configs:
      - files:
          - '/etc/prometheus/targets/*.json'
        refresh_interval: 30s

  # --- Blackbox exporter (зовнішній моніторинг HTTP) ---
  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - 'https://api.example.com/health'
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

**File-based SD — приклад файлу:**
```json
[
  {
    "targets": ["server10.dc1.example.com:9100", "server11.dc1.example.com:9100"],
    "labels": {
      "datacenter": "dc1",
      "role": "proxy"
    }
  }
]
```

---

## 9. Recording Rules — pre-computed метрики

Recording rules дозволяють **попередньо обчислити** важкі запити і зберегти результат як нову метрику.

```yaml
# /etc/prometheus/rules/recording.yml
groups:
  - name: api_recording_rules
    interval: 1m
    rules:
      # Pre-compute error rate по job:
      - record: job:http_request_errors:rate5m
        expr: |
          sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum by (job) (rate(http_requests_total[5m]))

      # Pre-compute p99 latency по job:
      - record: job:http_request_duration_p99:rate5m
        expr: |
          histogram_quantile(0.99,
            sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))
          )
```

**Конвенція найменування**: `level:metric:operations`
- `level` — рівень агрегації (job, instance, datacenter)
- `metric` — базова метрика
- `operations` — застосовані функції (rate5m, p99, etc.)

**Коли використовувати recording rules:**
- запит виконується у > 100ms (важкий для Grafana)
- запит використовується на кількох дашбордах/алертах
- потрібна довгострокова агрегація (тижні/місяці)

---

## 10. Best Practices: практичний чеклист

### 10.1 Найменування метрик

```
# Правила:
# 1. snake_case
# 2. Суфікс одиниці виміру:
#    _seconds, _bytes, _total, _ratio, _celsius, _info
# 3. Counter → _total суфікс
# 4. Не включати label name у metric name

# Добре:
http_request_duration_seconds
node_memory_free_bytes
http_requests_total

# Погано:
httpRequestDuration      # camelCase
http_request_time        # немає одиниці
http_get_requests_total  # метод (GET) краще як label
```

### 10.2 Labels: що варто і що не варто

```
# Добре — низька кардинальність, постійні значення:
{method="GET", status="200", datacenter="us-east-1"}

# Погано — висока кардинальність:
{user_id="12345"}          # мільйони користувачів
{url="/api/users/12345"}   # унікальний URL
{request_id="abc-xyz"}     # унікальний на запит
{timestamp="2024-01-01"}   # унікальне значення
```

### 10.3 Кардинальність: моніторинг і контроль

```promql
# Скільки активних time series у Prometheus:
prometheus_tsdb_head_series

# Топ метрик за кардинальністю:
topk(10, count by (__name__) ({__name__=~".+"}))

# Кількість series на метрику:
count by (__name__) ({__name__="http_requests_total"})
```

```yaml
# Обмеження кардинальності на рівні scrape:
scrape_configs:
  - job_name: 'my-app'
    sample_limit: 5000  # якщо > 5000 samples — весь scrape відкидається
    label_limit: 30     # максимум labels на серію
```

### 10.4 Retention та Storage

```yaml
# prometheus.yml запуск:
--storage.tsdb.retention.time=30d    # зберігати 30 днів
--storage.tsdb.retention.size=50GB   # або до 50GB (спрацьовує перший)

# Оцінка розміру (приблизна формула):
# needed_disk = retention_seconds * ingested_samples_per_second * bytes_per_sample
# bytes_per_sample ≈ 1-2 байти (після стиснення XOR delta encoding)
# 
# Приклад: 30d × 10,000 samples/s × 2 bytes = ~51GB
```

### 10.5 Rate/Increase: типові помилки

```promql
# ПОМИЛКА: rate на gauge
rate(node_memory_MemAvailable_bytes[5m])  # ❌ безглуздо — gauge не counter

# ПОМИЛКА: irate в alerting rule
alert: HighRPS
expr: irate(http_requests_total[5m]) > 1000  # ❌ нестабільний результат

# ПОМИЛКА: занадто малий інтервал
rate(http_requests_total[30s])  # ❌ якщо scrape_interval=15s — тільки 2 точки!

# ПРАВИЛЬНО:
rate(http_requests_total[5m])   # ✅ плавно, 20 точок при scrape 15s
```

### 10.6 Алерти: проти anti-patterns

```yaml
# ПОГАНО: алерт без "for" → шквал false positives
- alert: HighCPU
  expr: node_load1 > 2.0
  # ← немає "for"! Будь-який секундний спайк = алерт

# ДОБРЕ: з обґрунтованим "for"
- alert: HighCPU
  expr: node_load1 > 2.0
  for: 10m  # проблема має бути стабільною

# ПОГАНО: алерт на сирий counter
- alert: ErrorsSpike
  expr: http_requests_total{status="500"} > 100  # ❌

# ДОБРЕ: алерт на rate
- alert: ErrorsSpike
  expr: rate(http_requests_total{status="500"}[5m]) > 0.1  # ✅
```

### 10.7 Grafana: UX та організація

```
Dashboard структура (рекомендована):
├── Row: Overview (Stat/Gauge панелі — одне число)
│   ├── Current RPS
│   ├── Error rate %
│   └── p99 Latency
├── Row: Traffic (Time series)
│   ├── RPS by status code
│   └── Bandwidth in/out
├── Row: Latency (Time series + Heatmap)
│   ├── p50/p95/p99 over time
│   └── Latency distribution heatmap
├── Row: Errors (Time series + Table)
│   ├── Error rate over time
│   └── Top errors (Table)
└── Row: Infrastructure
    ├── CPU / Memory
    └── Network I/O
```

---

## 11. Повний робочий приклад: моніторинг HAProxy

> Реальний приклад для HAProxy native Prometheus exporter (порт 8405)

### Метрики HAProxy

```
# Поточні з'єднання (gauge):
haproxy_frontend_current_sessions{proxy="http-in"}

# Запити (counter):
haproxy_frontend_http_requests_total{proxy="http-in"}

# HTTP відповіді по кодах (counter):
haproxy_frontend_http_responses_total{proxy="http-in", code="1xx"}
haproxy_frontend_http_responses_total{proxy="http-in", code="5xx"}

# Байти in/out (counter):
haproxy_frontend_bytes_in_total
haproxy_frontend_bytes_out_total

# Backend servers up/down (gauge, 0 або 1):
haproxy_server_status{proxy="backend", server="app1"}
```

### PromQL запити для дашборду

```promql
# RPS по бекенду:
sum by (proxy) (rate(haproxy_frontend_http_requests_total[5m]))

# Error rate 5xx:
sum(rate(haproxy_frontend_http_responses_total{code="5xx"}[5m]))
/
sum(rate(haproxy_frontend_http_requests_total[5m]))
* 100

# Кількість живих серверів у кожному бекенді:
sum by (proxy) (haproxy_server_status == 1)

# Пропускна здатність (Мбіт/с):
sum(rate(haproxy_frontend_bytes_in_total[5m])) * 8 / 1e6

# Активні з'єднання vs максимум:
haproxy_frontend_current_sessions / haproxy_frontend_limit_sessions * 100
```

### Alerting rules для HAProxy

```yaml
groups:
  - name: haproxy
    rules:
      # Сервер виключився з ротації
      - alert: HAProxyServerDown
        expr: haproxy_server_status == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "HAProxy server {{ $labels.server }} is DOWN in {{ $labels.proxy }}"

      # Більше 50% серверів бекенду недоступні
      - alert: HAProxyBackendHalfDown
        expr: |
          sum by (proxy) (haproxy_server_status == 0)
          /
          sum by (proxy) (haproxy_server_status)
          > 0.5
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "More than 50% servers down in backend {{ $labels.proxy }}"

      # Висока кількість 5xx помилок
      - alert: HAProxyHighErrorRate
        expr: |
          sum by (proxy) (rate(haproxy_frontend_http_responses_total{code="5xx"}[5m]))
          /
          sum by (proxy) (rate(haproxy_frontend_http_requests_total[5m]))
          * 100 > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "HAProxy 5xx error rate > 5% on {{ $labels.proxy }}"
          description: "Current rate: {{ $value | printf \"%.2f\" }}%"

      # Насичення з'єднань
      - alert: HAProxySessionSaturation
        expr: |
          haproxy_frontend_current_sessions
          / haproxy_frontend_limit_sessions
          * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "HAProxy session limit at {{ $value | printf \"%.0f\" }}%"
```

---

## 12. Діагностика та troubleshooting

```bash
# --- Prometheus UI (порт 9090) ---
# /graph         — PromQL playground
# /targets       — стан всіх таргетів (UP/DOWN)
# /rules         — стан recording/alerting rules
# /alerts        — активні алерти та їх стан
# /tsdb-status   — кардинальність, топ series за розміром
# /config        — поточна конфігурація

# --- Перевірка конфігурації перед застосуванням ---
promtool check config /etc/prometheus/prometheus.yml
promtool check rules /etc/prometheus/rules/*.yml

# --- Валідація alertmanager конфігу ---
amtool check-config /etc/alertmanager/alertmanager.yml

# --- Тест алерту (симуляція firing) ---
amtool alert add \
  --alertmanager.url=http://alertmanager:9093 \
  alertname=TestAlert \
  severity=warning \
  summary="Test alert from CLI"

# --- Розмір TSDB на диску ---
du -sh /var/lib/prometheus/

# --- API для отримання series ---
curl 'http://localhost:9090/api/v1/series?match[]=http_requests_total&start=2024-01-01T00:00:00Z&end=2024-01-01T01:00:00Z'

# --- Скільки samples ingested/s ---
curl -s 'http://localhost:9090/api/v1/query?query=rate(prometheus_tsdb_head_samples_appended_total[1m])' \
  | python3 -m json.tool
```

---

## Підсумок

| Концепція | Ключовий висновок |
|---|---|
| **Pull model** | Prometheus сам ходить до таргетів — централізований контроль |
| **Time series** | Унікальний ряд = metric name + набір labels |
| **Counter** | Тільки зростає → завжди використовувати `rate()`/`increase()` |
| **Gauge** | Поточний стан → використовувати напряму |
| **Histogram** | Buckets + server-side quantiles → агрегується між інстансами |
| **Summary** | Client-side quantiles → НЕ агрегується між інстансами |
| **rate()** | Плавна середня швидкість → для алертів і дашбордів |
| **irate()** | Миттєва швидкість між 2 точками → тільки для live графіків |
| **Cardinality** | Головна небезпека OOM → уникати dynamic labels |
| **for: у alertах** | Захист від false positives → мінімум 2-5 хвилин |
| **Recording rules** | Pre-compute важких запитів → пришвидшують Grafana |
| **inhibit_rules** | Пригнічення похідних алертів → менше шуму |
