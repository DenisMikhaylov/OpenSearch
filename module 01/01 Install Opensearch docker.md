
### 📝 План лабораторной работы: Установка OpenSearch через Docker


#### **Шаг 1: Подготовка системы (Debian 13)**

Перед запуском OpenSearch в Docker нужно выполнить обязательную настройку ядра Linux. Это критически важный шаг, без которого контейнер не запустится.

1.  **Обновите систему** и установите базовые пакеты:
    ```bash
    sudo apt update && sudo apt upgrade -y
    sudo apt install -y curl wget gnupg apt-transport-https
    ```
2.  **Настройте параметры виртуальной памяти**:
    OpenSearch требует увеличить количество отображений памяти (`vm.max_map_count`). Временно отключите swap и примените новые настройки :
    ```bash
    sudo swapoff -a
    echo 'vm.max_map_count=262144' | sudo tee -a /etc/sysctl.conf
    sudo sysctl -p
    ```
   
#### **Шаг 2: Установка Docker и Docker Compose**
```
https://docs.docker.com/engine/install/debian/
```
```
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```
```
apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

#### **Шаг 3: Запуск OpenSearch кластера**

Самый простой способ — использовать готовый `docker-compose.yml` файл от проекта OpenSearch .

1.  **Скачайте файл `docker-compose.yml`** (для актуальной версии 3.x) :
    ```bash
    wget https://raw.githubusercontent.com/opensearch-project/documentation-website/main/assets/examples/docker-compose.yml
    ```
    *Этот файл создает кластер из двух узлов OpenSearch и одной панели Dashboards.*

2.  **Установите пароль администратора**.
    **Важно!** Для OpenSearch версии 2.12 и новее обязательно задать пароль для пользователя `admin` через переменную окружения перед запуском .
    ```bash
    export OPENSEARCH_INITIAL_ADMIN_PASSWORD="Ваш пароль"
    ```

3.  **Запустите кластер**:
    ```bash
    docker compose up -d
    ```
    Флаг `-d` запускает контейнеры в фоновом режиме.

#### **Шаг 4: Проверка работоспособности**

Убедитесь, что все запустилось корректно.

1.  **Проверьте статус контейнеров**:
    ```bash
    docker compose ps
    ```
    Вы должны увидеть три сервиса: `opensearch-node1`, `opensearch-node2` и `opensearch-dashboards` со статусом `running` .

2.  **Отправьте запрос к OpenSearch API**:
    ```bash
    curl -k -u admin:"ваш пароль" https://localhost:9200
    ```
    Успешный ответ будет содержать JSON с информацией о кластере .

3.  **Откройте OpenSearch Dashboards**:
    Откройте в браузере `http://localhost:5601` и войдите с логином `admin` и вашим паролем .
