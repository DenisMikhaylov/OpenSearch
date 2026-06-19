# Лабораторная работа: Полный цикл работы с NGINX логами в OpenSearch 3.7

##  Настройка среды для сбора NGINX логов

### 2.1. Требования к системе

Перед началом работы убедитесь, что:
- Установлен Docker и Docker Compose
- На сервере запущен NGINX (логи находятся в `/var/log/nginx/access.log`)
- Настроен OpenSearch кластер (локально или удалённо)

### 2.2. Настройка Fluent Bit

Fluent Bit будет читать логи NGINX из файла `/var/log/nginx/access.log` и отправлять их в Data Prepper .

**Полный путь к файлу конфигурации:** `/etc/fluent-bit/fluent-bit.conf`

```ini
[INPUT]
    name tail
    Tag nginx.access
    path /var/log/nginx/access.log
    parser nginx

[INPUT]
    Name  tail
    Tag   nginx.error
    path  /var/log/nginx/error.log

[OUTPUT]
    Name  http
    Match nginx.access
    Host  data-prepper
    Port  2021
    URI   /log/ingest
    Format json

[OUTPUT]
    Name  http
    Match nginx.error
    Host  data-prepper
    Port  2021
    URI   /log/ingest
    Format json
```

**Пояснение параметров:**
- `tail` — следит за изменениями в файле 
- `path` — путь к лог-файлу NGINX
- `parser nginx` — использует встроенный парсер Fluent Bit для NGINX логов
- `Host` и `Port` — адрес Data Prepper (по умолчанию порт 2021) 

### 2.3. Настройка Data Prepper

Data Prepper получает логи от Fluent Bit и обрабатывает их с помощью Grok Processor для структурирования неструктурированных логов NGINX .

**Полный путь к файлу:** `./log_pipeline.yaml` (в каталоге запуска Data Prepper)

```yaml
log-pipeline:
  workers: 4
  delay: 100
  source:
    http:
      ssl: false
      port: 2021
  buffer:
    bounded_blocking:
      buffer_size: 2048
      batch_size: 256
  processor:
    - grok:
        match:
          log: [ "%{COMMONAPACHELOG}" ]
  sink:
    - opensearch:
        hosts: [ "http://opensearch:9200" ]
        insecure: true
        username: "admin"
        password: "admin"
        index: nginx_logs
```

**Пояснение компонентов:**
- **HTTP Source** — принимает данные от Fluent Bit на порту 2021
- **Grok Processor** — парсит логи NGINX по шаблону `COMMONAPACHELOG`, извлекая поля: client IP, timestamp, HTTP метод, URI, статус код, размер ответа
- **OpenSearch Sink** — отправляет обработанные данные в кластер OpenSearch в индекс `nginx_logs`

### 2.4. Запуск Data Prepper

Используйте Docker для запуска Data Prepper :

```bash
# Скачивание образа Data Prepper
docker pull opensearchproject/data-prepper:2

# Запуск контейнера
docker run --name data-prepper \
  -v $(pwd)/log_pipeline.yaml:/usr/share/data-prepper/pipelines/log_pipeline.yaml \
  --network "opensearch-net" \
  -p 2021:2021 \
  opensearchproject/data-prepper:2
```

### 2.5. Запуск OpenSearch через Docker Compose

Для удобства можно использовать Docker Compose, запускающий все компоненты .

**Полный путь к файлу:** `docker-compose.yml`

```yaml
version: '3'
services:
  opensearch:
    image: opensearchproject/opensearch:3.0.0
    container_name: opensearch
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-node1
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=admin
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - opensearch-data:/usr/share/opensearch/data
    ports:
      - 9200:9200
      - 9600:9600
    networks:
      - opensearch-net

  opensearch-dashboards:
    image: opensearchproject/opensearch-dashboards:3.0.0
    container_name: opensearch-dashboards
    ports:
      - 5601:5601
    environment:
      OPENSEARCH_HOSTS: '["http://opensearch:9200"]'
    networks:
      - opensearch-net

  data-prepper:
    container_name: data-prepper
    image: opensearchproject/data-prepper:2
    volumes:
      - ./log_pipeline.yaml:/usr/share/data-prepper/pipelines/log_pipeline.yaml
    ports:
      - 2021:2021
    networks:
      - opensearch-net

volumes:
  opensearch-data:

networks:
  opensearch-net:
```

Запуск: `docker-compose up -d`

---

## Часть 3: Создание шаблонов индексов для NGINX логов

Шаблоны индексов автоматизируют создание индексов с предопределёнными настройками. Для NGINX логов мы создадим шаблон, который будет определять количество шардов, реплик и маппинг полей .

### 3.1. Создание компонентного шаблона для настроек

Компонентные шаблоны — повторно используемые строительные блоки для настроек, маппингов и алиасов .

**REST API запрос:**

```bash
curl -X PUT "http://localhost:9200/_component_template/nginx_settings" \
  -H "Content-Type: application/json" \
  -d '{
  "template": {
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1,
      "index.refresh_interval": "30s",
      "index.codec": "best_compression"
    }
  },
  "version": 1,
  "_meta": {
    "description": "Base settings for NGINX log indexes"
  }
}'
```

### 3.2. Создание компонентного шаблона для маппингов

Маппинг определяет схему данных для полей NGINX лога .

```bash
curl -X PUT "http://localhost:9200/_component_template/nginx_mappings" \
  -H "Content-Type: application/json" \
  -d '{
  "template": {
    "mappings": {
      "_source": {
        "enabled": true
      },
      "properties": {
        "clientip": { "type": "ip" },
        "timestamp": { "type": "date" },
        "verb": { "type": "keyword" },
        "request": { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
        "httpversion": { "type": "keyword" },
        "response": { "type": "integer" },
        "bytes": { "type": "long" },
        "referrer": { "type": "keyword" },
        "agent": { "type": "text" },
        "log": { "type": "text" }
      }
    }
  },
  "version": 1,
  "_meta": {
    "description": "Mappings for NGINX access logs"
  }
}'
```

### 3.3. Создание индекс-шаблона для потока данных

Для потоков данных необходимо добавить объект `data_stream` в определение шаблона .

```bash
curl -X PUT "http://localhost:9200/_index_template/nginx-logs-template" \
  -H "Content-Type: application/json" \
  -d '{
  "index_patterns": ["nginx-logs-*"],
  "data_stream": {},
  "priority": 200,
  "composed_of": ["nginx_settings", "nginx_mappings"],
  "template": {
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1
    },
    "mappings": {
      "properties": {
        "timestamp": {
          "type": "date",
          "format": "dd/MMM/yyyy:HH:mm:ss Z"
        }
      }
    }
  }
}'
```

**Ключевые особенности:**
- `data_stream: {}` — указывает, что это шаблон для потока данных 
- `timestamp` — поле, которое будет использоваться для временной маркировки документов
- `priority: 200` — более высокий приоритет позволяет переопределить общие шаблоны
- `composed_of` — ссылается на созданные компонентные шаблоны 

---

## Часть 4: Управление потоками данных

### 4.1. Создание потока данных

Поток данных можно создать явно или автоматически при первой индексации .

**Явное создание:**

```bash
curl -X PUT "http://localhost:9200/_data_stream/nginx-logs-prod"
```

**Автоматическое создание (при индексации):**

```bash
curl -X POST "http://localhost:9200/nginx-logs-prod/_doc" \
  -H "Content-Type: application/json" \
  -d '{
  "clientip": "192.168.1.1",
  "timestamp": "10/Oct/2024:13:55:36 +0000",
  "verb": "GET",
  "request": "/index.html",
  "httpversion": "1.1",
  "response": 200,
  "bytes": 612,
  "referrer": "-",
  "agent": "Mozilla/5.0"
}'
```

### 4.2. Просмотр информации о потоке данных

```bash
# Информация о потоке
curl -X GET "http://localhost:9200/_data_stream/nginx-logs-prod"

# Статистика потока
curl -X GET "http://localhost:9200/_data_stream/nginx-logs-prod/_stats"
```

Ответ покажет список резервных индексов, их статус и используемый шаблон .

### 4.3. Массовая загрузка данных в поток

Для массовой загрузки используйте Bulk API :

```bash
curl -X POST "http://localhost:9200/nginx-logs-prod/_bulk?pretty&refresh" \
  -H "Content-Type: application/json" \
  --data-binary @bulk_logs.json
```

**Пример файла `bulk_logs.json`:**
```json
{"index":{}}
{"clientip":"192.168.1.1","timestamp":"10/Oct/2024:13:55:36 +0000","verb":"GET","request":"/index.html","httpversion":"1.1","response":200,"bytes":612,"referrer":"-","agent":"Mozilla/5.0"}
{"index":{}}
{"clientip":"192.168.1.2","timestamp":"10/Oct/2024:13:56:10 +0000","verb":"POST","request":"/api/login","httpversion":"1.1","response":401,"bytes":45,"referrer":"-","agent":"curl/7.68.0"}
```

### 4.4. Поиск в потоке данных

Поиск выполняется так же, как для обычного индекса, и охватывает все резервные индексы :

```bash
curl -X GET "http://localhost:9200/nginx-logs-prod/_search" \
  -H "Content-Type: application/json" \
  -d '{
  "query": {
    "range": {
      "timestamp": {
        "gte": "2024-10-01",
        "lt": "2024-11-01"
      }
    }
  },
  "aggs": {
    "top_urls": {
      "terms": { "field": "request.keyword", "size": 10 }
    }
  }
}'
```

### 4.5. Ролловер потока данных

Ролловер создаёт новый резервный индекс, который становится активным для записи .

```bash
curl -X POST "http://localhost:9200/nginx-logs-prod/_rollover"
```

Ответ покажет, какой индекс был заменён:
```json
{
  "old_index": ".ds-nginx-logs-prod-000001",
  "new_index": ".ds-nginx-logs-prod-000002",
  "rolled_over": true
}
```

### 4.6. Перенос данных (Reindex)

Операция Reindex используется для копирования документов из одного потока/индекса в другой :

```bash
curl -X POST "http://localhost:9200/_reindex" \
  -H "Content-Type: application/json" \
  -d '{
  "source": {
    "index": "nginx-logs-prod"
  },
  "dest": {
    "index": "nginx-logs-archive"
  },
  "query": {
    "range": {
      "timestamp": {
        "lt": "2024-01-01"
      }
    }
  }
}'
```

**Особенности Reindex для потоков данных:**
- Для целевого потока необходимо установить `op_type: create` (append-only)
- Reindex не копирует автоматически настройки и маппинги

### 4.7. Удаление потока данных

```bash
curl -X DELETE "http://localhost:9200/_data_stream/nginx-logs-archive"
```

---

## Часть 5: Интеграция с OpenSearch Dashboards

### 5.1. Создание индекс-паттерна

Для просмотра данных в OpenSearch Dashboards необходимо создать индекс-паттерн :

```bash
curl -X POST "http://localhost:5601/api/saved_objects/index-pattern/nginx-logs" \
  -H "osd-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "attributes": {
      "title": "nginx-logs-*",
      "timeFieldName": "timestamp"
    }
  }'
```

### 5.2. Импорт дашбордов

Для визуализации NGINX логов можно импортировать готовые дашборды :

```bash
curl -X POST "http://localhost:5601/api/saved_objects/_import?overwrite=true" \
  -H "osd-xsrf: true" \
  --form file=@nginx-dashboard.ndjson
```

---

## Часть 6: Расширенная обработка данных (опционально)

### 6.1. Фильтрация данных

Чтобы хранить только неудачные запросы (статус >= 400), можно использовать `drop_events` процессор в Data Prepper:

**Обновлённый файл `log_pipeline.yaml`:**
```yaml
log-pipeline:
  source:
    http:
      ssl: false
      port: 2021
  processor:
    - grok:
        match:
          log: [ "%{COMMONAPACHELOG}" ]
    - drop_events:
        drop_when: "/response < 400"
  sink:
    - opensearch:
        hosts: [ "http://opensearch:9200" ]
        insecure: true
        username: "admin"
        password: "admin"
        index: nginx_logs_errors
```

### 6.2. Обогащение данных

Извлечение параметров запроса из URL с использованием дополнительных процессоров:

```yaml
processor:
  - grok:
      match:
        log: [ "%{COMMONAPACHELOG}" ]
  - split_string:
      entries:
        - source: request
          delimiter: "?"
  - key_value:
      source: "/request/1"
      field_split_characters: "&"
      value_split_characters: "="
      destination: query_params
```
