# Лабораторная работа: Визуализация данных NGINX и Syslog в Grafana


## Часть 1: Установка Grafana на Debian 13

### 1.1. Подготовка системы

Обновите пакетный менеджер и установите базовые зависимости:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https wget gnupg curl
```

**Важно:** В Debian 13 (Trixie) пакет `software-properties-common` был удалён из репозитория, поэтому его установка не требуется и вызовет ошибку `E: Unable to locate package software-properties-common` .

### 1.2. Добавление репозитория Grafana через зеркало Яндекса

В связи с ограничениями доступа к официальному репозиторию Grafana из РФ, используйте зеркало Яндекса .

**Шаг 1: Импорт GPG-ключа**

Создайте директорию для ключей и импортируйте GPG-ключ:

```bash
sudo mkdir -p /etc/apt/keyrings
wget -q -O - https://apt.grafana.com/gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/grafana.gpg
```

**Примечание:** В современных версиях Debian ключи хранятся в `/etc/apt/keyrings/`, что является более безопасным способом по сравнению с устаревшим `apt-key` .

**Шаг 2: Добавление репозитория через зеркало Яндекса**

```bash
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://mirror.yandex.ru/mirrors/packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
```

Если ранее был добавлен официальный репозиторий, закомментируйте его:

```bash
sudo sed -i 's|^deb .*apt.grafana.com|# &|' /etc/apt/sources.list.d/grafana.list
```

**Шаг 3: Установка Grafana**

```bash
sudo apt update
sudo apt install -y grafana
```

**Шаг 4: Проверка версии**

```bash
grafana-server -v
```

Ожидаемый вывод: `Version 11.x.x` или выше .

### 1.3. Запуск и настройка автозагрузки

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now grafana-server
sudo systemctl status grafana-server
```

### 1.4. Настройка сетевых параметров (опционально)

По умолчанию Grafana слушает на порту `3000` на всех интерфейсах. При необходимости измените параметры в файле `/etc/grafana/grafana.ini` :

```ini
# строка 39: протокол (http, https, h2, socket)
protocol = http

# строка 48: порт (по умолчанию 3000)
;http_port = 3000

# строка 51: доменное имя
;domain = localhost
```

После изменений перезапустите сервис:

```bash
sudo systemctl restart grafana-server
```

### 1.5. Доступ к Grafana

Откройте браузер и перейдите по адресу: `http://<ip-адрес-сервера>:3000`

**Учётные данные по умолчанию:**
- **Логин:** `admin`
- **Пароль:** `admin`

При первом входе система предложит сменить пароль.

---

## Часть 2: Установка плагина OpenSearch для Grafana

### 2.1. Требования

Плагин OpenSearch для Grafana требует **Grafana версии 10.4.0 или новее** . Установленная версия должна соответствовать этому требованию.

### 2.2. Установка плагина через CLI

Используйте утилиту `grafana-cli` для установки плагина :

```bash
sudo grafana-cli plugins install grafana-opensearch-datasource
```

Плагин будет установлен в директорию `/var/lib/grafana/plugins` .

### 2.3. Перезапуск Grafana

После установки плагина перезапустите Grafana:

```bash
sudo systemctl restart grafana-server
```

---

## Часть 3: Настройка источника данных OpenSearch

### 3.1. Добавление источника данных

1. В левом меню Grafana выберите **Connections** → **Add new connection** 
2. В строке поиска введите **OpenSearch**
3. Выберите **OpenSearch** из результатов
4. Нажмите **Add new data source** 

### 3.2. Основные настройки

Заполните следующие поля :

| Поле | Значение |
|------|----------|
| **Name** | `OpenSearch` |
| **Default** | Отметьте, если хотите сделать этот источник основным |
| **URL** | `http://localhost:9200` (или адрес вашего OpenSearch кластера) |

### 3.3. Настройки OpenSearch

| Поле | Значение |
|------|----------|
| **Index name** | `nginx-logs-*,syslog-*` |
| **Pattern** | No pattern (по умолчанию) |
| **Time field name** | `timestamp` |
| **PPL enabled** | Отметьте (включено по умолчанию) |

### 3.4. Проверка подключения

Нажмите **Save & test**. Ожидаемый результат: сообщение об успешном подключении и обнаружении полей.

### 3.5. Альтернатива: Provisioning через YAML

Источник данных можно настроить через YAML-файл в системе provisioning :

```yaml
apiVersion: 1

datasources:
  - name: OpenSearch
    type: grafana-opensearch-datasource
    access: proxy
    url: http://localhost:9200
    basicAuth: false
    jsonData:
      flavor: opensearch
      version: "2.18.0"
      database: nginx-logs-*,syslog-*
      timeField: timestamp
      pplEnabled: true
      maxConcurrentShardRequests: 5
```

---

## Часть 4: Создание визуализаций для NGINX логов

### 4.1. Запрос для визуализации распределения HTTP-статусов

**PPL-запрос:**

```ppl
source = nginx-logs-* | stats count() by response | sort - count()
```

**Тип визуализации:** Pie chart

**Настройки:**
1. В **Explore** выберите источник данных `OpenSearch`
2. Переключитесь на язык **PPL**
3. Введите запрос и нажмите **Run query**
4. В выпадающем меню выберите **Pie chart**

### 4.2. Запрос для визуализации топ запросов (URL)

**PPL-запрос:**

```ppl
source = nginx-logs-* | stats count() by request.keyword | sort - count() | limit 15
```

**Тип визуализации:** Bar chart (horizontal)

### 4.3. Запрос для временного ряда запросов

**PPL-запрос:**

```ppl
source = nginx-logs-* | stats count() by SPAN(timestamp, 1h)
```

**Тип визуализации:** Area chart

**Важно:** Для формата Time series запрос должен возвращать ровно два столбца: временная метка и числовое значение.

---

## Часть 5: Создание визуализаций для Syslog

### 5.1. Запрос для распределения по программам

**PPL-запрос:**

```ppl
source = syslog-* | stats count() by program.keyword | sort - count() | limit 15
```

**Тип визуализации:** Pie chart

### 5.2. Запрос для топ хостов по активности

**PPL-запрос:**

```ppl
source = syslog-* | stats count() by hostname.keyword | sort - count() | limit 10
```

**Тип визуализации:** Bar chart

### 5.3. Запрос для временного ряда сообщений с фильтрацией по критичности

**PPL-запрос:**

```ppl
source = syslog-* | where severity = "err" OR severity = "crit" | stats count() by SPAN(timestamp, 1h)
```

**Тип визуализации:** Area chart

---

## Часть 6: Создание дашборда

### 6.1. Создание дашборда

1. В левом меню выберите **Dashboards** → **New** → **New dashboard**
2. Нажмите **+ Add visualization**
3. Выберите источник данных `OpenSearch`
4. Настройте запрос и тип визуализации
5. Нажмите **Apply** для добавления панели

### 6.2. Добавление нескольких панелей

Повторите процесс для каждой созданной визуализации. Расположите панели на дашборде с помощью drag-and-drop.

**Рекомендуемая структура:**
- **Верхний ряд:** NGINX Request Timeline (Area chart)
- **Средний ряд слева:** HTTP Status Distribution (Pie chart)
- **Средний ряд справа:** Top URLs (Bar chart)
- **Нижний ряд слева:** Syslog Events by Program (Pie chart)
- **Нижний ряд справа:** Top Hosts (Bar chart)

### 6.3. Настройка переменных (Template Variables)

**Шаг 1: Создание переменной для фильтрации по хосту**

1. На дашборде нажмите **Dashboard settings** (шестерёнка)
2. Перейдите на вкладку **Variables**
3. Нажмите **Add variable**
4. Настройте:
   - **Name:** `hostname`
   - **Type:** `Query`
   - **Data source:** `OpenSearch`
   - **Query:** `{"find": "terms", "field": "hostname.keyword", "size": 100}` 
5. Нажмите **Apply**

**Шаг 2: Использование переменной в запросах**

Теперь в запросах можно использовать `$hostname`:

```ppl
source = syslog-* | where hostname = '$hostname' | stats count() by program.keyword
```

### 6.4. Сохранение дашборда

Нажмите **Save dashboard**, введите имя `Unified Log Analytics` и сохраните.

---
