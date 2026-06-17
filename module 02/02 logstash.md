

## 🧪 Лабораторная работа: Централизованный сбор логов с Logstash и OpenSearch


### Среда выполнения

*   **ОС хоста:** Debian 13 
*   **Контейнеризация:** Docker и Docker Compose .
*   **Стек:**
    *   **OpenSearch** и **OpenSearch Dashboards** (образы `bitnami/opensearch:latest` и `bitnami/opensearch-dashboards:latest`) .
    *   **Logstash** (образ `opensearchproject/logstash-oss-with-opensearch-output-plugin:latest`) .


### Создание структуры проекта

Создайте следующую структуру каталогов:
```
~/opensearch-lab/
├── docker-compose.yml
├── logstash/
│   ├── pipeline/
│   │   └── logstash.conf
│   └── data/
│       └── access.log
└── dashboards/
```

1.  **Создайте тестовый лог-файл** `~/opensearch-lab/logstash/data/access.log` с содержимым (пример из формата Common Log Format):
    ```
    192.168.1.1 - admin [10/Oct/2023:13:55:36 +0000] "GET /index.html HTTP/1.1" 200 2326
    10.0.0.2 - - [10/Oct/2023:13:56:40 +0000] "POST /api/data HTTP/1.1" 404 512
    ```

### Конфигурация

#### `docker-compose.yml`

Файл определяет три сервиса, объединённые общей сетью `opensearch-net` .

```yaml
version: '3'
services:
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
    # Для отладки можно раскомментировать и смотреть логи в консоли
    # command: -f /usr/share/logstash/pipeline/logstash.conf --config.reload.automatic

volumes:
  opensearch-data:

networks:
  opensearch-net:
    driver: bridge
```

#### Конфигурация Logstash (`logstash/pipeline/logstash.conf`)

Этот пайплайн читает лог-файл, парсит его с помощью Grok по шаблону Common Apache Log, преобразует типы данных и отправляет результат в OpenSearch .

```ruby
input {
  file {
    path => "/usr/share/logstash/data/access.log"
    start_position => "beginning" # Начать чтение с начала файла
    sincedb_path => "/dev/null"   # Не сохранять позицию (для тестов)
    codec => plain { charset => "UTF-8" }
  }
}

filter {
  # Парсинг по шаблону Combined Log Format (включает реферер и агент)
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

  # Если в логе нет размера (bytes = "-"), заменить на 0
  if [bytes] == "-" {
    mutate { replace => { "bytes" => "0" } }
  }

  # Удаление служебного поля "message" после парсинга
  mutate { remove_field => ["message"] }
}

output {
  opensearch {
    hosts       => ["http://opensearch-node1:9200"]
    user        => "admin"
    password    => "MyStrongPassword123!" # Должен совпадать с паролем в OpenSearch
    index       => "web-logs-%{+YYYY.MM.dd}"
    ssl_certificate_verification => false # Отключаем для локальной среды (http)
  }
  # Вывод в консоль для отладки
  stdout { codec => rubydebug }
}
```

### Запуск лабораторной работы

1.  **Запустите стек** из директории `~/opensearch-lab`:
    ```bash
    docker-compose up -d
    ```
    Подождите 30-60 секунд, пока все сервисы запустятся .

2.  **Проверьте статус контейнеров:**
    ```bash
    docker-compose ps
    ```

3.  **Проверьте доступность OpenSearch:**
    ```bash
    curl -k -u admin:MyStrongPassword123! https://localhost:9200
    ```

4.  **Проверьте, что Logstash обработал логи:**
    ```bash
    docker logs logstash
    ```
    В выводе должны быть сообщения об успешной отправке событий в OpenSearch.

5.  **Проверьте индексы в OpenSearch:** После обработки логов, должен создаться индекс, начинающийся с `web-logs-`:
    ```bash
    curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cat/indices?v"
    ```

6.  **Проверьте данные в индексе:**
    ```bash
    curl -k -u admin:MyStrongPassword123! "https://localhost:9200/web-logs-*/_search?pretty"
    ```

7.  **Откройте OpenSearch Dashboards** в браузере по адресу `http://<IP_вашего_сервера>:5601`. Используйте логин `admin` и пароль `MyStrongPassword123!`. В разделе "Discover" выберите индекс `web-logs-*`, чтобы увидеть и проанализировать распарсенные логи .

*   **Logstash не видит файл лога:** Проверьте, что файл смонтирован в контейнер (`docker exec -it logstash ls -la /usr/share/logstash/data/`). Проверьте, что путь в конфигурации `logstash.conf` совпадает с путём внутри контейнера.
*   **Logstash не подключается к OpenSearch:** Проверьте имя хоста (`opensearch-node1`) — оно должно разрешаться внутри сети Docker. Убедитесь, что `depends_on` в `docker-compose.yml` не гарантирует готовность сервиса, поэтому может потребоваться небольшое ожидание.
*   **Ошибка парсинга Grok:** Используйте `stdout { codec => rubydebug }` в выводе Logstash для отладки. Проверьте соответствие паттерна содержимому лога.
