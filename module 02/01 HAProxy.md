## Лабораторная работа: Установка и настройка HAProxy для балансировки нагрузки кластера OpenSearch (2 узла)

### Требования к лабораторной работе
- Виртуальная машина или контейнер с Debian 13 (Trixie)
- Предварительно установленный и запущенный кластер OpenSearch из двух узлов (см. предыдущие лабораторные работы)
- Доступ в интернет для установки HAProxy
- Права суперпользователя (`sudo`)
- Минимум 2 ГБ оперативной памяти

### Ход выполнения работы

#### Часть 1. Подготовка системы и установка HAProxy

**1.1 Обновление системы**
```bash
sudo apt update && sudo apt upgrade -y
```

**1.2 Установка HAProxy**
Установите HAProxy и текстовый редактор для редактирования конфигурационных файлов:
```bash
sudo apt install -y haproxy vim
```

**1.3 Создание резервной копии конфигурации**
Перед внесением изменений создайте резервную копию файла конфигурации по умолчанию:
```bash
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.default
```

#### Часть 2. Настройка SSL-сертификата для HAProxy

Для работы с HTTPS необходимо создать самоподписанный сертификат (для тестовой среды). Для рабочего окружения рекомендуется использовать сертификат, подписанный доверенным центром сертификации .

**2.1 Создание директории для сертификатов**
```bash
sudo mkdir -p /etc/haproxy/certs
```

**2.2 Генерация ключа и сертификата**
```bash
sudo openssl req -x509 -nodes -days 365 \
   -newkey rsa:2048 \
   -keyout /etc/haproxy/certs/haproxy.key \
   -out /etc/haproxy/certs/haproxy.crt \
   -subj "/CN=haproxy.local"
```

**2.3 Объединение ключа и сертификата в PEM-файл**
HAProxy требует объединенный файл с ключом и сертификатом:
```bash
sudo cat /etc/haproxy/certs/haproxy.key /etc/haproxy/certs/haproxy.crt > /etc/haproxy/certs/haproxy.pem
```

#### Часть 3. Настройка HAProxy для балансировки OpenSearch

**3.1 Редактирование конфигурационного файла**
Откройте файл `/etc/haproxy/haproxy.cfg` для редактирования:
```bash
sudo vim /etc/haproxy/haproxy.cfg
```

**3.2 Добавление конфигурации для OpenSearch**
Внесите в файл следующую конфигурацию, заменив `<ip-node1>` и `<ip-node2>` на реальные IP-адреса ваших узлов OpenSearch :

```haproxy
global
   log /dev/log local1 notice
   maxconn 4096
   user haproxy
   group haproxy
   daemon
   tune.ssl.default-dh-param 2048

defaults
   log     global
   mode    http
   option  dontlognull
   timeout connect 5s
   timeout client  30s
   timeout server  30s
   retries 3

# ====================================
# Frontend — точка входа для клиентов
# ====================================
frontend opensearch_frontend
   bind *:9200 ssl crt /etc/haproxy/certs/haproxy.pem
   default_backend opensearch_backend

# ====================================
# Backend — узлы OpenSearch с балансировкой
# ====================================
backend opensearch_backend
   balance roundrobin
   server node1 <ip-node1>:9200 ssl verify none check
   server node2 <ip-node2>:9200 ssl verify none check

# ====================================
# Страница статистики HAProxy (опционально)
# ====================================
listen stats
   bind *:8404
   mode http
   stats enable
   stats uri /haproxy?stats
   stats refresh 10s
   stats auth admin:your_password_here
```

**Описание ключевых параметров:**
- `balance roundrobin` — алгоритм балансировки, при котором запросы распределяются по очереди между узлами 
- `ssl verify none` — отключает проверку SSL-сертификатов бэкендов (необходимо для самоподписанных сертификатов OpenSearch) 
- `check` — включает периодическую проверку доступности узла, автоматически исключая неработающие
- `stats auth` — задает логин и пароль для доступа к странице статистики

**3.3 Перезапуск HAProxy для применения конфигурации**
```bash
sudo systemctl restart haproxy
```

#### Часть 4. Проверка работы балансировщика

**4.1 Проверка статуса службы**
```bash
sudo systemctl status haproxy
```

**4.2 Проверка доступности узлов через HAProxy**
Выполните запрос к HAProxy (на локальном хосте) с аутентификацией (пароль администратора OpenSearch):
```bash
curl -XGET -ku admin:'your_opensearch_admin_password' "https://127.0.0.1:9200/_cat/nodes?v" --insecure
```

В ответе вы должны увидеть список узлов кластера OpenSearch.

**4.3 Проверка балансировки**
Выполните несколько последовательных запросов к корневому API OpenSearch и проверьте логи на узлах, чтобы убедиться, что запросы распределяются между ними:
```bash
curl -XGET -ku admin:'your_opensearch_admin_password' "https://127.0.0.1:9200/" --insecure
```
При каждом запросе HAProxy будет направлять трафик на разные узлы в соответствии с алгоритмом `roundrobin`.

**4.4 Доступ к странице статистики HAProxy**
Откройте в браузере:
```
http://<IP-адрес_сервера>:8404/haproxy?stats
```
Введите учетные данные, заданные в параметре `stats auth` .

#### Часть 5. Проверка отказоустойчивости

**5.1 Имитация отказа узла**
Остановите один из узлов OpenSearch:
```bash
sudo systemctl stop opensearch
```

**5.2 Проверка переключения трафика**
Выполните запрос к HAProxy:
```bash
curl -XGET -ku admin:'your_opensearch_admin_password' "https://127.0.0.1:9200/_cluster/health?pretty" --insecure
```

HAProxy автоматически исключит неработающий узел и будет направлять весь трафик на работающий.

**5.3 Восстановление узла и возврат в балансировку**
Запустите остановленный узел:
```bash
sudo systemctl start opensearch
```
После восстановления узла HAProxy автоматически включит его в пул бэкендов (при условии, что проверки `check` прошли успешно).
