## Лабораторная работа: Установка OpenSearch Dashboards на Debian 13 с использованием DEB-пакета


**Важное требование:** Для работы OpenSearch Dashboards необходим запущенный и настроенный кластер OpenSearch с включенной безопасностью .

### Требования к лабораторной работе
Для выполнения работы потребуется:
- Виртуальная машина или контейнер с Debian 13 (Trixie)
- Предварительно установленный и запущенный OpenSearch (см. предыдущую лабораторную работу)
- Доступ в интернет для скачивания пакета
- Права суперпользователя (`sudo`)
- Минимум 2 ГБ оперативной памяти

### Ход выполнения работы

#### Часть 1. Подготовка системы

**1.1 Обновление системы**
```bash
sudo apt update && sudo apt upgrade -y
```

**1.2 Установка необходимых зависимостей**
Установите пакеты, необходимые для загрузки и установки:
```bash
sudo apt install -y curl wget gnupg apt-transport-https ca-certificates
```

**1.3 Проверка доступности OpenSearch**
Убедитесь, что OpenSearch запущен и доступен по адресу `https://localhost:9200` с сертификатами самоподписи:
```bash
curl -X GET https://localhost:9200 -u 'admin:пароль_администратора' --insecure
```
Если вы забыли пароль администратора, установленный при настройке OpenSearch, вам потребуется его сбросить или переустановить OpenSearch.

#### Часть 2. Установка OpenSearch Dashboards из APT-репозитория

**2.1 Импорт GPG-ключа для верификации пакетов**
```bash
curl -fsSL https://artifacts.opensearch.org/publickeys/opensearch.pgp | sudo gpg --dearmor -o /usr/share/keyrings/opensearch.gpg
```
Этот ключ необходим для проверки подлинности пакетов из репозитория .

**2.2 Добавление APT-репозитория**
Для установки последней версии добавьте репозиторий OpenSearch Dashboards. Для ветки 3.x используйте команду :
```bash
echo "deb [signed-by=/usr/share/keyrings/opensearch.gpg] https://artifacts.opensearch.org/releases/bundle/opensearch-dashboards/3.x/apt stable main" | sudo tee /etc/apt/sources.list.d/opensearch-dashboards-3.x.list
```

**2.3 Обновление индекса пакетов и установка**
```bash
sudo apt update
sudo apt install -y opensearch-dashboards
```
Это установит последнюю доступную версию из репозитория .

#### Часть 3. Альтернативная установка из DEB-пакета (ручная)

**3.1 Загрузка DEB-пакета**
Скачайте последнюю версию пакета для архитектуры x64 с сайта загрузок OpenSearch. Для версии 3.1.0 команда будет выглядеть так :
```bash
curl -SLO https://artifacts.opensearch.org/releases/bundle/opensearch-dashboards/3.1.0/opensearch-dashboards-3.1.0-linux-x64.deb
```

**3.2 Установка пакета**
Установите скачанный пакет с помощью `dpkg` :
```bash
sudo dpkg -i opensearch-dashboards-3.1.0-linux-x64.deb
```
> **Примечание:** В отличие от установки OpenSearch, для OpenSearch Dashboards не требуется задавать пароль администратора на этапе установки пакета.

**3.3 Верификация подписи пакета (опционально)**
Для проверки целостности пакета скачайте файл подписи и проверьте его :
```bash
curl -SLO https://artifacts.opensearch.org/releases/bundle/opensearch-dashboards/3.1.0/opensearch-dashboards-3.1.0-linux-x64.deb.sig
curl -o- https://artifacts.opensearch.org/publickeys/opensearch.pgp | gpg --import -
gpg --verify opensearch-dashboards-3.1.0-linux-x64.deb.sig opensearch-dashboards-3.1.0-linux-x64.deb
```

#### Часть 4. Настройка и запуск OpenSearch Dashboards

**4.1 Базовая настройка конфигурации**
Отредактируйте основной файл конфигурации `/etc/opensearch-dashboards/opensearch_dashboards.yml`:
```bash
sudo nano /etc/opensearch-dashboards/opensearch_dashboards.yml
```

Добавьте или измените следующие параметры :
```yaml
# Адрес интерфейса для доступа извне (0.0.0.0 - все интерфейсы)
server.host: "0.0.0.0"
# Порт веб-интерфейса (по умолчанию 5601, но можно изменить)
server.port: 5601

# Адрес OpenSearch кластера (с https, т.к. используется демонстрационная безопасность)
opensearch.hosts: ["https://localhost:9200"]
# Отключаем проверку SSL сертификата для самоподписанных сертификатов
opensearch.ssl.verificationMode: none

# Учетные данные администратора для подключения к OpenSearch
opensearch.username: "admin"
opensearch.password: "пароль_администратора_установленный_при_установке_opensearch"
```

**4.2 Запуск службы**
Перезагрузите конфигурацию systemd, включите и запустите службу :
```bash
sudo systemctl daemon-reload
sudo systemctl enable opensearch-dashboards
sudo systemctl start opensearch-dashboards
```

**4.3 Проверка статуса службы**
Убедитесь, что служба работает корректно:
```bash
sudo systemctl status opensearch-dashboards
```
Статус должен быть `active (running)`.

#### Часть 5. Проверка работы и устранение неполадок

**5.1 Доступ к веб-интерфейсу**
Откройте в браузере на хост-машине или на сервере адрес:
```
http://<IP-адрес_вашего_сервера>:5601
```
Вы должны увидеть страницу входа OpenSearch Dashboards. Введите логин `admin` и пароль, который вы установили при настройке OpenSearch .

**5.2 Просмотр логов при ошибках**
Если веб-интерфейс не открывается или возникают ошибки, проверьте логи:
```bash
sudo journalctl -u opensearch-dashboards -f
sudo tail -f /var/log/opensearch-dashboards/dashboards.log
```
Путь к логу может отличаться в зависимости от настроек. Убедитесь, что в файле конфигурации правильно указан параметр `logging.dest` .

**5.3 Проверка подключения к OpenSearch**
Если Dashboards не подключается к OpenSearch, проверьте настройки `opensearch.hosts`, `opensearch.username` и `opensearch.password`. Также убедитесь, что параметр `opensearch.ssl.verificationMode` установлен в `none` для работы с самоподписанными сертификатами .


### Список использованных источников
1. OpenSearch Documentation. Installing OpenSearch Dashboards (Debian). https://opensearch.org/docs/latest/install-and-configure/install-dashboards/debian/ 
2. tutorialforlinux.com. How to Install OpenSearch Dashboards on Debian 13 (Complete Setup Guide). 
3. mka.in. Opensearch fluentd stack. 
4. OpenSearch Forum. Debian/Ubuntu installer please? 
5. opensearch.org.cn. Installing OpenSearch Dashboards (Debian). 
