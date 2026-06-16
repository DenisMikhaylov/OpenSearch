## Лабораторная работа: Установка OpenSearch на Debian 13 с использованием DEB-пакета


### Требования к лабораторной работе
Для выполнения работы потребуется:
- Виртуальная машина или контейнер с Debian 13 (Trixie)
- Доступ в интернет для скачивания пакета
- Права суперпользователя (`sudo`)
- Минимум 2 ГБ оперативной памяти

### Ход выполнения работы

#### Часть 1. Подготовка системы

**1.1 Обновление системы**
Перед установкой необходимо обновить индекс пакетов и установить последние обновления безопасности:
```bash
sudo apt update && sudo apt upgrade -y
```

**1.2 Установка необходимых зависимостей**
Для работы OpenSearch и загрузки пакета установите следующие пакеты:
```bash
sudo apt install -y curl wget gnupg apt-transport-https ca-certificates
```

**1.3 Настройка системных параметров**
OpenSearch требует корректной настройки параметров ядра для работы в продакшн-среде. Настройте виртуальную память:
```bash
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

#### Часть 2. Установка OpenSearch из APT-репозитория

**2.1 Импорт GPG-ключа**
Для верификации подлинности пакетов импортируйте официальный GPG-ключ OpenSearch:
```bash
curl -o- https://artifacts.opensearch.org/publickeys/opensearch.pgp | sudo gpg --dearmor --batch --yes -o /usr/share/keyrings/opensearch-keyring
```
Данный ключ используется для проверки подписи репозитория APT .

**2.2 Добавление APT-репозитория**
Для установки последней версии OpenSearch добавьте репозиторий для ветки 3.x:
```bash
echo "deb [signed-by=/usr/share/keyrings/opensearch-keyring] https://artifacts.opensearch.org/releases/bundle/opensearch/3.x/apt stable main" | sudo tee /etc/apt/sources.list.d/opensearch-3.x.list
```

**2.3 Обновление индекса пакетов**
Обновите список доступных пакетов с новым репозиторием:
```bash
sudo apt update
```

**2.4 Установка OpenSearch**
Начиная с версии 2.12, установка требует задания пароля для встроенного администратора `admin` для настройки демонстрационной конфигурации безопасности . Задайте пароль (замените `<custom-admin-password>` на свой пароль) и установите пакет:
```bash
sudo env OPENSEARCH_INITIAL_ADMIN_PASSWORD=<custom-admin-password> apt-get install opensearch
```

**2.5 Проверка подлинности репозитория**
После установки проверьте, что GPG-ключ соответствует официальному:
```bash
gpg --no-default-keyring --keyring /usr/share/keyrings/opensearch-keyring --fingerprint
```
В выводе должен присутствовать отпечаток: `C5B7 4989 65EF D1C2 924B A9D5 39D3 1987 9310 D3FC` .

#### Часть 3. Альтернативная установка из DEB-пакета (ручная)

Если требуется установка без APT-репозитория, используйте следующий метод.

**3.1 Загрузка DEB-пакета**
Скачайте последнюю версию пакета для архитектуры x64:
```bash
curl -SLO https://artifacts.opensearch.org/releases/bundle/opensearch/3.2.0/opensearch-3.2.0-linux-x64.deb
```

**3.2 Установка пакета**
Установите пакет с указанием пароля администратора:
```bash
sudo env OPENSEARCH_INITIAL_ADMIN_PASSWORD=<custom-admin-password> dpkg -i opensearch-3.2.0-linux-x64.deb
```
> **Примечание:** Для установки версий 2.11 и ранее пароль не требовался и команда выглядела как `sudo dpkg -i opensearch-<version>-linux-x64.deb` .

**3.3 Верификация подписи (опционально)**
Для проверки целостности пакета скачайте файл подписи и проверьте его:
```bash
curl -SLO https://artifacts.opensearch.org/releases/bundle/opensearch/3.2.0/opensearch-3.2.0-linux-x64.deb.sig
curl -o- https://artifacts.opensearch.org/publickeys/opensearch.pgp | gpg --import -
gpg --verify opensearch-3.2.0-linux-x64.deb.sig opensearch-3.2.0-linux-x64.deb
```

#### Часть 4. Запуск и проверка работы

**4.1 Запуск службы**
Включите и запустите службу OpenSearch через systemd:
```bash
sudo systemctl enable opensearch
sudo systemctl start opensearch
```

**4.2 Проверка статуса**
Убедитесь, что служба работает корректно:
```bash
sudo systemctl status opensearch
```
Статус должен быть `active (running)`.

**4.3 Тестирование подключения**
Проверьте, что OpenSearch принимает запросы. Используйте `--insecure` для игнорирования самоподписанного SSL-сертификата:
```bash
curl -X GET https://localhost:9200 -u 'admin:<custom-admin-password>' --insecure
```
При успешном ответе вы получите JSON-объект с информацией о кластере :
```json
{
  "name": "hostname",
  "cluster_name": "opensearch",
  "version": {
    "distribution": "opensearch",
    "number": "3.2.0",
    ...
  },
  "tagline": "The OpenSearch Project: https://opensearch.org/"
}
```

**4.4 Просмотр установленных плагинов**
Выведите список активных плагинов:
```bash
curl -X GET https://localhost:9200/_cat/plugins?v -u 'admin:<custom-admin-password>' --insecure
```

