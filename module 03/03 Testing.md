Вот лабораторная работа для проверки отказоустойчивости и распределения поисковых запросов в кластере OpenSearch. Она основана на предыдущей конфигурации с разделением ролей и статическими IP-адресами.

---

## 🧪 Лабораторная работа: Проверка отказоустойчивости и распределения запросов в OpenSearch

### Цель работы

Проверить, как кластер OpenSearch с разделением ролей (Cluster Manager, Data, Search) ведёт себя при сбоях, и убедиться, что поисковые запросы распределяются между доступными репликами. В ходе работы вы смоделируете отказ различных узлов, проанализируете поведение кластера и оцените балансировку нагрузки.

### Теоретическое введение

**Отказоустойчивость** кластера OpenSearch зависит от:
- **Кворума** — для выбора cluster manager требуется большинство голосов. Кластер из 3 cluster-manager-eligible узлов может выдержать отказ 1 узла.
- **Реплик** — при отказе узла с первичным шардом, реплика на другом узле автоматически продвигается в primary.
- **Voting configuration** — OpenSearch обновляет список голосующих узлов при их добавлении или удалении.

**Распределение поисковых запросов**:
- OpenSearch использует все доступные узлы для маршрутизации поисковых запросов.
- Наличие реплик позволяет распределять нагрузку: запросы могут обслуживаться как с primary, так и с replica шардов.
- При использовании выделенных search-узлов, запросы направляются на search-реплики.

---

### Часть 1. Подготовка среды

Предполагается, что у вас уже развёрнут кластер из предыдущей лабораторной работы (8 узлов с разделением ролей и статическими IP). При необходимости разверните его заново:

```bash
cd ~/opensearch-roles-lab
docker compose up -d
```

**Архитектура кластера:**

| Роль | Имя контейнера | IP-адрес | Порт (хост) |
|------|---------------|----------|-------------|
| Cluster Manager 1 | opensearch-cm1 | 172.20.0.10 | 9201 |
| Cluster Manager 2 | opensearch-cm2 | 172.20.0.11 | 9202 |
| Cluster Manager 3 | opensearch-cm3 | 172.20.0.12 | 9203 |
| Data 1 | opensearch-d1 | 172.20.0.20 | 9204 |
| Data 2 | opensearch-d2 | 172.20.0.21 | 9205 |
| Search 1 | opensearch-s1 | 172.20.0.30 | 9206 |
| Search 2 | opensearch-s2 | 172.20.0.31 | 9207 |
| Coordinating | opensearch-c1 | 172.20.0.40 | 9200 |

---

### Часть 2. Подготовка тестовых данных

#### Шаг 1. Создание индекса с репликами

```bash
curl -k -u admin:MyStrongPassword123! -X PUT "https://localhost:9200/test-failover" -H 'Content-Type: application/json' -d '
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 2
  },
  "mappings": {
    "properties": {
      "title": { "type": "text" },
      "category": { "type": "keyword" },
      "timestamp": { "type": "date" }
    }
  }
}'
```

**Ожидаемый результат:** `{"acknowledged":true}`

#### Шаг 2. Загрузка тестовых документов

Создайте скрипт для загрузки 1000 тестовых документов:

```bash
for i in {1..1000}; do
  CATEGORY=$((RANDOM % 5 + 1))
  curl -k -u admin:MyStrongPassword123! -X POST "https://localhost:9200/test-failover/_doc/$i" \
    -H 'Content-Type: application/json' \
    -d "{\"title\": \"Document $i\", \"category\": \"cat$CATEGORY\", \"timestamp\": \"2026-08-04T$(printf '%02d' $((RANDOM % 24))):$(printf '%02d' $((RANDOM % 60))):$(printf '%02d' $((RANDOM % 60)))\"}" \
    2>/dev/null
done
echo "Загружено 1000 документов"
```

#### Шаг 3. Проверка распределения шардов

```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cat/shards/test-failover?v&h=index,shard,prirep,node,state"
```

Пример вывода:
```
index          shard prirep node            state
test-failover  0     p      opensearch-d1   STARTED
test-failover  0     r      opensearch-s1   STARTED
test-failover  0     r      opensearch-s2   STARTED
test-failover  1     p      opensearch-d2   STARTED
test-failover  1     r      opensearch-s1   STARTED
test-failover  1     r      opensearch-s2   STARTED
test-failover  2     p      opensearch-d1   STARTED
test-failover  2     r      opensearch-s2   STARTED
test-failover  2     r      opensearch-d2   STARTED
```

> **Обратите внимание:** Primary шарды (`p`) размещены на data-узлах, а реплики (`r`) — на search-узлах и data-узлах.

---

### Часть 3. Проверка отказоустойчивости

#### Тест 1. Отказ data-узла (потеря primary шарда)

**Шаг 1.** Проверьте текущее состояние кластера:
```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cluster/health?pretty"
```
Статус должен быть `green`.

**Шаг 2.** Остановите data-узел `opensearch-d1`:
```bash
docker stop opensearch-d1
```

**Шаг 3.** Проверьте состояние кластера:
```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cluster/health?pretty"
```

**Ожидаемый результат:** Статус станет `yellow`, так как некоторые реплики стали неактивными. Однако кластер продолжает работать, так как primary шарды были перераспределены.

**Шаг 4.** Проверьте, какие шарды стали primary:
```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cat/shards/test-failover?v&h=index,shard,prirep,node"
```

Вы увидите, что реплики на других узлах были повышены до primary.

**Шаг 5.** Выполните поисковый запрос, чтобы убедиться, что данные доступны:
```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/test-failover/_search?q=*&size=5" | jq '.hits.total.value'
```
Должно вернуться `1000`.

**Шаг 6.** Восстановите узел:
```bash
docker start opensearch-d1
```

**Шаг 7.** Подождите 30 секунд и проверьте, что кластер вернулся в состояние `green`:
```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cluster/health?pretty"
```

**Вывод:** Кластер выдержал отказ data-узла благодаря наличию реплик.

---

#### Тест 2. Отказ cluster manager узла

**Шаг 1.** Определите текущего active cluster manager:
```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cat/master?v&h=id,host,node"
```
Запомните имя узла.

**Шаг 2.** Остановите активный cluster manager узел (например, если активный `opensearch-cm1`):
```bash
docker stop opensearch-cm1
```

**Шаг 3.** Проверьте состояние кластера:
```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cluster/health?pretty"
```

**Ожидаемый результат:** Кластер остаётся доступным (статус `yellow` или `green`), так как остальные 2 cluster manager узла обеспечивают кворум.

**Шаг 4.** Определите нового active cluster manager:
```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cat/master?v&h=id,host,node"
```

**Шаг 5.** Выполните операцию создания индекса, чтобы убедиться, что кластер управляем:
```bash
curl -k -u admin:MyStrongPassword123! -X PUT "https://localhost:9200/test-after-failover"
```

**Ожидаемый результат:** `{"acknowledged":true}`

**Шаг 6.** Восстановите узел:
```bash
docker start opensearch-cm1
```

**Вывод:** Кластер из 3 cluster manager узлов выдерживает отказ одного узла без потери управляемости.

---

#### Тест 3. Отказ search-узла

**Шаг 1.** Проверьте распределение шардов до отказа:
```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cat/shards/test-failover?v&h=index,shard,prirep,node"
```

**Шаг 2.** Остановите search-узел `opensearch-s1`:
```bash
docker stop opensearch-s1
```

**Шаг 3.** Проверьте состояние кластера:
```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cluster/health?pretty"
```

**Ожидаемый результат:** Статус может стать `yellow`, так как некоторые реплики стали неактивными.

**Шаг 4.** Проверьте, что поиск всё ещё работает:
```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/test-failover/_search?q=category:cat1&size=10" | jq '.hits.total.value'
```

**Шаг 5.** Проверьте, как перераспределились реплики:
```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cat/shards/test-failover?v&h=index,shard,prirep,node"
```

**Ожидаемый результат:** Реплики, которые были на `opensearch-s1`, перераспределились на другие узлы.

**Шаг 6.** Восстановите узел:
```bash
docker start opensearch-s1
```

**Вывод:** Отказ search-узла не влияет на доступность данных, так как реплики присутствуют на других узлах.

---

### Часть 4. Проверка распределения поисковых запросов

#### Тест 4. Балансировка запросов между репликами

**Шаг 1.** Выполните серию поисковых запросов и запишите, какой узел их обрабатывает. Для этого включите вывод узла-обработчика:

```bash
for i in {1..20}; do
  curl -k -u admin:MyStrongPassword123! "https://localhost:9200/test-failover/_search?q=*&size=1&pretty" 2>/dev/null | grep -E '"node"|"took"' | head -2
  echo "---"
done
```

**Шаг 2.** Проанализируйте, какие узлы обрабатывали запросы. В идеале запросы должны распределяться между search-узлами и data-узлами, на которых есть реплики.

**Шаг 3.** Для более детального анализа используйте API `_nodes/stats`:
```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_nodes/stats/indices/search?pretty" | jq '.nodes | to_entries[] | {node: .key, query_total: .value.indices.search.query_total}'
```

Это покажет общее количество поисковых запросов, обработанных каждым узлом.

**Ожидаемый результат:** Нагрузка распределена между search-узлами и data-узлами.

---

#### Тест 5. Проверка маршрутизации при отключении search-узла

**Шаг 1.** Остановите оба search-узла:
```bash
docker stop opensearch-s1 opensearch-s2
```

**Шаг 2.** Проверьте состояние кластера:
```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cluster/health?pretty"
```

**Шаг 3.** Выполните поисковый запрос:
```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/test-failover/_search?q=*&size=5" | jq '.hits.total.value'
```

**Ожидаемый результат:** Запрос всё ещё выполняется, так как реплики также находятся на data-узлах. Однако нагрузка теперь ложится на data-узлы.

**Шаг 4.** Восстановите search-узлы:
```bash
docker start opensearch-s1 opensearch-s2
```

---

#### Тест 6. Проверка strict routing для search-реплик

Если в вашем кластере настроены выделенные search-узлы, вы можете проверить, что поисковые запросы по умолчанию направляются именно на них.

**Шаг 1.** Проверьте настройку `cluster.routing.search_only.strict`:
```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cluster/settings?include_defaults=true&pretty" | grep -A2 "search_only"
```

**Шаг 2.** Если параметр включён, запросы должны направляться только на search-узлы. При отключении всех search-узлов запросы могут не выполняться (если нет реплик на data-узлах).

---

### Часть 5. Задания для самостоятельного выполнения

**Задание 1.** Проверка отказа coordinating-узла:
- Остановите coordinating узел (`opensearch-c1`).
- Подключитесь напрямую к одному из data-узлов (порт 9204 или 9205) и выполните поисковый запрос.
- Сравните результат с запросом через coordinating узел.

**Задание 2.** Симуляция потери кворума:
- Остановите два cluster manager узла одновременно (`docker stop opensearch-cm1 opensearch-cm2`).
- Проверьте состояние кластера.
- **Ожидаемый результат:** Кластер становится недоступным для управления (потеря кворума).
- Восстановите узлы.

**Задание 3.** Нагрузочное тестирование:
- Используйте утилиту `wrk` или `ab` для отправки множества поисковых запросов.
- Замерьте время отклика до и после отключения search-узлов.
- Сравните результаты.

**Задание 4.** Проверка auto-rebalance:
- Создайте новый индекс с 5 шардами и 1 репликой.
- Остановите один data-узел.
- Наблюдайте, как OpenSearch автоматически перераспределяет шарды на оставшиеся узлы.
- Используйте API `_cluster/reroute` для ручного управления.

---

### Часть 6. Мониторинг и визуализация

#### Шаг 1. Просмотр статистики в OpenSearch Dashboards

1. Откройте OpenSearch Dashboards (`http://<IP>:5601`).
2. Перейдите в **Dashboards Management** → **Dev Tools**.
3. Выполните запросы для мониторинга:

```json
GET _cluster/health?pretty
GET _cat/nodes?v&h=name,node.role,heap.percent,ram.percent,cpu,load_1m
GET _cat/shards/test-failover?v&h=index,shard,prirep,node,state
```

#### Шаг 2. Создание дашборда для мониторинга

1. В OpenSearch Dashboards создайте визуализации:
   - **Pie chart** — распределение шардов по узлам.
   
   Vega  
   ```
   {
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "data": {
    "url": {
      "%context%": true,
      "path": "/_cat/shards?format=json"
    },
    "format": {"property": "name_of_root_array"}
  },
  "mark": "arc",
  "encoding": {
    "theta": {
      "aggregate": "count",
      "field": "node",
      "type": "quantitative"
    },
    "color": {
      "field": "node",
      "type": "nominal",
      "legend": {"title": "Узел"}
    },
    "tooltip": [
      {"field": "node", "type": "nominal", "title": "Узел"},
      {"aggregate": "count", "field": "node", "type": "quantitative", "title": "Количество шардов"}
    ]
  },
  "config": {"view": {"stroke": null}}
}
   ```
     
   - **Line chart** — количество запросов в секунду.
   - **Metric** — общее количество документов.

3. Объедините их в дашборд **Cluster Health Dashboard**.




**Ключевые выводы:**
- Реплики критически важны для отказоустойчивости.
- Три cluster manager узла — минимальная конфигурация для production.
- Выделенные search-узлы улучшают производительность и изолируют нагрузки.
- OpenSearch автоматически перераспределяет шарды при сбоях.
