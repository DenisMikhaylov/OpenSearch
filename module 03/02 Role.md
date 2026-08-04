
## 🧪 Лабораторная работа: Кластер OpenSearch с разделением ролей (статическая IP-адресация)

### Особенности данной конфигурации

- Все контейнеры используют **кастомную сеть `opensearch-net`** с подсетью `172.20.0.0/16`.
- **Каждому узлу назначен фиксированный IP-адрес** в этой сети.
- Параметры `discovery.seed_hosts` и `cluster.initial_cluster_manager_nodes` используют **IP-адреса**, а не имена сервисов, что исключает зависимость от DNS.
- Для внешнего доступа к каждому узлу проброшены уникальные порты на хосте (9201–9208), но основной вход в кластер осуществляется через **coordinating узел** на порту `9200`.

---

### Часть 1. Подготовка среды (без изменений)

```bash
sudo swapoff -a
sudo sysctl -w vm.max_map_count=262144
# Добавить в /etc/sysctl.conf для постоянства
```

Установите Docker и Docker Compose.

Создайте рабочую директорию:

```bash
mkdir ~/opensearch-roles-lab
cd ~/opensearch-roles-lab
```

---

### Часть 2. Конфигурация с фиксированными IP-адресами

Создайте файл `docker-compose.yml` со следующим содержимым:

```yaml
version: '3'

services:
  # ============================================
  # 1. CLUSTER MANAGER NODES (3 узла)
  # ============================================
  opensearch-cm1:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-cm1
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-cm1
      - node.roles=cluster_manager
      - discovery.seed_hosts=172.20.0.10,172.20.0.11,172.20.0.12
      - cluster.initial_cluster_manager_nodes=172.20.0.10,172.20.0.11,172.20.0.12
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms256m -Xmx256m"
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=MyStrongPassword123!
      - "OPENSEARCH_NETWORK_HOST=0.0.0.0"
      - "OPENSEARCH_NODE_NAME=opensearch-cm1"
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
      opensearch-net:
        ipv4_address: 172.20.0.10   # статический IP

  opensearch-cm2:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-cm2
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-cm2
      - node.roles=cluster_manager
      - discovery.seed_hosts=172.20.0.10,172.20.0.11,172.20.0.12
      - cluster.initial_cluster_manager_nodes=172.20.0.10,172.20.0.11,172.20.0.12
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms256m -Xmx256m"
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=MyStrongPassword123!
      - "OPENSEARCH_NETWORK_HOST=0.0.0.0"
      - "OPENSEARCH_NODE_NAME=opensearch-cm2"
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
      opensearch-net:
        ipv4_address: 172.20.0.11

  opensearch-cm3:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-cm3
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-cm3
      - node.roles=cluster_manager
      - discovery.seed_hosts=172.20.0.10,172.20.0.11,172.20.0.12
      - cluster.initial_cluster_manager_nodes=172.20.0.10,172.20.0.11,172.20.0.12
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms256m -Xmx256m"
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=MyStrongPassword123!
      - "OPENSEARCH_NETWORK_HOST=0.0.0.0"
      - "OPENSEARCH_NODE_NAME=opensearch-cm3"
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
      opensearch-net:
        ipv4_address: 172.20.0.12

  # ============================================
  # 2. DATA NODES (2 узла)
  # ============================================
  opensearch-d1:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-d1
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-d1
      - node.roles=data
      - discovery.seed_hosts=172.20.0.10,172.20.0.11,172.20.0.12
      - cluster.initial_cluster_manager_nodes=172.20.0.10,172.20.0.11,172.20.0.12
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m"
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=MyStrongPassword123!
      - "OPENSEARCH_NETWORK_HOST=0.0.0.0"
      - "OPENSEARCH_NODE_NAME=opensearch-d1"
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
      opensearch-net:
        ipv4_address: 172.20.0.20
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
      - discovery.seed_hosts=172.20.0.10,172.20.0.11,172.20.0.12
      - cluster.initial_cluster_manager_nodes=172.20.0.10,172.20.0.11,172.20.0.12
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m"
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=MyStrongPassword123!
      - "OPENSEARCH_NETWORK_HOST=0.0.0.0"
      - "OPENSEARCH_NODE_NAME=opensearch-d2"
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
      opensearch-net:
        ipv4_address: 172.20.0.21
    depends_on:
      - opensearch-cm1
      - opensearch-cm2
      - opensearch-cm3

  # ============================================
  # 3. SEARCH NODES (2 узла)
  # ============================================
  opensearch-s1:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-s1
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-s1
      - node.roles=search
      - discovery.seed_hosts=172.20.0.10,172.20.0.11,172.20.0.12
      - cluster.initial_cluster_manager_nodes=172.20.0.10,172.20.0.11,172.20.0.12
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m"
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=MyStrongPassword123!
      - "OPENSEARCH_NETWORK_HOST=0.0.0.0"
      - "OPENSEARCH_NODE_NAME=opensearch-s1"
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
      opensearch-net:
        ipv4_address: 172.20.0.30
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
      - discovery.seed_hosts=172.20.0.10,172.20.0.11,172.20.0.12
      - cluster.initial_cluster_manager_nodes=172.20.0.10,172.20.0.11,172.20.0.12
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m"
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=MyStrongPassword123!
      - "OPENSEARCH_NETWORK_HOST=0.0.0.0"
      - "OPENSEARCH_NODE_NAME=opensearch-s2"
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
      opensearch-net:
        ipv4_address: 172.20.0.31
    depends_on:
      - opensearch-cm1
      - opensearch-cm2
      - opensearch-cm3

  # ============================================
  # 4. COORDINATING NODE (1 узел)
  # ============================================
  opensearch-c1:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-c1
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-c1
      - node.roles=[]
      - discovery.seed_hosts=172.20.0.10,172.20.0.11,172.20.0.12
      - cluster.initial_cluster_manager_nodes=172.20.0.10,172.20.0.11,172.20.0.12
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms256m -Xmx256m"
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=MyStrongPassword123!
      - "OPENSEARCH_NETWORK_HOST=0.0.0.0"
      - "OPENSEARCH_NODE_NAME=opensearch-c1"
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
      - "9200:9200"            # основной порт для клиентов
      - "9600:9600"
    networks:
      opensearch-net:
        ipv4_address: 172.20.0.40
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
      - OPENSEARCH_HOSTS=["http://172.20.0.40:9200"]
    networks:
      opensearch-net:
        ipv4_address: 172.20.0.50
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
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

---

### Часть 3. Запуск и проверка (с использованием IP-адресов)

#### Шаг 1. Запуск стека

```bash
cd ~/opensearch-roles-lab
docker compose up -d
```

#### Шаг 2. Проверка состояния кластера

```bash
# Подключение через coordinating узел (порт 9200)
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cluster/health?pretty"
```

Ожидается статус `green` или `yellow`, количество узлов — 8.

#### Шаг 3. Просмотр всех узлов с их IP и ролями

```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cat/nodes?v&h=ip,node.role,name"
```

Пример вывода:
```
ip          node.role name
172.20.0.10 m         opensearch-cm1
172.20.0.11 m         opensearch-cm2
172.20.0.12 m         opensearch-cm3
172.20.0.20 d         opensearch-d1
172.20.0.21 d         opensearch-d2
172.20.0.30 s         opensearch-s1
172.20.0.31 s         opensearch-s2
172.20.0.40 -         opensearch-c1
```

#### Шаг 4. Проверка активного cluster manager

```bash
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cat/master?v&h=host,name"
```

В колонке `host` будет отображаться IP-адрес текущего master-узла.

---

### Часть 4. Задания для самостоятельного выполнения (с учётом статических IP)

**Задание 1.** Симуляция отказа cluster manager:

```bash
docker stop opensearch-cm1
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cluster/health?pretty"
curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cat/master?v"
docker start opensearch-cm1
```

**Задание 2.** Создание индекса и проверка распределения реплик:

```bash
curl -k -u admin:MyStrongPassword123! -X PUT "https://localhost:9200/test-index" -H 'Content-Type: application/json' -d '
{
  "settings": {
    "number_of_shards": 2,
    "number_of_replicas": 2
  }
}'

curl -k -u admin:MyStrongPassword123! "https://localhost:9200/_cat/shards/test-index?v&h=index,shard,prirep,node"
```

Обратите внимание, что реплики должны распределяться между data и search узлами.

**Задание 3.** Прямой доступ к data узлу (порт 9204) и попытка создания индекса:

```bash
curl -k -u admin:MyStrongPassword123! -X PUT "https://localhost:9204/test-index2"
```

Ожидается ошибка `403` или `400`, так как data-узел не является cluster-manager.

---

### Часть 5. Особенности работы без DNS

- Все узлы общаются по **IP-адресам** внутри сети `172.20.0.0/16`.
- При добавлении новых узлов необходимо вручную назначать им IP и добавлять их в `discovery.seed_hosts` на всех существующих узлах.
- Для внешних клиентов (например, приложений) входная точка — **coordinating узел** с IP `172.20.0.40` (или хост-порт `9200`).
- Если необходимо изменить подсеть, нужно отредактировать секцию `ipam` в сети и пересчитать IP для всех контейнеров.

-
