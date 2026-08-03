
## 🧪 Лабораторная работа: Настройка стека OpenSearch, Logstash и OpenSearch Dashboards

### Цель работы

Развернуть с помощью Docker Compose полноценный стек для сбора, обработки и визуализации логов: OpenSearch (хранилище и поисковый движок), Logstash (конвейер обработки данных) и OpenSearch Dashboards (веб-интерфейс для визуализации). В рамках работы будет настроен пайплайн Logstash для чтения тестового лог-файла, его парсинга с помощью Grok-фильтра и отправки структурированных данных в OpenSearch.

### Среда выполнения

*   **ОС хоста:** Debian 13 (или любая ОС с Docker).
*   **Инструменты:** Docker и Docker Compose.
*   **Стек:**
    *   **OpenSearch** (образ `opensearchproject/opensearch:latest`).
    *   **OpenSearch Dashboards** (образ `opensearchproject/opensearch-dashboards:latest`).
    *   **Logstash** (образ `opensearchproject/logstash-oss-with-opensearch-output-plugin:latest`).

---

### Часть 1. Подготовка среды

#### Шаг 1. Установка Docker и Docker Compose

Установите Docker и Docker Compose, следуя официальной документации для вашей операционной системы.

#### Шаг 2. Создание структуры проекта

Создайте рабочую директорию и необходимые поддиректории:

```bash
mkdir ~/opensearch-logstash-lab
cd ~/opensearch-logstash-lab
mkdir -p logstash/pipeline logstash/data
```

Структура проекта будет выглядеть так:
```
~/opensearch-logstash-lab/
├── docker-compose.yml
└── logstash/
    ├── pipeline/
    │   └── logstash.conf
    └── data/
        └── access.log
```

---

### Часть 2. Конфигурация

#### Шаг 1. Создание тестового лог-файла

Создайте файл `~/opensearch-logstash-lab/logstash/data/access.log` и добавьте в него несколько строк в формате Combined Log Format (пример из датасета):

```
199.227.6.10 - - [01/Jul/1995:21:57:07 -0400] "GET /images/launch-logo.gif HTTP/1.0" 200 3092
205.212.175.133 - - [01/Jul/1995:22:00:27 -0400] "GET /shuttle/countdown/ HTTP/1.0" 200 3985
209.190.233.27 - - [01/Jul/1995:22:01:09 -0400] "GET /images/NASA-logosmall.gif HTTP/1.0" 200 786
```

#### Шаг 2. Создание конфигурации Logstash

Создайте файл `~/opensearch-logstash-lab/logstash/pipeline/logstash.conf`. Этот пайплайн будет читать файл, парсить логи с помощью Grok и отправлять данные в OpenSearch.

```ruby
input {
  file {
    path => "/usr/share/logstash/data/access.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    codec => plain { charset => "UTF-8" }
  }
}

filter {
  # Парсинг логов в формате Combined Log Format
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
  }

  # Преобразование timestamp из лога в @timestamp Logstash
  date {
    match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
    target => "@timestamp"
    remove_field => ["timestamp"]
  }

  # Конвертация полей в числа
  mutate {
    convert => {
      "response" => "integer"
      "bytes"    => "integer"
    }
  }

  # Удаление служебного поля "message" после парсинга
  mutate { remove_field => ["message"] }
}

output {
  opensearch {
    hosts       => ["http://opensearch-node1:9200"]
    user        => "admin"
    password    => "MyStrongPassword123!"
    index       => "access-logs-%{+YYYY.MM.dd}"
    ssl_certificate_verification => false
  }
  # Вывод в консоль для отладки
  stdout { codec => rubydebug }
}
```

#### Шаг 3. Создание docker-compose.yml

Создайте файл `~/opensearch-logstash-lab/docker-compose.yml`.

**Важно:** Начиная с OpenSearch 2.12, для установки демо-конфигурации требуется задать пароль администратора через переменную `OPENSEARCH_INITIAL_ADMIN_PASSWORD`. Пароль должен быть не менее 8 символов и содержать заглавную, строчную буквы, цифру и спецсимвол.

```yaml
version: '3'
services:
  opensearch-node1:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-node1
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-node1
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m"
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=MyStrongPassword123!
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - opensearch-data:/usr/share/opensearch/data
    ports:
      - "9200:9200"
      - "9600:9600"
    networks:
      - opensearch-net

  opensearch-dashboards:
    image: opensearchproject/opensearch-dashboards:latest
    container_name: opensearch-dashboards
    ports:
      - "5601:5601"
    environment:
      - OPENSEARCH_HOSTS=["http://opensearch-node1:9200"]
    networks:
      - opensearch-net
    depends_on:
      - opensearch-node1

  logstash:
    image: opensearchproject/logstash-oss-with-opensearch-output-plugin:latest
    container_name: logstash
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
      - ./logstash/data:/usr/share/logstash/data
    networks:
      - opensearch-net
    depends_on:
      - opensearch-node1

volumes:
  opensearch-data:

networks:
  opensearch-net:
    driver: bridge
```

---

### Часть 3. Запуск и проверка

#### Шаг 1. Запуск стека

Из директории `~/opensearch-logstash-lab` выполните:

```bash
docker compose up -d
```

#### Шаг 2. Проверка статуса контейнеров

Убедитесь, что все три контейнера запущены:

```bash
docker compose ps
```

#### Шаг 3. Проверка доступности OpenSearch

```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200"
```

#### Шаг 4. Просмотр логов Logstash

Убедитесь, что Logstash обработал лог-файл:

```bash
docker logs logstash --tail 20
```

В выводе должны быть сообщения об успешной отправке событий в OpenSearch.

#### Шаг 5. Проверка индексов в OpenSearch

После обработки логов должен создаться индекс `access-logs-YYYY.MM.dd`:

```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cat/indices?v"
```

#### Шаг 6. Проверка данных в индексе

```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/access-logs-*/_search?pretty"
```

#### Шаг 7. Открытие OpenSearch Dashboards

Откройте в браузере `http://<IP_вашего_сервера>:5601`. Используйте логин `admin` и пароль `MyStrongPassword123!`. В разделе **Discover** выберите индекс `access-logs-*`, чтобы увидеть распарсенные логи.

---

| **Logstash не видит файл лога** | Проверьте, что файл смонтирован в контейнер: `docker exec -it logstash ls -la /usr/share/logstash/data/`. Убедитесь, что путь в `logstash.conf` совпадает с путём внутри контейнера. |
| **Logstash не подключается к OpenSearch** | Проверьте имя хоста (`opensearch-node1`) — оно должно разрешаться внутри сети Docker. Убедитесь, что `depends_on` не гарантирует готовность сервиса; может потребоваться перезапуск Logstash. |
| **Ошибка парсинга Grok** | Используйте `stdout { codec => rubydebug }` в выводе Logstash для отладки. Проверьте соответствие паттерна содержимому лога. |
