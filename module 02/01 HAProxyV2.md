## 🧪 Лабораторная работа: OpenSearch кластер из 2 узлов + HAProxy на одном хосте с локальными IP-адресами

### Цель
Настроить отказоустойчивый кластер OpenSearch (2 узла) и балансировщик HAProxy, используя статические IP-адреса внутри Docker-сети для имитации разных серверов.

---

### 1. Подготовка системы (Debian 13)

Выполните на хосте:

```bash
# Обновление и установка базовых пакетов
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget gnupg apt-transport-https

# Установка Docker
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Настройка ядра (обязательно для OpenSearch)
sudo swapoff -a
echo 'vm.max_map_count=262144' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Добавление пользователя в группу docker (перезайдите после)
sudo usermod -aG docker $USER
```

---

### 2. Создание проекта

Создайте директорию и перейдите в неё:

```bash
mkdir ~/opensearch-haproxy-lab && cd ~/opensearch-haproxy-lab
```

Внутри создайте два файла:
- `docker-compose.yml` – описание всех контейнеров и сети
- `haproxy.cfg` – конфигурация балансировщика

---

### 3. Файл `docker-compose.yml`

Здесь мы задаём **пользовательскую сеть** с подсетью `172.20.0.0/24` и назначаем каждому контейнеру статический IP.

```yaml
version: '3.8'

services:
  # Узел OpenSearch 1
  opensearch-node1:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-node1
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-node1
      - discovery.seed_hosts=172.20.0.2,172.20.0.3   # IP-адреса обоих узлов
      - cluster.initial_cluster_manager_nodes=opensearch-node1,opensearch-node2
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m"
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=MyStr0ngP@ssw0rd!
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - data01:/usr/share/opensearch/data
    networks:
      opensearch-net:
        ipv4_address: 172.20.0.2   # локальный IP для узла 1
    ports:
      - "9201:9200"   # проброс портов для внешнего доступа (опционально)
      - "9601:9600"

  # Узел OpenSearch 2
  opensearch-node2:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-node2
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-node2
      - discovery.seed_hosts=172.20.0.2,172.20.0.3
      - cluster.initial_cluster_manager_nodes=opensearch-node1,opensearch-node2
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m"
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=MyStr0ngP@ssw0rd!
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - data02:/usr/share/opensearch/data
    networks:
      opensearch-net:
        ipv4_address: 172.20.0.3   # локальный IP для узла 2
    ports:
      - "9202:9200"
      - "9602:9600"

  # HAProxy балансировщик
  haproxy:
    image: haproxy:latest
    container_name: haproxy
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    networks:
      opensearch-net:
        ipv4_address: 172.20.0.4   # IP балансировщика (можно и без статики)
    ports:
      - "9200:9200"   # внешний порт для доступа к балансировщику
      - "8404:8404"   # страница статистики
    depends_on:
      - opensearch-node1
      - opensearch-node2

volumes:
  data01:
  data02:

networks:
  opensearch-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24
```

---

### 4. Файл `haproxy.cfg`

Конфигурация HAProxy использует **локальные IP-адреса** узлов (172.20.0.2 и 172.20.0.3) внутри Docker-сети.

Создайте файл `haproxy.cfg` в той же директории:

```cfg
global
    log /dev/log local0
    maxconn 4096
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5s
    timeout client  30s
    timeout server  30s
    retries 3

frontend opensearch_frontend
    bind *:9200
    default_backend opensearch_backend

backend opensearch_backend
    balance roundrobin
    # Используем локальные IP-адреса контейнеров
    server node1 172.20.0.2:9200 check
    server node2 172.20.0.3:9200 check

listen stats
    bind *:8404
    mode http
    stats enable
    stats uri /haproxy?stats
    stats refresh 10s
    stats auth admin:Pandor4!
```

---

### 5. Запуск и проверка

Запустите всё командой:

```bash
docker compose up -d
```

Проверьте статус контейнеров:

```bash
docker compose ps
```

Должны быть три контейнера в состоянии `Up`.

---

### 6. Тестирование балансировки

**Запрос к кластеру через HAProxy** (порт 9200 на хосте):

```bash
curl -XGET -ku admin:'MyStr0ngP@ssw0rd!' "http://localhost:9200/_cat/nodes?v"
```

В ответе вы увидите два узла с именами `opensearch-node1` и `opensearch-node2`, что подтверждает работу кластера через балансировщик.

**Проверка распределения нагрузки** – выполните несколько запросов и посмотрите логи HAProxy (или откройте страницу статистики):

```
http://<IP_вашего_хоста>:8404/haproxy?stats
```
Логин: `admin`, пароль: `Pandor4!`.

На странице статистики вы увидите, как HAProxy переключает запросы между узлами (алгоритм `roundrobin`).

---

### 7. Дополнительная проверка – отказоустойчивость

Остановите один из узлов (например, `opensearch-node1`):

```bash
docker stop opensearch-node1
```

Снова выполните запрос через HAProxy:

```bash
curl -XGET -ku admin:'MyStr0ngP@ssw0rd!' "http://localhost:9200/_cat/nodes?v"
```

Вы всё ещё получите ответ, но теперь только от работающего узла (`opensearch-node2`). HAProxy автоматически исключил недоступный сервер из ротации.

Восстановите узел:

```bash
docker start opensearch-node1
```

HAProxy снова добавит его в пул после успешной проверки (через несколько секунд).

---

Теперь у вас есть полноценная лабораторная с двумя OpenSearch узлами и HAProxy, использующая **локальные IP-адреса** внутри Docker-сети. Это идеальный полигон для дальнейшего изучения мониторинга, масштабирования и безопасности OpenSearch.
