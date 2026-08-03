## 🧪 Лабораторная работа: Отказоустойчивый стек OpenSearch с балансировкой нагрузки и сбором Syslog

### Цель работы

Развернуть с помощью Docker Compose отказоустойчивый стек для сбора, обработки и визуализации логов: кластер OpenSearch из двух узлов, балансировщик нагрузки HAProxy для распределения запросов между ними, OpenSearch Dashboards для визуализации и Logstash для приёма и парсинга syslog-сообщений.

### Среда выполнения

*   **ОС хоста:** Debian 13 (или любая ОС с Docker).
*   **Инструменты:** Docker и Docker Compose.
*   **Стек:**
    *   **OpenSearch** (2 узла) — распределённое хранилище и поисковый движок.
    *   **HAProxy** — балансировщик нагрузки для OpenSearch и Dashboards.
    *   **OpenSearch Dashboards** — веб-интерфейс для визуализации.
    *   **Logstash** — конвейер обработки данных с парсингом syslog.

---

### Часть 1. Подготовка среды

#### Шаг 1. Настройка системы Debian 13

```bash
# Отключение swap для производительности OpenSearch
sudo swapoff -a

# Увеличение лимита карт памяти (критично для OpenSearch)
sudo sysctl -w vm.max_map_count=262144
```

Чтобы изменения сохранились после перезагрузки, добавьте `vm.max_map_count=262144` в `/etc/sysctl.conf`.

#### Шаг 2. Установка Docker и Docker Compose

Установите Docker и Docker Compose, следуя официальной документации для Debian.

#### Шаг 3. Создание структуры проекта

```bash
mkdir ~/opensearch-ha-lab
cd ~/opensearch-ha-lab
mkdir -p logstash/pipeline logstash/data haproxy
```

Структура проекта:
```
~/opensearch-ha-lab/
├── docker-compose.yml
├── haproxy/
│   └── haproxy.cfg
└── logstash/
    ├── pipeline/
    │   └── logstash.conf
    └── data/
        └── syslog.log
```

---

### Часть 2. Конфигурация

#### Шаг 1. Создание тестового syslog-файла

Создайте файл `~/opensearch-ha-lab/logstash/data/syslog.log` с примерами syslog-сообщений в формате RFC 3164:

```
Jun 17 10:00:01 debian13 systemd[1]: Started Session 123 of user root.
Jun 17 10:00:15 debian13 sshd[2456]: Accepted password for user from 192.168.1.100 port 54322 ssh2
Jun 17 10:01:20 debian13 sudo[3124]: user1 : TTY=pts/0 ; PWD=/home/user1 ; USER=root ; COMMAND=/usr/bin/apt update
Jun 17 10:02:45 debian13 kernel: [ 1245.678912] TCP: request_sock_TCP: Possible SYN flooding on port 80. Sending cookies.
Jun 17 10:03:10 debian13 postfix/smtpd[4123]: connect from mail.example.com[203.0.113.5]
Jun 17 10:03:15 debian13 postfix/smtpd[4123]: lost connection after UNKNOWN from mail.example.com[203.0.113.5]
```

#### Шаг 2. Создание конфигурации HAProxy

Создайте файл `~/opensearch-ha-lab/haproxy/haproxy.cfg`:

```haproxy
global
    log stdout format raw local0
    maxconn 4096
    stats timeout 30s
    user haproxy
    group haproxy

defaults
    log global
    mode http
    option httplog
    option dontlognull
    retries 3
    timeout connect 5000
    timeout client 50000
    timeout server 50000

# Статистика HAProxy (доступна по адресу http://<IP>:8400/stats)
listen stats
    bind :8400
    mode http
    stats enable
    stats uri /stats
    stats refresh 10s
    stats admin if LOCALHOST

# Балансировка для OpenSearch API (порт 9200)
frontend opensearch_frontend
    bind :9200
    mode http
    default_backend opensearch_backend

backend opensearch_backend
    mode http
    balance roundrobin
    option httpchk GET /_cluster/health HTTP/1.0
    http-check expect string "green" or "yellow"
    server opensearch-node1 opensearch-node1:9200 check inter 5s rise 2 fall 3
    server opensearch-node2 opensearch-node2:9200 check inter 5s rise 2 fall 3

# Балансировка для OpenSearch Dashboards (порт 5601)
frontend dashboards_frontend
    bind :5601
    mode http
    default_backend dashboards_backend

backend dashboards_backend
    mode http
    balance roundrobin
    option httpchk GET /api/status HTTP/1.0
    http-check expect status 200
    server dashboards opensearch-dashboards:5601 check inter 5s rise 2 fall 3
```

**Что здесь настроено:**
- **Статистика HAProxy** доступна на порту `8400/stats`.
- **Балансировка OpenSearch** (порт 9200) с проверкой здоровья через `/_cluster/health`.
- **Балансировка Dashboards** (порт 5601) с проверкой через `/api/status`.
- Алгоритм `roundrobin` для равномерного распределения запросов.

#### Шаг 3. Создание конфигурации Logstash

Создайте файл `~/opensearch-ha-lab/logstash/pipeline/logstash.conf` для парсинга syslog:

```ruby
input {
  file {
    path => "/usr/share/logstash/data/syslog.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    codec => plain { charset => "UTF-8" }
  }
}

filter {
  # Парсинг syslog-сообщений с использованием встроенного шаблона SYSLOGBASE
  grok {
    match => { 
      "message" => "%{SYSLOGBASE} %{GREEDYDATA:syslog_message}" 
    }
  }

  # Преобразование timestamp из syslog в @timestamp Logstash
  date {
    match => [ "timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
    target => "@timestamp"
    remove_field => ["timestamp"]
  }

  # Добавление поля с типом лога
  mutate {
    add_field => { "log_type" => "syslog" }
  }

  # Удаление служебного поля "message" после парсинга
  mutate { remove_field => ["message"] }
}

output {
  # Отправка в OpenSearch через HAProxy (единая точка входа)
  opensearch {
    hosts       => ["http://haproxy:9200"]
    user        => "admin"
    password    => "MyStrongPassword123!"
    index       => "syslog-logs-%{+YYYY.MM.dd}"
    ssl_certificate_verification => false
  }
  # Вывод в консоль для отладки
  stdout { codec => rubydebug }
}
```

**Ключевые моменты:**
- Logstash отправляет данные **не напрямую** в OpenSearch, а через HAProxy (`haproxy:9200`), что обеспечивает балансировку нагрузки.
- Фильтр `grok` с шаблоном `%{SYSLOGBASE}` парсит стандартные syslog-поля: `timestamp`, `host`, `program`, `pid`.
- Оставшаяся часть сообщения сохраняется в поле `syslog_message`.

#### Шаг 4. Создание docker-compose.yml

Создайте файл `~/opensearch-ha-lab/docker-compose.yml`:

```yaml
version: '3'

services:
  # ---- Узел OpenSearch 1 ----
  opensearch-node1:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-node1
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-node1
      - discovery.seed_hosts=opensearch-node1,opensearch-node2
      - cluster.initial_cluster_manager_nodes=opensearch-node1,opensearch-node2
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m"
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=MyStrongPassword123!
      - "OPENSEARCH_NETWORK_HOST=0.0.0.0"          # Слушаем все интерфейсы
      - "OPENSEARCH_NODE_NAME=opensearch-node1"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - opensearch-data1:/usr/share/opensearch/data
    ports:
      - "9201:9200"      # Порт API для node1
      - "9601:9600"      # Performance Analyzer для node1
    networks:
      opensearch-net:
        ipv4_address: 172.20.0.10   # Статический IP для node1

  # ---- Узел OpenSearch 2 ----
  opensearch-node2:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-node2
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-node2
      - discovery.seed_hosts=opensearch-node1,opensearch-node2
      - cluster.initial_cluster_manager_nodes=opensearch-node1,opensearch-node2
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m"
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=MyStrongPassword123!
      - "OPENSEARCH_NETWORK_HOST=0.0.0.0"
      - "OPENSEARCH_NODE_NAME=opensearch-node2"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - opensearch-data2:/usr/share/opensearch/data
    ports:
      - "9202:9200"      # Порт API для node2
      - "9602:9600"      # Performance Analyzer для node2
    networks:
      opensearch-net:
        ipv4_address: 172.20.0.11   # Статический IP для node2

  # ---- OpenSearch Dashboards ----
  opensearch-dashboards:
    image: opensearchproject/opensearch-dashboards:latest
    container_name: opensearch-dashboards
    environment:
      - OPENSEARCH_HOSTS=["http://opensearch-node1:9200","http://opensearch-node2:9200"]
    ports:
      - "5602:5601"      # Прямой доступ к Dashboards (не через HAProxy)
    networks:
      opensearch-net:
        ipv4_address: 172.20.0.20

  # ---- HAProxy ----
  haproxy:
    image: haproxy:latest
    container_name: haproxy
    volumes:
      - ./haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    ports:
      - "9200:9200"      # Балансировка OpenSearch API
      - "5601:5601"      # Балансировка Dashboards
      - "8400:8400"      # Статистика HAProxy
    networks:
      opensearch-net:
        ipv4_address: 172.20.0.30
    depends_on:
      - opensearch-node1
      - opensearch-node2
      - opensearch-dashboards

  # ---- Logstash ----
  logstash:
    image: opensearchproject/logstash-oss-with-opensearch-output-plugin:latest
    container_name: logstash
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
      - ./logstash/data:/usr/share/logstash/data
    networks:
      opensearch-net:
        ipv4_address: 172.20.0.40
    depends_on:
      - haproxy

networks:
  opensearch-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16   # Кастомная подсеть для статических IP

volumes:
  opensearch-data1:
  opensearch-data2:
```

**Ключевые особенности конфигурации:**

| Элемент | Описание |
|---------|----------|
| **Две ноды OpenSearch** | `opensearch-node1` и `opensearch-node2` образуют кластер. |
| **Статические IP** | Каждой ноде назначен свой IP в подсети `172.20.0.0/16`. |
| **Разные порты на хосте** | `9201` и `9202` для прямого доступа к каждой ноде (для отладки). |
| **HAProxy** | Балансирует запросы между нодами на портах `9200` (API) и `5601` (Dashboards). |
| **Logstash** | Отправляет данные на `haproxy:9200`, а не напрямую в OpenSearch. |

---

### Часть 3. Запуск и проверка

#### Шаг 1. Запуск стека

```bash
cd ~/opensearch-ha-lab
docker compose up -d
```

#### Шаг 2. Проверка статуса контейнеров

```bash
docker compose ps
```

Ожидаемый вывод: все 5 контейнеров (`opensearch-node1`, `opensearch-node2`, `opensearch-dashboards`, `haproxy`, `logstash`) в состоянии `Up`.

#### Шаг 3. Проверка кластера OpenSearch

**Через HAProxy (порт 9200):**
```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cluster/health?pretty"
```

**На каждой ноде напрямую:**
```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9201/_cat/nodes?v"
curl -k -u admin:MyStrongPassword123! "https://localhost:9202/_cat/nodes?v"
```

В выводе должно быть две строки — обе ноды в кластере.

#### Шаг 4. Проверка балансировки HAProxy

Откройте в браузере `http://<IP_сервера>:8400/stats`. Вы увидите статистику HAProxy с информацией о состоянии бэкендов `opensearch_backend` и `dashboards_backend`.

#### Шаг 5. Проверка Logstash

```bash
docker logs logstash --tail 30
```

В выводе должны быть сообщения об успешной отправке событий в OpenSearch через HAProxy.

#### Шаг 6. Проверка индексов в OpenSearch

```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cat/indices?v"
```

Должен появиться индекс `syslog-logs-YYYY.MM.dd`.

#### Шаг 7. Открытие OpenSearch Dashboards

Откройте в браузере `http://<IP_сервера>:5601` (через HAProxy) или `http://<IP_сервера>:5602` (напрямую). Используйте логин `admin` и пароль `MyStrongPassword123!`.

В разделе **Discover** выберите индекс `syslog-logs-*` и убедитесь, что syslog-сообщения распарсены по полям: `host`, `program`, `pid`, `syslog_message`.

---

### Часть 4. Задания для самостоятельного выполнения

1.  **Симуляция отказа ноды:** Остановите одну из нод OpenSearch (`docker stop opensearch-node1`). Проверьте, что HAProxy автоматически перенаправляет трафик на работающую ноду. Восстановите ноду и убедитесь, что она снова включилась в ротацию.
2.  **Добавление нового syslog-сообщения:** Добавьте новую строку в файл `logstash/data/syslog.log` (не перезапуская Logstash). Logstash должен автоматически подхватить изменения и отправить новое событие.
3.  **Модификация Grok-паттерна:** Изучите документацию по Grok-паттернам и добавьте в `logstash.conf` извлечение IP-адреса из сообщения, если он там присутствует.
4.  **Настройка оповещений:** Настройте в OpenSearch Dashboards монитор для отслеживания ошибок в syslog-логах (например, появление слова "error" или "failed").
5.  **Перенастройка Logstash на приём по сети:** Измените `input` в `logstash.conf` с `file` на `syslog` (порт 514) для приёма syslog-сообщений по UDP/TCP от других серверов.

---

### Часть 5. Дополнительно: Проверка отказоустойчивости

#### Тест 1. Отказ одной ноды OpenSearch

```bash
# Остановка node1
docker stop opensearch-node1

# Проверка состояния кластера (статус должен стать yellow, так как реплики могут быть не назначены)
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cluster/health?pretty"

# Отправка тестового запроса через HAProxy
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cat/nodes?v"

# Запуск node1 обратно
docker start opensearch-node1
```

#### Тест 2. Отказ HAProxy (симуляция)

```bash
# Остановка HAProxy
docker stop haproxy

# Попытка подключиться к OpenSearch через HAProxy (порт 9200) - должна быть ошибка
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cluster/health?pretty"

# Прямое подключение к ноде (порт 9201) - должно работать
curl -k -u admin:MyStrongPassword123! "https://localhost:9201/_cluster/health?pretty"

# Запуск HAProxy обратно
docker start haproxy
```

---

### Возможные проблемы и решения

| Проблема | Возможное решение |
| :--- | :--- |
| **OpenSearch не запускается** | Проверьте, что `vm.max_map_count=262144` установлен. Убедитесь, что пароль соответствует требованиям (минимум 8 символов, заглавная, строчная буквы, цифра, спецсимвол). |
| **Ноды не видят друг друга** | Проверьте, что `discovery.seed_hosts` и `cluster.initial_cluster_manager_nodes` содержат правильные имена узлов. Убедитесь, что обе ноды в одной сети Docker. |
| **HAProxy не подключается к OpenSearch** | Проверьте, что HAProxy использует имена сервисов (`opensearch-node1:9200`), которые резолвятся внутри сети Docker. Проверьте логи HAProxy: `docker logs haproxy`. |
| **Logstash не отправляет данные** | Убедитесь, что Logstash может достичь HAProxy (`haproxy:9200`). Проверьте логи Logstash на наличие ошибок подключения. |
| **Grok не парсит syslog** | Используйте `stdout { codec => rubydebug }` для отладки. Проверьте, что формат сообщений соответствует шаблону `SYSLOGBASE`. |

---

### Заключение

В этой лабораторной работе вы:

1.  Развернули **кластер OpenSearch из двух узлов** с назначением статических IP-адресов.
2.  Настроили **HAProxy** для балансировки нагрузки между нодами и обеспечения отказоустойчивости.
3.  Настроили **Logstash** для чтения и парсинга syslog-файлов с использованием Grok.
4.  Протестировали отказоустойчивость кластера при выходе из строя одной из нод.
