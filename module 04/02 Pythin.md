# Лабораторная работа: Работа с OpenSearch из Python


### 1. Подготовка окружения

Установите необходимые пакеты:

```bash
pip install opensearch-py pandas jupyter
```


### 2. Подключение клиента

```python
from opensearchpy import OpenSearch

client = OpenSearch(
    hosts=[{'host': 'localhost', 'port': 9200}],
    http_auth=('admin', 'StrongPassword123!'),
    use_ssl=True,
    verify_certs=False,
    ssl_show_warn=False
)
print(f"Connected: {client.ping()}")
```

---

## 📝 Часть 1. Создание индекса с векторным полем

**Задание:** Создать индекс `documents` с поддержкой k-NN для векторного поиска.

```python
index_body = {
    "settings": {
        "index": {
            "knn": True,
            "number_of_shards": 1,
            "number_of_replicas": 1
        }
    },
    "mappings": {
        "properties": {
            "title": {"type": "text"},
            "body": {"type": "text"},
            "embedding": {
                "type": "knn_vector",
                "dimension": 384,
                "method": {
                    "name": "hnsw",
                    "engine": "faiss",
                    "space_type": "innerproduct"
                }
            }
        }
    }
}

# Удаляем индекс, если существует
if client.indices.exists(index="documents"):
    client.indices.delete(index="documents")

# Создаём индекс
response = client.indices.create(index="documents", body=index_body)
print(response)
```


## 📤 Часть 2. Массовая загрузка данных 

**Задание:** Реализовать массовую загрузку данных с использованием `helpers.bulk`.

```python
from opensearchpy import helpers
import random

# Генерируем тестовые данные
documents = []
for i in range(100):
    documents.append({
        "id": f"doc-{i}",
        "title": f"Document {i} about OpenSearch",
        "body": f"This is the body of document number {i}",
        "embedding": [random.uniform(-1, 1) for _ in range(384)]
    })

def generate_actions(docs):
    for doc in docs:
        yield {
            "_op_type": "index",
            "_index": "documents",
            "_id": doc["id"],
            "_source": {
                "title": doc["title"],
                "body": doc["body"],
                "embedding": doc["embedding"]
            }
        }

# Загрузка
response = helpers.bulk(
    client,
    generate_actions(documents),
    chunk_size=50,
    request_timeout=60
)

client.indices.refresh(index="documents")
print(f"Загружено документов: {response[0]}")
```


## Часть 3. Построение поисковых запросов 

### 3.1 Match-запрос (полнотекстовый поиск)

**Задание:** Найти все документы, содержащие слово "OpenSearch" в поле `title`.

```python
query_body = {
    "query": {
        "match": {
            "title": {
                "query": "OpenSearch",
                "operator": "or"
            }
        }
    },
    "size": 10
}

response = client.search(index="documents", body=query_body)
for hit in response["hits"]["hits"]:
    print(f"Score: {hit['_score']} | {hit['_source']['title']}")
```

### 3.2 Multi-match запрос (поиск по нескольким полям)

**Задание:** Найти документы, где "OpenSearch" встречается в `title` или `body`.

```python
query_body = {
    "query": {
        "multi_match": {
            "query": "OpenSearch",
            "fields": ["title", "body"]
        }
    }
}

response = client.search(index="documents", body=query_body)
print(f"Найдено: {response['hits']['total']['value']} документов")
```

### 3.3 Range-запрос (поиск по диапазону)

**Задание:** Модифицируйте данные, добавив числовое поле `rating`, и найдите документы с рейтингом от 4.0 до 5.0.

```python
# Добавляем rating в данные
for i, doc in enumerate(documents):
    doc["rating"] = round(random.uniform(1, 5), 1)

# Создаём mapping с rating
# Затем выполняем range-запрос:
query_body = {
    "query": {
        "range": {
            "rating": {
                "gte": 4.0,
                "lte": 5.0
            }
        }
    }
}
```

---

##  Часть 4. Векторный поиск 

**Задание:** Выполнить поиск k ближайших соседей (k-NN) для заданного вектора.

```python
# Пример: ищем 5 документов, наиболее похожих на целевой вектор
query_vector = [random.uniform(-1, 1) for _ in range(384)]

query_body = {
    "size": 5,
    "query": {
        "knn": {
            "embedding": {
                "vector": query_vector,
                "k": 5
            }
        }
    }
}

response = client.search(index="documents", body=query_body)
for hit in response["hits"]["hits"]:
    print(f"ID: {hit['_id']} | Score: {hit['_score']}")
```

