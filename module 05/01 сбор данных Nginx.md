# Лабораторная работа: Сбор, подготовка и управление NGINX логами в OpenSearch 3.7


**Архитектура для NGINX логов:**
```
NGINX Logs → Fluent Bit → Data Prepper → OpenSearch → OpenSearch Dashboards
```


## Часть 2: Настройка сбора NGINX логов

### Шаг 1: Настройка Fluent Bit для сбора логов NGINX

Fluent Bit будет читать логи NGINX из файла `/var/log/nginx/access.log` и отправлять их в Data Prepper .

**Файл `fluent-bit.conf`:**
```
[INPUT]
  name                  tail
  refresh_interval      5
  path                  /var/log/nginx/access.log
  read_from_head        true

[OUTPUT]
  Name http
  Match *
  Host localhost
  Port 2021
  URI /log/ingest
  Format json
```

### Шаг 2: Настройка Data Prepper для обработки логов

Data Prepper получает логи от Fluent Bit и обрабатывает их с помощью Grok Processor для структурирования неструктурированных логов NGINX .

**Файл `log_pipeline.yaml`:**
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
  sink:
    - opensearch:
        hosts: [ "http://opensearch:9200" ]
        insecure: true
        username: "admin"
        password: "admin"
        index: nginx_logs
```

### Шаг 3: Запуск Data Prepper

Для запуска Data Prepper используйте Docker :

**Файл `docker-compose.yml`:**
```yaml
version: '3'
services:
  data-prepper:
    container_name: data-prepper
    image: opensearchproject/data-prepper:latest
    volumes:
      - ./log_pipeline.yaml:/usr/share/data-prepper/pipelines/log_pipeline.yaml
    ports:
      - 2021:2021
    networks:
      - opensearch-net
```

Запуск: `docker-compose up -d` 

---

## Часть 3: Создание шаблонов индексов для NGINX логов

Шаблоны индексов автоматизируют создание индексов с предопределёнными настройками. Для NGINX логов мы создадим шаблон, который будет определять количество шардов, реплик и маппинг полей .

### Шаг 1: Создание компонентного шаблона для настроек

Компонентные шаблоны — повторно используемые строительные блоки для настроек, маппингов и алиасов.

```json
PUT _component_template/nginx_settings
{
  "template": {
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1,
      "index.refresh_interval": "30s"
    }
  },
  "version": 1,
  "_meta": {
    "description": "Base settings for NGINX log indexes"
  }
}
```

### Шаг 2: Создание компонентного шаблона для маппингов

Маппинг определяет схему данных для полей NGINX лога.

```json
PUT _component_template/nginx_mappings
{
  "template": {
    "mappings": {
      "properties": {
        "clientip": { "type": "ip" },
        "timestamp": { "type": "date" },
        "verb": { "type": "keyword" },
        "request": { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
        "httpversion": { "type": "keyword" },
        "response": { "type": "integer" },
        "bytes": { "type": "long" },
        "referrer": { "type": "keyword" },
        "agent": { "type": "text" }
      }
    }
  }
}
```

### Шаг 3: Создание индекс-шаблона для потока данных

Для потоков данных необходимо добавить объект `data_stream` в определение шаблона .

```json
PUT _index_template/nginx-logs-template
{
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
}
```


## Часть 4: Управление потоками данных

### Шаг 1: Создание потока данных

Поток данных можно создать явно или автоматически при первой индексации .

**Явное создание:**
```json
PUT _data_stream/nginx-logs-prod
```

**Автоматическое создание (при индексации):**
```json
POST nginx-logs-prod/_doc
{
  "clientip": "192.168.1.1",
  "timestamp": "10/Oct/2024:13:55:36 +0000",
  "verb": "GET",
  "request": "/index.html",
  "httpversion": "1.1",
  "response": 200,
  "bytes": 612,
  "referrer": "-",
  "agent": "Mozilla/5.0"
}
```

### Шаг 2: Просмотр информации о потоке данных

```json
GET _data_stream/nginx-logs-prod
```

Ответ покажет список резервных индексов, их статус и используемый шаблон .

### Шаг 3: Поиск в потоке данных

Поиск выполняется так же, как для обычного индекса, и охватывает все резервные индексы:

```json
GET nginx-logs-prod/_search
{
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
}
```

### Шаг 4: Ролловер потока данных

Ролловер создаёт новый резервный индекс, который становится активным для записи .

```json
POST nginx-logs-prod/_rollover
```

Ответ покажет, какой индекс был заменён:
```json
{
  "old_index": ".ds-nginx-logs-prod-000001",
  "new_index": ".ds-nginx-logs-prod-000002",
  "rolled_over": true
}
```

**Автоматизация:** Ролловер можно автоматизировать с помощью политик Index State Management (ISM) при достижении определённого размера индекса или по истечении времени.

### Шаг 5: Перенос данных (Reindex)

Операция Reindex используется для копирования документов из одного потока/индекса в другой:

```json
POST /_reindex
{
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
}
```

## Часть 5: Расширенная обработка данных (опционально)

### Фильтрация данных

Чтобы хранить только неудачные запросы (статус >= 400), можно использовать `drop_events` процессор :

```yaml
processor:
  - grok:
      match:
        log: [ "%{COMMONAPACHELOG_DATATYPED}" ]
  - drop_events:
      drop_when: "/response < 400"
```

### Обогащение данных

Извлечение параметров запроса из URL :

```yaml
processor:
  - grok:
      match:
        message: [ "%{COMMONAPACHELOG_DATATYPED}" ]
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

