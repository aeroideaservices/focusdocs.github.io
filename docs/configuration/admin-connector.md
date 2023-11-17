---
layout: page
title: Подключение сервисов
parent: Настройка
permalink: /configuration/admin-connector
---
# Подключение сервисов к Focus Admin

## Назначение сервиса

Сервис admin-connector объединяет API разных сервисов под один эндпоинт и служит неким прокси для доступа к ним. 
Все методы сервиса, кроме метода `health-check`, требуют авторизации. 

Сервис умеет проверять права на совершение конкретных действий (запросов). Проверка прав происходит на основании привилегий, переданных в токене авторизации 

[больше про настройку привилегий]({{"/configuration/access-management" | relative_url}}){: .btn .btn-purple }

Также сервис раз в минуту делает проверку доступности подключенных сервисов, вызывая их `health-check методы`. 
В случае, если сервис не отвечает, при попытке обращения к нему будет возвращаться ошибка `503 Service Unavailable`.


{: .note }
Возможно, в будущем в сервисе будет подключена возможность управления пользователями и их привилегиями.

## Подключение нового сервиса

Для каждого нового сервиса добавляются свои переменные окружения


1. Добавить переменную окружения, в которой прописать значение вида ```{SERVICE_CODE};{SERVICE_NAME};{API_URL}```
2. В переменные окружения вида `PLUGINCODE_SERVICES` добавить через точку-запятую название переменной окружения из п.1
3. Перезапустить сервис `admin-connector`

Пример:

```bash
CATALOG_SERVICE=catalog;Каталог;https://catalog.example.service/api/v1/catalog
CONTENT_SERVICE=content;Контент;http://content.example.service/api/v1/content

# Определяем новую переменную для нового сервиса
NEW_SERVICE=<код сервиса>;<название сервиса>;<урл до апи>

# Добавляем определенную выше переменную в переменные ниже (соответственно плагинам в сервисе)
CONFIGURATIONS_SERVICES=CATALOG_SERVICE;CONTENT_SERVICE;NEW_SERVICE
FORM_SERVICES=CONTENT_SERVICE
MAIL_TEMPLATES_SERVICES=CONTENT_SERVICE;NEW_SERVICE
MEDIA_SERVICES=CATALOG_SERVICE;CONTENT_SERVICE;NEW_SERVICE
MENU_SERVICES=CONTENT_SERVICE
MODELS_SERVICES=CATALOG_SERVICE;CONTENT_SERVICE
```

### Если сервис не отображается в списке:
1. Проверить урл до корня апи с плагинами
2. Проверить работу метода проверки жизнеспособности (`{service-endpoint}}/health/check`)


