##  Лабораторная работа: Настройка LDAP-аутентификации в OpenSearch


### Среда выполнения

*   **ОС хоста:** Debian 13.
*   **Контейнеризация:** Docker и Docker Compose.
*   **Стек:**
    *   **OpenSearch** (образ `opensearchproject/opensearch:latest`) и **OpenSearch Dashboards** (образ `opensearchproject/opensearch-dashboards:latest`) на одном хосте .
    *   **LDAP-сервер** (образ `sourcemation/openldap:latest`) на отдельной виртуальной машине Debian 13 .


2.  **Установите Docker и Docker Compose** на обеих машинах, следуя официальной документации.

### Часть 1. Установка и настройка LDAP-сервера (на отдельной ВМ, склонированная машина Debian13)

Эта часть выполняется на выделенной виртуальной машине Debian 13, которая будет выступать в роли централизованного поставщика учетных данных.

#### Шаг 1. Установка пакетов

Установите OpenLDAP (slapd) и утилиты командной строки для работы с LDAP :

```bash
sudo apt update
sudo apt install -y slapd ldap-utils
```

В процессе установки будет предложено задать пароль администратора LDAP. Запомните его (в примере будет использоваться пароль `admin@1234`) .

#### Шаг 2. Настройка базового домена

Создайте структуру для хранения пользователей и групп. Создайте файл `base.ldif` :

```bash
nano base.ldif
```

Добавьте в него следующее содержимое, заменив `dc=corp,dc=local` на ваш собственный домен (например, `dc=example,dc=com`):

```
dn: ou=people,dc=corp,dc=local
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=corp,dc=local
objectClass: organizationalUnit
ou: groups
```

Примените изменения, используя пароль администратора LDAP :

```bash
sudo ldapadd -x -D cn=admin,dc=corp,dc=local -W -f base.ldif
```

#### Шаг 3. Наполнение LDAP тестовыми пользователями и группами

Создайте файл `users.ldif`, который добавит двух пользователей (`osdev1`, `osadmin1`) и две группы (`Developers`, `Administrators`). Замените `dc=corp,dc=local` на ваш домен.

```bash
nano users.ldif
```

Содержимое файла :

```
# Пользователь osdev1
dn: cn=osdev1,ou=people,dc=corp,dc=local
objectClass: person
objectClass: inetOrgPerson
objectClass: organizationalPerson
cn: osdev1
sn: osdev1
givenName: osdev1
mail: osdev1@stack.com
uid: 1001
userPassword: osdev1

# Пользователь osadmin1
dn: cn=osadmin1,ou=people,dc=corp,dc=local
objectClass: person
objectClass: inetOrgPerson
objectClass: organizationalPerson
cn: osadmin1
sn: osadmin1
givenName: osadmin1
mail: osadmin1@stack.com
uid: 1003
userPassword: osadmin1

# Группа Developers (включает osdev1)
dn: cn=Developers,ou=groups,dc=corp,dc=local
objectClass: groupOfNames
cn: Developers
member: cn=osdev1,ou=people,dc=corp,dc=local

# Группа Administrators (включает osadmin1)
dn: cn=Administrator,ou=groups,dc=corp,dc=local
objectClass: groupOfNames
cn: Administrator
member: cn=osadmin1,ou=people,dc=corp,dc=local
```

Примените изменения:

```bash
sudo ldapadd -x -D cn=admin,dc=corp,dc=local -W -f users.ldif
```

#### Шаг 4. Проверка структуры LDAP

Убедитесь, что данные добавлены корректно:

```bash
sudo ldapsearch -x -b "dc=corp,dc=local" "(objectClass=*)"
```
В выводе вы должны увидеть созданные записи .

### Часть 2. Настройка OpenSearch и OpenSearch Dashboards (на основной ВМ)

Эта часть выполняется на основной виртуальной машине, где будут работать сервисы OpenSearch.




#### Шаг 1. Конфигурация LDAP через API

Вместо ручного редактирования YAML-файлов, используйте API OpenSearch для применения настроек безопасности. Выполните PUT-запрос к `_plugins/_security/api/securityconfig/config` .

**Важно:** Замените `"10.0.XXX.XXX:389"` на реальный IP-адрес вашей ВМ с LDAP-сервером.

```bash
curl -X PUT "https://localhost:9200/_plugins/_security/api/securityconfig/config" \
     -u 'admin:MyStrongPassword123!' \
     -k \
     -H 'Content-Type: application/json' \
     -d '{
  "dynamic": {
    "authc": {
      "basic_internal_auth_domain": {
        "description": "Authenticate via HTTP Basic against internal users database",
        "http_enabled": true,
        "transport_enabled": true,
        "order": 0,
        "http_authenticator": {
          "type": "basic",
          "challenge": true
        },
        "authentication_backend": {
          "type": "intern"
        }
      },
      "ldap": {
        "description": "Authenticate via LDAP",
        "http_enabled": true,
        "transport_enabled": true,
        "order": 1,
        "http_authenticator": {
          "type": "basic",
          "challenge": false
        },
        "authentication_backend": {
          "type": "ldap",
          "config": {
            "enable_ssl": false,
            "hosts": ["<IP-АДРЕС-LDAP-СЕРВЕРА>:389"],
            "bind_dn": "cn=admin,dc=corp,dc=local",
            "password": "admin@1234",
            "userbase": "ou=people,dc=corp,dc=local",
            "usersearch": "(cn={0})",
            "username_attribute": "cn"
          }
        }
      }
    },
    "authz": {
      "ldap_authz": {
        "description": "Authorize via LDAP",
        "http_enabled": true,
        "transport_enabled": true,
        "authorization_backend": {
          "type": "ldap",
          "config": {
            "enable_ssl": false,
            "hosts": ["<IP-АДРЕС-LDAP-СЕРВЕРА>:389"],
            "bind_dn": "cn=admin,dc=corp,dc=local",
            "password": "admin@1234",
            "rolebase": "ou=groups,dc=corp,dc=local",
            "rolesearch": "(member={0})",
            "rolename": "cn",
            "userbase": "ou=people,dc=corp,dc=local",
            "usersearch": "(cn={0})"
          }
        }
      }
    }
  }
}'
```


#### Шаг 2. Создание ролей и отображение (Role Mapping)

Теперь нужно связать LDAP-группы с правами в OpenSearch .

1.  Откройте OpenSearch Dashboards в браузере по адресу `http://<IP-ОСНОВНОЙ-ВМ>:5601`.
2.  Авторизуйтесь как `admin` с паролем `MyStrongPassword123!`.
3.  Перейдите в **Security** -> **Roles**.
4.  Нажмите **Create role**:
    *   **Name:** `ldap_admin_role`
    *   **Cluster Permissions:** `cluster_all` (для теста)
    *   **Index Permissions:** Добавьте `*` и отметьте `*` в `Permissions`.
    *   Нажмите **Create**.
5.  Нажмите на созданную роль `ldap_admin_role`.
6.  Перейдите на вкладку **Mapped users** -> **Manage mapping**.
7.  В поле **Backend roles** введите `Administrator` (имя группы из LDAP).
8.  Нажмите **Map**.

### Часть 3. Проверка работоспособности

1.  **Проверка аутентификации:**
    *   Попробуйте зайти в OpenSearch Dashboards как `osadmin1` (пароль: `osadmin1`). Вы должны увидеть панель управления с полными правами (если сопоставили с `ldap_admin_role`).
    *   Попробуйте зайти как `osdev1` (пароль: `osdev1`). Права должны быть ограничены (если сопоставили с `ldap_dev_role`).

2.  **Проверка авторизации через API:**
    *   От имени `osadmin1` можно создать индекс:
        ```bash
        curl -XPUT "https://localhost:9200/test-index" -u 'osadmin1:osadmin1' -k
        ```
        Ожидаемый ответ: `{"acknowledged":true}`.
    *   От имени `osdev1` попытка создания индекса должна завершиться ошибкой `403 Forbidden`:
        ```bash
        curl -XPUT "https://localhost:9200/test-index" -u 'osdev1:osdev1' -k
        ```

| **OpenSearch не видит LDAP-сервер** | Убедитесь, что сеть между контейнерами OpenSearch и ВМ с LDAP настроена (проверьте `ping` и доступность порта `389`) . |
| **Ошибка при применении конфигурации через API** | Проверьте синтаксис JSON. Убедитесь, что пароль администратора закодирован правильно, если содержит спецсимволы. |
