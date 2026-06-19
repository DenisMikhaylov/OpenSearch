# Лабораторная работа: Полный цикл сбора и анализа Syslog в OpenSearch 3.7

## Часть 1: Обзор методов сбора логов

### 1.1. Архитектура сбора системных логов

Для сбора syslog с множества серверов и устройств обычно применяется следующая архитектура:
- **rsyslog** — системный демон, собирающий логи с источников и пересылающий их на центральный сервер
- **Data Prepper** — обрабатывает и структурирует логи для OpenSearch
- **OpenSearch** — хранилище и поисковый движок
- **OpenSearch Dashboards** — визуализация и аналитика

### 1.2. Сравнение методов сбора

| Метод | Сложность | Масштабируемость | Особенности |
|-------|-----------|------------------|-------------|
| rsyslog → Data Prepper | Средняя | Высокая | Нативная поддержка syslog, проверенная временем |
| Fluent Bit → Data Prepper | Средняя | Высокая | Лёгкий, подходит для контейнеров  |
| Filebeat → Logstash → OpenSearch | Высокая | Высокая | Требует лицензирования для некоторых версий  |

## Часть 2: Установка и настройка rsyslog

### 2.1. Установка rsyslog (на всех серверах-источниках)

```bash
# Обновление пакетного менеджера
sudo apt update

# Установка rsyslog
sudo apt install rsyslog -y

# Проверка статуса
sudo systemctl status rsyslog
```

### 2.2. Настройка rsyslog для отправки логов на центральный сервер

**Полный путь к файлу:** `/etc/rsyslog.conf` или `/etc/rsyslog.d/50-forward.conf`

Создайте файл конфигурации для пересылки логов:

```bash
sudo nano /etc/rsyslog.d/50-forward.conf
```

**Содержимое файла:**
```
# Пересылка всех логов на центральный сервер Data Prepper
*.* @192.168.1.100:5140
```

**Перезапуск rsyslog:**
```bash
sudo systemctl restart rsyslog
```

### 2.3. Расширенная конфигурация rsyslog (опционально)

Для более гибкой настройки можно использовать шаблоны и фильтры:

```bash
sudo nano /etc/rsyslog.d/50-forward.conf
```

```
# Шаблон формата сообщения для лучшей совместимости с OpenSearch
$template OpenSearchFormat,"<%pri%>%timestamp% %hostname% %programname% %msg%\n"

# Пересылка только критических логов
*.crit @192.168.1.100:5140;OpenSearchFormat

# Пересылка логов определённых программ
:programname, isequal, "nginx" @192.168.1.100:5140
```


## Часть 3: Настройка Data Prepper для приёма Syslog

### 3.1. Установка Data Prepper

**Запуск через Docker:**

```bash
# Скачивание образа Data Prepper
docker pull opensearchproject/data-prepper:2

# Создание каталога для конфигураций
mkdir -p ~/data-prepper
cd ~/data-prepper
```

### 3.2. Создание конфигурации Data Prepper для syslog

**Полный путь к файлу:** `~/data-prepper/log_pipeline.yaml`

```yaml
syslog-pipeline:
  workers: 4
  delay: 100
  source:
    http:
      ssl: false
      port: 5140
  buffer:
    bounded_blocking:
      buffer_size: 2048
      batch_size: 256
  processor:
    - grok:
        match:
          # Сначала пытаемся распарсить как стандартный syslog
          message: [ "%{SYSLOGBASE}" ]
        break_on_match: false
  sink:
    - opensearch:
        hosts: [ "http://opensearch:9200" ]
        insecure: true
        username: "admin"
        password: "admin"
        index: syslog_logs
```

**Пояснение компонентов :**
- **HTTP Source** — принимает данные от rsyslog на порту 5140
- **Grok Processor** — парсит логи по шаблону `SYSLOGBASE`, извлекая поля: timestamp, hostname, program name, message
- **OpenSearch Sink** — отправляет обработанные данные в кластер OpenSearch

### 3.3. Альтернативная конфигурация — приём через UDP

Data Prepper также поддерживает UDP Source для syslog:

```yaml
syslog-pipeline:
  source:
    udp:
      port: 5140
      ssl: false
  processor:
    - grok:
        match:
          message: [ "%{SYSLOGBASE}" ]
  sink:
    - opensearch:
        hosts: [ "http://opensearch:9200" ]
        username: "admin"
        password: "admin"
        index: syslog_logs
```

### 3.4. Запуск Data Prepper

```bash
docker run --name data-prepper \
  -v $(pwd)/log_pipeline.yaml:/usr/share/data-prepper/pipelines/log_pipeline.yaml \
  --network "opensearch-net" \
  -p 5140:5140 \
  opensearchproject/data-prepper:2
```

---

## Часть 4: Запуск OpenSearch через Docker Compose

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
      - 5140:5140
    networks:
      - opensearch-net

volumes:
  opensearch-data:

networks:
  opensearch-net:
```

Запуск: `docker-compose up -d`

---

## Часть 5: Создание шаблонов индексов для Syslog

### 5.1. Создание компонентного шаблона для настроек

**REST API запрос:**

```bash
curl -X PUT "http://localhost:9200/_component_template/syslog_settings" \
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
    "description": "Base settings for syslog indexes"
  }
}'
```

### 5.2. Создание компонентного шаблона для маппингов

Маппинг определяет схему данных для полей syslog.

```bash
curl -X PUT "http://localhost:9200/_component_template/syslog_mappings" \
  -H "Content-Type: application/json" \
  -d '{
  "template": {
    "mappings": {
      "_source": {
        "enabled": true
      },
      "properties": {
        "timestamp": { "type": "date" },
        "hostname": { "type": "keyword" },
        "program": { "type": "keyword" },
        "priority": { "type": "integer" },
        "severity": { "type": "keyword" },
        "facility": { "type": "keyword" },
        "message": { "type": "text" },
        "log": { "type": "text" }
      }
    }
  },
  "version": 1,
  "_meta": {
    "description": "Mappings for syslog messages"
  }
}'
```

### 5.3. Создание индекс-шаблона для потока данных

```bash
curl -X PUT "http://localhost:9200/_index_template/syslog-template" \
  -H "Content-Type: application/json" \
  -d '{
  "index_patterns": ["syslog-*"],
  "data_stream": {},
  "priority": 200,
  "composed_of": ["syslog_settings", "syslog_mappings"],
  "template": {
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1
    },
    "mappings": {
      "properties": {
        "timestamp": {
          "type": "date"
        }
      }
    }
  }
}'
```



## Часть 6: Управление потоками данных для Syslog

### 6.1. Создание потока данных

```bash
curl -X PUT "http://localhost:9200/_data_stream/syslog-prod"
```

### 6.2. Ввод тестовых данных в поток

Отправка тестового syslog-сообщения напрямую через API:

```bash
curl -X POST "http://localhost:9200/syslog-prod/_doc" \
  -H "Content-Type: application/json" \
  -d '{
  "timestamp": "2024-10-10T13:55:36Z",
  "hostname": "webserver-01",
  "program": "sshd",
  "priority": 6,
  "severity": "info",
  "facility": "auth",
  "message": "Accepted password for user admin from 192.168.1.100 port 22 ssh2"
}'
```

### 6.3. Генерация тестовых syslog-сообщений

Для тестирования можно использовать команду `logger`:

```bash
logger "Test syslog message from webserver-01"
logger -p auth.info "User login attempt from 192.168.1.100"
logger -p local0.err "Disk space warning on /dev/sda1"
```

### 6.4. Поиск в потоке данных

Поиск системных событий:

```bash
curl -X GET "http://localhost:9200/syslog-prod/_search" \
  -H "Content-Type: application/json" \
  -d '{
  "query": {
    "bool": {
      "must": [
        { "term": { "program": "sshd" } },
        { "range": { "timestamp": { "gte": "now-1h" } } }
      ]
    }
  },
  "aggs": {
    "failed_logins": {
      "filter": {
        "match": { "message": "Failed password" }
      }
    }
  }
}'
```

### 6.5. Ролловер потока данных

```bash
curl -X POST "http://localhost:9200/syslog-prod/_rollover"
```

### 6.6. Перенос данных (Reindex) для архивации

```bash
curl -X POST "http://localhost:9200/_reindex" \
  -H "Content-Type: application/json" \
  -d '{
  "source": {
    "index": "syslog-prod"
  },
  "dest": {
    "index": "syslog-archive"
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

---

## Часть 7: Интеграция с OpenSearch Dashboards

### 7.1. Создание индекс-паттерна для syslog

```bash
curl -X POST "http://localhost:5601/api/saved_objects/index-pattern/syslog" \
  -H "osd-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "attributes": {
      "title": "syslog-*",
      "timeFieldName": "timestamp"
    }
  }'
```

### 7.2. Создание дашборда для мониторинга

В OpenSearch Dashboards можно создать визуализации для:
- Количество сообщений по хостам (hostname)
- Распределение по severity (критичность)
- Топ программ, генерирующих логи
- График событий во времени
