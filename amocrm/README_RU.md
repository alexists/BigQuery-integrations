# Интеграция Google BigQuery и [amoCRM](https://www.amocrm.ru/)

## Общая информация

Модуль **amocrm-bq-integration** предназначен для автоматизации передачи данных из amoCRM в Google BigQuery с помощью Google Cloud функции.
В настоящее время модуль позволяет импортировать из amoCRM в Google BigQuery такие сущности, как: account, leads, customers, contacts,
companies, catalogs, catalog_elements, customers_periods.

## Принцип работы

С помощью HTTP POST запроса вызывается Cloud-функция, которая получает данные из amoCRM, используя [amoCRM API](https://www.amocrm.ru/developers/content/platform/abilities/) и загружает их в соответствующие таблицы Google BigQuery.
Если таблица уже существует в выбранном датасете, то она будет перезаписана. 

## Требования

- проект в Google Cloud Platform с активированным биллингом;
- доступ к amoCRM;
- доступ на редактирование *WRITER* к датасету и выполнение заданий (роль *Пользователь заданий BigQuery*) для сервисного аккаунта Cloud-функции в проекте BigQuery, куда будет загружена таблица (см. раздел [Доступы](https://github.com/OWOX/BigQuery-integrations/blob/master/amocrm/README_RU.md#Доступы));
- HTTP-клиент для выполнения POST запросов, вызывающих Cloud-функцию.

## Настройка 

1. Перейдите в [Google Cloud Platform Console](https://console.cloud.google.com/home/dashboard/) и авторизуйтесь с помощью Google аккаунта, или зарегистрируйтесь, если аккаунта еще нет.
2. Перейдите в проект с активированным биллингом или [создайте](https://cloud.google.com/billing/docs/how-to/manage-billing-account#create_a_new_billing_account) новый биллинг аккаунт для проекта.
3. Перейдите в раздел [Cloud Functions](https://console.cloud.google.com/functions/) и нажмите **СОЗДАТЬ ФУНКЦИЮ**. Обратите внимание, что за использование Cloud-функций взимается [плата](https://cloud.google.com/functions/pricing).
4. Заполните следующие поля:

    **Название**: *amocrm-bq-integration* или любое другое подходящее название;

    **Выделенный объем памяти**: *2 ГБ* или меньше, в зависимости от размера обрабатываемого файла;

    **Триггер**: *HTTP*;

    **Исходный код**: *Встроенный редактор*;

    **Среда выполнения**: Python 3.X.
5. Скопируйте содержимое файла **main.py** в встроенный редактор, вкладка *main.py*.
6. Скопируйте содержимое файла **requirements.txt** в встроенный редактор, вкладка *requirements.txt*.
7. В качестве **вызываемой функции** укажите *amocrm*. 
8. В дополнительных параметрах увеличьте **время ожидания** с *60 сек.* до *540 сек.* или меньшее, в зависимости от размеров обрабатываемого файла.
9. Завершите создание Cloud-функции, нажав на кнопку **Создать**. 

## Доступы

### amoCRM

Получите в amoCRM уникальный ключ для API.

Для этого переходим в Настройки → API → Ваш API ключ. 
Этот ключ будет использоваться в качестве значения для параметра **apiKey**.

Вся работа через API происходит с учетом прав доступа пользователя в аккаунте.


### BigQuery

Если проект в [Google Cloud Platform Console](https://console.cloud.google.com/home/dashboard/), в котором расположена Cloud-функция и проект в BigQuery — одинаковые — никаких дополнительных шагов не требуется.

В случае, если проекты разные:
1. Перейдите в раздел [Cloud Functions](https://console.cloud.google.com/functions/) и кликните по только что созданной функции для того, чтобы открыть окно **Сведения о функции**.
2. На вкладке **Общие** найдите поле *Сервисный аккаунт* и скопируйте указанный email.
3. В Google Cloud Platform перейдите в IAM и администрирование - [IAM](https://console.cloud.google.com/iam-admin/iam) и выберите проект, в который будет загружена таблица в BigQuery. 
4. **Добавьте участника** - скопированный email и укажите для него роль - *Пользователь заданий BigQuery*. Сохраните участника.
5. Перейдите к датасету в проекте BigQuery и выдайте доступ на редактирование для сервисного аккаунта. Необходимо [предоставить](https://cloud.google.com/bigquery/docs/datasets#controlling_access_to_a_dataset) *WRITER* доступ к датасету.

## Использование

1. Перейдите в раздел [Cloud Functions](https://console.cloud.google.com/functions/) и кликните по только что созданной функции для того, чтобы открыть окно **Сведения о функции**.
2. На вкладке **Триггер** скопируйте *URL-адрес*. 
С помощью HTTP-клиента отправьте POST запрос по этому *URL-адресу*. Тело запроса должно быть в формате JSON:
```
{
    "amocrm": {
        "user": "login@amocrm",
        "apiKey": "amoCRM API key",
        "apiAddress": "https://example.amocrm.ru",
        "entities": [
            "leads",
            "contacts",
            "customers",
            "companies",
            "catalogs",
            "catalog_elements",
            "account",
            "customers_periods"
        ]
    },
    "bq": {
      "project_id": "YOUR GCP PROJECT",
      "dataset_id": "YOUR DATASET NAME",
      "location": "US"
   }
}
```

| Параметр | Объект | Описание |
| --- | --- | --- |
| Обязательные параметры |  
| user | amocrm | Логин пользователя - e-mail.| 
| apiKey | amocrm | amoCRM API-ключ. (см. раздел [Доступы](https://github.com/OWOX/BigQuery-integrations/blob/master/amocrm/README_RU.md#Доступы)).|
| apiAddress | amocrm | API адресс: имя хоста для пользователя в системе. Напрмиер, если текущей страницей в системе является https://example.amocrm.ru/dashboard/, то в качестве адресса необходимо указать https://example.amocrm.ru/ |
| entities | amocrm | Список сущностей в amoCRM, которые необходимо импортировать в BigQuery. В примере приведен полный список возможных значений. |
| project_id | bq | Название проекта в BigQuery, куда будет загружена таблица. Проект может отличаться от того, в котором создана Cloud-функция. |
| dataset_id | bq | Название датасета в BigQuery, куда будет загружена таблица. |
| Опциональные параметры |
| location | bq | Географическое расположение таблицы. По умолчанию указан “US”. |

## Работа с ошибками

Каждое выполнение Cloud-функции логируется. Логи можно посмотреть в Google Cloud Platform:

1. Перейдите в раздел [Cloud Functions](https://console.cloud.google.com/functions/) и кликните по только что созданной функции для того, чтобы открыть окно **Сведения о функции**.
2. Кликнете **Посмотреть журналы** и посмотрите наиболее новые записи на уровне *Ошибки, Критические ошибки*.

## Ограничения

1. Размер обрабатываемого файла не должен превышать 2 ГБ.
2. Время выполнения Cloud-функции не может превышать 540 сек.

В случае превышения одного из лимитов вы можете запускать функцию отдельно для каждой сущности в amoCRM. 
Таким образом, чтобы получить данные для leads и contacts — вам необходимо выполнить функцию два раза. Первый раз в entities необходимо передать "leads", а при втором вызове - "contacts".
При первом вызове entities будет таким:
```
     "entities": [
        "leads"
      ]
```
При втором:
```
     "entities": [
        "contacts"
      ]
```

## Пример использования

### Linux 

Вызовите функцию через терминал Linux:

```
curl -X POST https://REGION-PROJECT_ID.cloudfunctions.net/amocrm/ -H "Content-Type:application/json"  -d 
 '{
    "amocrm": {
        "user": "login@amocrm",
        "apiKey": "amoCRM API key",
        "apiAddress": "https://example.amocrm.ru",
        "entities": [
            "leads",
            "contacts",
            "customers",
            "companies",
            "catalogs",
            "catalog_elements",
            "account",
            "customers_periods"
        ]
    },
    "bq": {
      "project_id": "YOUR GCP PROJECT",
      "dataset_id": "YOUR DATASET NAME",
      "location": "US"
   }
}'
```

### Python

```
from urllib import urlencode
from httplib2 import Http

trigger_url = "https://REGION-PROJECT_ID.cloudfunctions.net/expertsender/"
headers = { "Content-Type": "application/json" }
payload = {
    "amocrm": {
        "user": "login@amocrm",
        "apiKey": "amoCRM API key",
        "apiAddress": "https://example.amocrm.ru",
        "entities": [
            "leads",
            "contacts",
            "customers",
            "companies",
            "catalogs",
            "catalog_elements",
            "account",
            "customers_periods"
        ]
    },
    "bq": {
      "project_id": "YOUR GCP PROJECT",
      "dataset_id": "YOUR DATASET NAME",
      "location": "US"
   }
}
Http().request(trigger_url, "POST", urlencode(payload), headers = headers)
```

### [Google Apps Script](https://developers.google.com/apps-script/)

Вставьте следующий код со своими параметрами и запустите функцию:

```
function runAmoCrm() {
  trigger_url = "https://REGION-PROJECT_ID.cloudfunctions.net/amocrm/"
  payload = {
    "amocrm": {
        "user": "login@amocrm",
        "apiKey": "amoCRM API key",
        "apiAddress": "https://example.amocrm.ru",
        "entities": [
            "leads",
            "contacts",
            "customers",
            "companies",
            "catalogs",
            "catalog_elements",
            "account",
            "customers_periods"
        ]
    },
    "bq": {
      "project_id": "YOUR GCP PROJECT",
      "dataset_id": "YOUR DATASET NAME",
      "location": "US"
   }
};

  var options = {
        "method": "post",
        "contentType": "application/json",
        "payload": JSON.stringify(payload)
  };

  var request = UrlFetchApp.fetch(trigger_url, options);
}
```
