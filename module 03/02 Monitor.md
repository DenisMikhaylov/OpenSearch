

## 🧪 Лабораторная работа: Кластер OpenSearch с разделением ролей (Cluster Manager, Data, Search)

### Цель работы

Развернуть с помощью Docker Compose многоролевой кластер OpenSearch, в котором каждый узел выполняет строго определённую функцию. Это позволит изолировать нагрузки, повысить производительность и обеспечить отказоустойчивость за счёт дублирования критичных ролей.

### Теоретическое введение

В OpenSearch каждый узел может выполнять одну или несколько ролей. Для production-сред рекомендуется выделять специализированные узлы для каждой роли.

| Роль | Описание | Рекомендации для production |
|------|----------|------------------------------|
| **Cluster Manager** | Управляет кластером: создание/удаление индексов, отслеживание узлов, выделение шардов. | **Три выделенных узла** в разных зонах для обеспечения кворума. |
| **Data** | Хранит данные и выполняет операции индексации, поиска и агрегации на локальных шардах. | Балансируйте узлы между зонами, используйте Storage и RAM-heavy узлы. |
| **Search** | Выделенные узлы, обслуживающие только search replica shards, отделяя поисковые нагрузки от индексации. | Используйте для выделенных memory-optimized инстансов. |
| **Coordinating** | Принимает клиентские запросы, направляет их на шарды, агрегирует результаты. | Пара dedicated узлов для search-heavy workloads. |

По умолчанию каждый узел является cluster-manager-eligible, data, ingest и coordinating. В этой лабораторной мы явно назначим роли через параметр `node.roles`.

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
mkdir ~/opensearch-roles-lab
cd ~/opensearch-roles-lab
```

---

### Часть 2. Архитектура кластера

В этой лабораторной мы развернём кластер со следующей архитектурой:

```
┌─────────────────────────────────────────────────────────────────┐
│                      OpenSearch Cluster                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ Cluster Mgr  │  │ Cluster Mgr  │  │ Cluster Mgr  │         │
│  │   (cm1)      │  │   (cm2)      │  │   (cm3)      │         │
│  │ node.roles:  │  │ node.roles:  │  │ node.roles:  │         │
│  │ [cluster_    │  │ [cluster_    │  │ [cluster_    │         │
│  │  manager]    │  │  manager]    │  │  manager]    │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│         │                 │                 │                  │
│         └─────────────────┼─────────────────┘                  │
│                           │                                    │
│         ┌─────────────────┼─────────────────┐                  │
│         ▼                 ▼                 ▼                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │    Data      │  │    Data      │  │   Search     │         │
│  │   (d1)       │  │   (d2)       │  │   (s1)       │         │
│  │ node.roles:  │  │ node.roles:  │  │ node.roles:  │         │
│  │ [data]       │  │ [data]       │  │ [search]     │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│                                                    │          │
│                                         ┌──────────┴──────────┐
│                                         ▼                     ▼
│                                  ┌──────────────┐  ┌──────────────┐
│                                  │   Search     │  │ Coordinating │
│                                  │   (s2)       │  │   (c1)       │
│                                  │ node.roles:  │  │ node.roles:  │
│                                  │ [search]     │  │ []           │
│                                  └──────────────┘  └──────────────┘
└─────────────────────────────────────────────────────────────────┘
```

**Всего узлов: 8**
- **3 Cluster Manager** — для обеспечения кворума (может выдержать отказ 1 узла)
- **2 Data** — для хранения и обработки данных
- **2 Search** — для выделенных поисковых нагрузок
- **1 Coordinating** — для приёма и маршрутизации запросов (пустой список ролей)

---

### Часть 3. Конфигурация

#### Шаг 1. Создание docker-compose.yml

Создайте файл `~/opensearch-roles-lab/docker-compose.yml`:

```yaml
version: '3'

services:
  # ============================================
  # 1. CLUSTER MANAGER NODES (3 узла для кворума)
  # ============================================
  opensearch-cm1:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-cm1
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-cm1
      - node.roles=cluster_manager
      - discovery.seed_hosts=opensearch-cm1,opensearch-cm2,opensearch-cm3
      - cluster.initial_cluster_manager_nodes=opensearch-cm1,opensearch-cm2,opensearch-cm3
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms256m -Xmx256m"
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=MyStrongPassword123!
      - "OPENSEARCH_NETWORK_HOST=0.0.0.0"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - cm1-data:/usr/share/opensearch/data
    ports:
      - "9201:9200"
      - "9601:9600"
    networks:
      - opensearch-net

  opensearch-cm2:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-cm2
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-cm2
      - node.roles=cluster_manager
      - discovery.seed_hosts=opensearch-cm1,opensearch-cm2,opensearch-cm3
      - cluster.initial_cluster_manager_nodes=opensearch-cm1,opensearch-cm2,opensearch-cm3
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms256m -Xmx256m"
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=MyStrongPassword123!
      - "OPENSEARCH_NETWORK_HOST=0.0.0.0"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - cm2-data:/usr/share/opensearch/data
    ports:
      - "9202:9200"
      - "9602:9600"
    networks:
      - opensearch-net

  opensearch-cm3:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-cm3
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-cm3
      - node.roles=cluster_manager
      - discovery.seed_hosts=opensearch-cm1,opensearch-cm2,opensearch-cm3
      - cluster.initial_cluster_manager_nodes=opensearch-cm1,opensearch-cm2,opensearch-cm3
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms256m -Xmx256m"
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=MyStrongPassword123!
      - "OPENSEARCH_NETWORK_HOST=0.0.0.0"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - cm3-data:/usr/share/opensearch/data
    ports:
      - "9203:9200"
      - "9603:9600"
    networks:
      - opensearch-net

  # ============================================
  # 2. DATA NODES (2 узла для хранения данных)
  # ============================================
  opensearch-d1:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-d1
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-d1
      - node.roles=data
      - discovery.seed_hosts=opensearch-cm1,opensearch-cm2,opensearch-cm3
      - cluster.initial_cluster_manager_nodes=opensearch-cm1,opensearch-cm2,opensearch-cm3
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m"
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=MyStrongPassword123!
      - "OPENSEARCH_NETWORK_HOST=0.0.0.0"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - d1-data:/usr/share/opensearch/data
    ports:
      - "9204:9200"
      - "9604:9600"
    networks:
      - opensearch-net
    depends_on:
      - opensearch-cm1
      - opensearch-cm2
      - opensearch-cm3

  opensearch-d2:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-d2
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-d2
      - node.roles=data
      - discovery.seed_hosts=opensearch-cm1,opensearch-cm2,opensearch-cm3
      - cluster.initial_cluster_manager_nodes=opensearch-cm1,opensearch-cm2,opensearch-cm3
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m"
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=MyStrongPassword123!
      - "OPENSEARCH_NETWORK_HOST=0.0.0.0"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - d2-data:/usr/share/opensearch/data
    ports:
      - "9205:9200"
      - "9605:9600"
    networks:
      - opensearch-net
    depends_on:
      - opensearch-cm1
      - opensearch-cm2
      - opensearch-cm3

  # ============================================
  # 3. SEARCH NODES (2 узла для поисковых нагрузок)
  # ============================================
  opensearch-s1:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-s1
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-s1
      - node.roles=search
      - discovery.seed_hosts=opensearch-cm1,opensearch-cm2,opensearch-cm3
      - cluster.initial_cluster_manager_nodes=opensearch-cm1,opensearch-cm2,opensearch-cm3
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m"
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=MyStrongPassword123!
      - "OPENSEARCH_NETWORK_HOST=0.0.0.0"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - s1-data:/usr/share/opensearch/data
    ports:
      - "9206:9200"
      - "9606:9600"
    networks:
      - opensearch-net
    depends_on:
      - opensearch-cm1
      - opensearch-cm2
      - opensearch-cm3

  opensearch-s2:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-s2
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-s2
      - node.roles=search
      - discovery.seed_hosts=opensearch-cm1,opensearch-cm2,opensearch-cm3
      - cluster.initial_cluster_manager_nodes=opensearch-cm1,opensearch-cm2,opensearch-cm3
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m"
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=MyStrongPassword123!
      - "OPENSEARCH_NETWORK_HOST=0.0.0.0"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - s2-data:/usr/share/opensearch/data
    ports:
      - "9207:9200"
      - "9607:9600"
    networks:
      - opensearch-net
    depends_on:
      - opensearch-cm1
      - opensearch-cm2
      - opensearch-cm3

  # ============================================
  # 4. COORDINATING NODE (1 узел для маршрутизации)
  # ============================================
  opensearch-c1:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-c1
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-c1
      - node.roles=[]           # Пустой список = coordinating-only узел
      - discovery.seed_hosts=opensearch-cm1,opensearch-cm2,opensearch-cm3
      - cluster.initial_cluster_manager_nodes=opensearch-cm1,opensearch-cm2,opensearch-cm3
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms256m -Xmx256m"
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=MyStrongPassword123!
      - "OPENSEARCH_NETWORK_HOST=0.0.0.0"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - c1-data:/usr/share/opensearch/data
    ports:
      - "9200:9200"            # Основной порт для клиентских подключений
      - "9600:9600"
    networks:
      - opensearch-net
    depends_on:
      - opensearch-cm1
      - opensearch-cm2
      - opensearch-cm3
      - opensearch-d1
      - opensearch-d2
      - opensearch-s1
      - opensearch-s2

  # ============================================
  # 5. OPENSEARCH DASHBOARDS
  # ============================================
  opensearch-dashboards:
    image: opensearchproject/opensearch-dashboards:latest
    container_name: opensearch-dashboards
    ports:
      - "5601:5601"
    environment:
      - OPENSEARCH_HOSTS=["http://opensearch-c1:9200"]
    networks:
      - opensearch-net
    depends_on:
      - opensearch-c1

volumes:
  cm1-data:
  cm2-data:
  cm3-data:
  d1-data:
  d2-data:
  s1-data:
  s2-data:
  c1-data:

networks:
  opensearch-net:
    driver: bridge
```

**Ключевые особенности конфигурации:**

| Элемент | Описание |
|---------|----------|
| **`node.roles=cluster_manager`** | Узлы только для управления кластером |
| **`node.roles=data`** | Узлы только для хранения и обработки данных |
| **`node.roles=search`** | Узлы только для поисковых нагрузок |
| **`node.roles=[]`** | Coordinating-only узел (принимает запросы, не хранит данные) |
| **`discovery.seed_hosts`** | Список узлов для обнаружения кластера |
| **`cluster.initial_cluster_manager_nodes`** | Начальные узлы, eligible для выбора cluster manager |
| **Разные порты** | Каждый узел доступен на своём порту для отладки |
| **`depends_on`** | Обеспечивает порядок запуска (сначала cluster manager) |

---

### Часть 4. Запуск и проверка

#### Шаг 1. Запуск стека

```bash
cd ~/opensearch-roles-lab
docker compose up -d
```

#### Шаг 2. Проверка статуса контейнеров

```bash
docker compose ps
```

Ожидаемый вывод: все 9 контейнеров (3 cm, 2 data, 2 search, 1 coordinating, 1 dashboards) в состоянии `Up`.

#### Шаг 3. Проверка состояния кластера

```bash
# Через coordinating узел (порт 9200)
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cluster/health?pretty"
```

Ожидаемый ответ: статус `green` или `yellow`, количество узлов `8`.

#### Шаг 4. Проверка списка узлов и их ролей

```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cat/nodes?v"
```

Пример вывода:
```
ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role cluster_manager name
172.20.0.6           25          65   0    0.00    0.00     0.00 m          -                  opensearch-cm1
172.20.0.7           28          65   0    0.00    0.00     0.00 m          -                  opensearch-cm2
172.20.0.8           22          65   0    0.00    0.00     0.00 m          -                  opensearch-cm3
172.20.0.9           45          70   0    0.00    0.00     0.00 d          -                  opensearch-d1
172.20.0.10          42          70   0    0.00    0.00     0.00 d          -                  opensearch-d2
172.20.0.11          30          65   0    0.00    0.00     0.00 s          -                  opensearch-s1
172.20.0.12          32          65   0    0.00    0.00     0.00 s          -                  opensearch-s2
172.20.0.13          15          60   0    0.00    0.00     0.00 -          -                  opensearch-c1
```

**Расшифровка `node.role`:**
- `m` — cluster_manager
- `d` — data
- `s` — search
- `-` — coordinating-only (нет ролей)

#### Шаг 5. Проверка, кто является активным cluster manager

```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cat/master?v"
```

#### Шаг 6. Проверка через OpenSearch Dashboards

Откройте в браузере `http://<IP_сервера>:5601`. Авторизуйтесь как `admin` с паролем `MyStrongPassword123!`. В разделе **Dashboards Management** → **Dev Tools** выполните:

```
GET _cat/nodes?v
```

---

### Часть 5. Задания для самостоятельного выполнения

#### Задание 1. Симуляция отказа cluster manager узла

1. Остановите один из cluster manager узлов:
   ```bash
   docker stop opensearch-cm1
   ```

2. Проверьте состояние кластера:
   ```bash
   curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cluster/health?pretty"
   ```

3. Проверьте, кто стал новым cluster manager:
   ```bash
   curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cat/master?v"
   ```

4. Восстановите узел:
   ```bash
   docker start opensearch-cm1
   ```

5. **Вопрос:** Почему кластер продолжает работать после остановки одного из трёх cluster manager узлов?

#### Задание 2. Проверка изоляции ролей

1. Попробуйте создать индекс через data узел напрямую (порт 9204):
   ```bash
   curl -k -u admin:MyStrongPassword123! -X PUT "https://localhost:9204/test-index"
   ```
   **Ожидаемый результат:** ошибка, так как data узел не является cluster manager и не может создавать индексы.

2. Создайте индекс через coordinating узел (порт 9200):
   ```bash
   curl -k -u admin:MyStrongPassword123! -X PUT "https://localhost:9200/test-index"
   ```
   **Ожидаемый результат:** успех (`{"acknowledged":true}`).

3. **Вопрос:** Почему запрос через coordinating узел сработал, а через data узел — нет?

#### Задание 3. Настройка выделенных search-реплик

1. Создайте индекс с настройкой `index.number_of_replicas: 2`:
   ```bash
   curl -k -u admin:MyStrongPassword123! -X PUT "https://localhost:9200/logs-index" -H 'Content-Type: application/json' -d '
   {
     "settings": {
       "number_of_shards": 1,
       "number_of_replicas": 2
     }
   }'
   ```

2. Проверьте распределение шардов:
   ```bash
   curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cat/shards/logs-index?v"
   ```

3. **Вопрос:** На каких узлах разместились реплики? Попали ли они на search-узлы?

#### Задание 4. Сравнение производительности

1. Индексируйте 10 000 тестовых документов через coordinating узел:
   ```bash
   for i in {1..10000}; do
     curl -k -u admin:MyStrongPassword123! -X POST "https://localhost:9200/test-index/_doc/$i" -H 'Content-Type: application/json' -d "{\"field\": \"value$i\"}" 2>/dev/null
   done
   ```

2. Выполните поисковый запрос и замерьте время:
   ```bash
   time curl -k -u admin:MyStrongPassword123! "https://localhost:9200/test-index/_search?q=*&size=100"
   ```

3. Отключите search-узлы и повторите замер:
   ```bash
   docker stop opensearch-s1 opensearch-s2
   time curl -k -u admin:MyStrongPassword123! "https://localhost:9200/test-index/_search?q=*&size=100"
   ```

4. **Вопрос:** Как изменилось время выполнения поиска после отключения search-узлов?

---

### Часть 6. Дополнительно: Добавление ingest-узлов

При необходимости можно добавить выделенные ingest-узлы для предобработки данных:

```yaml
opensearch-i1:
  image: opensearchproject/opensearch:latest
  container_name: opensearch-i1
  environment:
    - cluster.name=opensearch-cluster
    - node.name=opensearch-i1
    - node.roles=ingest
    - discovery.seed_hosts=opensearch-cm1,opensearch-cm2,opensearch-cm3
    - cluster.initial_cluster_manager_nodes=opensearch-cm1,opensearch-cm2,opensearch-cm3
    - bootstrap.memory_lock=true
    - "OPENSEARCH_JAVA_OPTS=-Xms256m -Xmx256m"
    - OPENSEARCH_INITIAL_ADMIN_PASSWORD=MyStrongPassword123!
    - "OPENSEARCH_NETWORK_HOST=0.0.0.0"
  ulimits:
    memlock:
      soft: -1
      hard: -1
  volumes:
    - i1-data:/usr/share/opensearch/data
  ports:
    - "9208:9200"
  networks:
    - opensearch-net
  depends_on:
    - opensearch-cm1
    - opensearch-cm2
    - opensearch-cm3
```

---

### Возможные проблемы и решения

| Проблема | Возможное решение |
| :--- | :--- |
| **OpenSearch не запускается** | Проверьте `vm.max_map_count=262144`. Убедитесь, что пароль соответствует требованиям (минимум 8 символов, заглавная, строчная буквы, цифра, спецсимвол). |
| **Узлы не видят друг друга** | Проверьте `discovery.seed_hosts` — имена должны совпадать с `container_name` других узлов. Убедитесь, что все узлы в одной сети Docker. |
| **Cluster manager не выбирается** | Проверьте, что `cluster.initial_cluster_manager_nodes` содержит минимум 3 узла. Убедитесь, что все 3 узла запущены. |
| **Data узлы не принимают данные** | Data узлы не являются cluster-manager-eligible, поэтому не могут создавать индексы. Используйте coordinating узел для всех клиентских запросов. |
| **Search узлы не видят данные** | Search узлы содержат только search-реплики. Убедитесь, что при создании индекса указано `number_of_replicas >= 1` и настройки аллокации позволяют размещать реплики на search-узлах. |
| **Координирующий узел не стартует** | `node.roles=[]` — это корректный синтаксис для coordinating-only узла. Убедитесь, что в `docker-compose.yml` нет лишних пробелов. |

---

