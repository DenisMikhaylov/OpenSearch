# Лабораторная работа: Визуализация данных NGINX и Syslog в OpenSearch Dashboards 3.7

## Часть 1: Предварительная настройка

### 1.1. Проверка данных в OpenSearch

Прежде чем приступить к визуализации, убедитесь, что данные из потоков `nginx-logs-prod` и `syslog-prod` присутствуют в кластере.

```bash
# Проверка документов в потоке NGINX
curl -X GET "http://localhost:9200/nginx-logs-prod/_count"

# Проверка документов в потоке Syslog
curl -X GET "http://localhost:9200/syslog-prod/_count"
```

### 1.2. Настройка Workspace для визуализаций

В OpenSearch 3.7 для использования расширенных возможностей визуализаций требуется создать Workspace .

1.  Откройте OpenSearch Dashboards по адресу `http://localhost:5601`
2.  На главной странице выберите **Create workspace**
3.  Введите имя: `Log Analytics Workspace`
4.  В разделе **Use case** выберите **Observability**
5.  Нажмите **Create workspace**

### 1.3. Создание индекс-паттернов

Индекс-паттерн необходим для того, чтобы OpenSearch Dashboards знал, какие индексы искать при создании визуализаций .

**Создание паттерна для NGINX:**

1.  В левом меню перейдите в **Management** → **Index patterns**
2.  Нажмите **Create index pattern**
3.  В поле **Index pattern name** введите `nginx-logs-*`
4.  Нажмите **Next step**
5.  В поле **Time field** выберите `timestamp`
6.  Нажмите **Create index pattern**

**Создание паттерна для Syslog:**

1.  В **Index patterns** снова нажмите **Create index pattern**
2.  В поле **Index pattern name** введите `syslog-*`
3.  Нажмите **Next step**
4.  В поле **Time field** выберите `timestamp`
5.  Нажмите **Create index pattern**

---

## Часть 2: Создание визуализаций для NGINX логов

### 2.1. Круговая диаграмма: Распределение HTTP-статусов

Эта визуализация покажет, какие HTTP-статус-коды наиболее часто возвращаются сервером.

**Пошаговая инструкция** :

1.  В левом меню перейдите в **Visualize**
2.  Нажмите **Create visualization**
3.  Выберите **Pie** как тип диаграммы
4.  В качестве источника выберите индекс-паттерн `nginx-logs-*`
5.  В разделе **Buckets** нажмите **Add** → **Split slices**
6.  Настройте параметры:
    - **Aggregation**: `Terms`
    - **Field**: `response.keyword` (или `response` для числового поля)
    - **Size**: `10`
7.  Нажмите **Update** — диаграмма отобразит распределение статусов
8.  Нажмите **Save** и сохраните визуализацию как `[NGINX] HTTP Status Distribution`

### 2.2. Столбчатая диаграмма: Топ запрашиваемых URL

Эта визуализация покажет, какие URL запрашиваются чаще всего.

1.  В **Visualize** нажмите **Create visualization**
2.  Выберите **Bar**
3.  В качестве источника выберите `nginx-logs-*`
4.  В разделе **Buckets** нажмите **Add** → **X-axis**
5.  Настройте параметры:
    - **Aggregation**: `Terms`
    - **Field**: `request.keyword`
    - **Size**: `15`
6.  Нажмите **Add** → **Y-axis** (если требуется дополнительная метрика)
7.  Нажмите **Update**
8.  Сохраните как `[NGINX] Top URLs`

### 2.3. Линейная диаграмма: Количество запросов по времени

Эта визуализация покажет динамику трафика.

1.  В **Visualize** нажмите **Create visualization**
2.  Выберите **Area** (площадная диаграмма)
3.  Источник: `nginx-logs-*`
4.  В **Buckets** → **X-axis**:
    - **Aggregation**: `Date Histogram`
    - **Field**: `timestamp`
    - **Interval**: `1h` (или `15m` для более детальной картины)
5.  Нажмите **Add** → **Split series** для группировки по статусам (опционально):
    - **Aggregation**: `Terms`
    - **Field**: `response`
    - **Size**: `5`
6.  Нажмите **Update**
7.  Сохраните как `[NGINX] Request Timeline`

---

## Часть 3: Создание визуализаций для Syslog

### 3.1. Диаграмма-прибор: Количество критических событий

Эта визуализация покажет количество сообщений с уровнем критичности.

1.  В **Visualize** → **Create visualization** → **Metric**
2.  Источник: `syslog-*`
3.  Настройте метрику:
    - **Aggregation**: `Count`
    - Добавьте фильтр: `severity: "err" OR severity: "crit"`
4.  Нажмите **Update**
5.  Сохраните как `[Syslog] Critical Events Count`

### 3.2. Круговая диаграмма: Распределение по программам

Покажет, какие приложения генерируют больше всего логов.

1.  Создайте **Pie** диаграмму
2.  Источник: `syslog-*`
3.  В **Buckets** → **Split slices**:
    - **Aggregation**: `Terms`
    - **Field**: `program.keyword`
    - **Size**: `15`
4.  Нажмите **Update**
5.  Сохраните как `[Syslog] Events by Program`

### 3.3. Столбчатая диаграмма: Топ хостов по активности

Покажет серверы с наибольшим количеством сообщений.

1.  Создайте **Bar** диаграмму
2.  Источник: `syslog-*`
3.  **X-axis**:
    - **Aggregation**: `Terms`
    - **Field**: `hostname.keyword`
    - **Size**: `10`
4.  Нажмите **Update**
5.  Сохраните как `[Syslog] Top Hosts`

---

## Часть 4: Создание параметризованных дашбордов (OpenSearch 3.7)

В OpenSearch 3.7 появилась функция **Dashboard Variables**, которая позволяет создавать динамические дашборды с фильтрацией по переменным . Вместо создания десятков дублирующихся визуализаций для разных значений можно создать один параметризованный шаблон .

### 4.1. Создание дашборда с переменной (NGINX)

**Шаг 1: Создание дашборда**

1.  Перейдите в **Dashboards**
2.  Нажмите **Create new dashboard**
3.  Сохраните дашборд как `NGINX Analysis`

**Шаг 2: Создание переменной для фильтрации по статус-коду** 

1.  На дашборде нажмите **Add variable**
2.  Настройте параметры:
    - **Name**: `status_filter`
    - **Type**: `Query`
    - **Dataset**: `nginx-logs-*`
    - **Options Query**:
      ```
      source = nginx-logs-* | stats count() by response | fields response
      ```
3.  Нажмите **Preview** для проверки — должны отобразиться значения статусов
4.  Нажмите **Add variable**

**Шаг 3: Создание визуализации с использованием переменной**

1.  На дашборде нажмите **Create new** → **Add visualization**
2.  В редакторе запросов выберите **PPL**
3.  Введите запрос, использующий переменную `$status_filter`:
   ```
   source = nginx-logs-* | where response = $status_filter | stats count() by request.keyword
   ```
4.  Нажмите **Update**
5.  Теперь при изменении значения в выпадающем списке статусов визуализация автоматически обновляется 

### 4.2. Создание переменной для Syslog (по типу критичности)

**Шаг 1: Создание переменной**

1.  Создайте новый дашборд `Syslog Analysis`
2.  Нажмите **Add variable**
3.  Настройте **Custom variable**:
    - **Name**: `severity_level`
    - **Type**: `Custom`
    - **Values**: `info,warning,error,critical,debug`
4.  Нажмите **Add variable**

**Шаг 2: Использование переменной в визуализации**

1.  Добавьте новую визуализацию через **Add visualization**
2.  Используйте PPL-запрос:
   ```
   source = syslog-* | where severity = '$severity_level' | stats count() by hostname
   ```
3.  Нажмите **Update**

### 4.3. Создание составного дашборда

Объедините созданные визуализации на одном дашборде:

1.  Создайте новый дашборд `Unified Log Analytics`
2.  Нажмите **Add an existing** и добавьте все сохранённые визуализации
3.  Упорядочите панели, перетаскивая их и изменяя размеры 
4.  Настройте общий временной фильтр (например, "Last 60 minutes") 
5.  Сохраните дашборд

---

## Часть 5: Использование готовых интеграций

OpenSearch предоставляет готовые интеграции для NGINX, которые включают предварительно настроенные шаблоны индексов и дашборды . Это быстрый способ начать мониторинг без ручного создания визуализаций.

**Использование интеграции NGINX** :

1.  В левом меню перейдите в **Management** → **Integrations**
2.  В списке доступных найдите **NginX Dashboard**
3.  Нажмите **Try it** — система автоматически:
    - Создаст пример индекс-шаблона
    - Добавит пример данных
    - Создаст интеграцию на основе этих данных
4.  Просмотрите созданные визуализации и дашборды

---

## Часть 6: Экспорт и импорт визуализаций

OpenSearch использует **NDJSON** (Newline Delimited JSON) для экспорта и импорта сохранённых объектов . NDJSON представляет собой формат, где каждый JSON-объект находится на отдельной строке, что упрощает миграцию между средами и резервное копирование .

**Экспорт** :

1.  Перейдите в **Management** → **Saved objects**
2.  Найдите нужную визуализацию или дашборд
3.  Нажмите **Export** — файл NDJSON будет загружен

**Импорт** :

1.  Перейдите в **Management** → **Saved objects**
2.  Нажмите **Import**
3.  Выберите NDJSON файл
4.  Отметьте **Create new objects with random IDs**
5.  Нажмите **Import**

---

## Часть 7: Продвинутые визуализации через PPL

OpenSearch Dashboards позволяет создавать визуализации через запросы на языке **Piped Processing Language (PPL)** . Это даёт точный контроль над формой данных .

**Примеры PPL-запросов для визуализаций:**

**Анализ медленных запросов NGINX**:
```
source = nginx-logs-* | where bytes > 10000 | stats count() by request.keyword | sort - count()
```

**Распределение syslog-сообщений по часам** :
```
source = syslog-* | stats count() by SPAN(timestamp, 1h)
```

**Выявление неудачных логинов**:
```
source = syslog-* | where message LIKE '%Failed password%' | stats count() by hostname
```
