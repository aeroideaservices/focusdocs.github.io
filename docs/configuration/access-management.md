---
layout: page
title: Настройка доступов
parent: Настройка
permalink: /configuration/access-management
---

# Настройка доступов

{: .note }
> Для доступа к функционалу Focus Admin используется jwt-токен. 
>
> Токен передается в заголовке запроса `Authorization` в виде “Bearer <jwt-token>“.
>
> Для неавторизованных пользователей Focus Admin недоступна.
>
> Авторизация происходит через KeyCloak.

### Синтаксис привилегий

Каждая строка привилегий включает в себя от 1 до 5 частей:

| Порядок | Часть привилегии | Синтаксис | Описание |
|----|----|----|----|
| 0. | Знак отрицания | `!` | Если ставится, то пользователь не будет иметь доступ к этой части |
| 1. | Код сервиса, обязательно | Код сервиса или `*` | В рамках cms для A+ код сервиса может быть `content` или `catalog` |
| 2. | Код плагина | Один из: `configurations`, `forms`, `mail-templates`, `media`, `menus`, `models` или `*` | К каким плагинам у пользователя есть доступ |
| 3. | URI, для которого настраивается доступ | Все `/` должны быть заменены точками. Если uri содержит переменную, вместо нее ставится либо `*`, либо конкретное значение этой переменной | Эта часть описывает,  к каким роутам и подроутам у пользователя есть доступ |
| 4. | Метод запроса | `get`, `post`, `put`, `patch`, `delete` | Самая детальная часть, описывает, какие методы доступны для конкретного роута |

Для того, чтобы указать доступ для всего, не обязательно прописывать права для каждого запроса, можно просто создать роль с правами `*`. Для того, чтобы 


{: .warning }
Для того, чтобы плагин был доступен пользователю, необходимо напрямую указать в привилегиях разрешение на этот плагин (например, `content.media` для доступа к плагину медиа в сервисе контент), либо указать привилегии на весь сервис, чтобы был доступ ко всем плагинам (например, `content` для доступа ко всем плагинам сервиса контент). Если указать доступ только на один метод внутри плагина, плагин будет все еще недоступен для пользователя.

### Примеры

```
{
  "all": [
    "*" // разрешение на все сервисы
  ],
  "content_only": [
    "content" // разрешение на весь сервис контент
  ],
  "content_without_media": [
    "content", // разрешение на весь сервис контент
    "!content.media" // запрет на плагин медиа в контенте
  ],
  "content_models_without_store_creating_and_deleting": [
    "content.models", // разрешение на плагин моделей
    "!content.models.stores.*.post", // запрет на создание модели ПВЗ
    "!content.models.stores.*.delete" // запрет на удаление модели ПВЗ
  ],
  "content_models_with_only_get_and_update_store": [
    "content.models", // разрешение на плагин моделей
    "!content.models.stores", // запрет на модель ПВЗ
    "content.models.stores.*.get", // разрешение на просмотр ПВЗ
    "content.models.stores.*.put"  // разрешение на обновление ПВЗ
  ],
  "custom": [
    "content.models", // разрешение на плагин моделей
    "!content.media", // запрет на плагин медиа - излишне, так как нет разрешающих привилегий
    "content.menus", // разрешение на плагин меню
    "!content.menus.*.delete", // запрет на удаление любых меню
    "!content.menus.*.menu-items*.delete", // запрет на удаление любых пунктов меню
    "content.configurations" // разрешение на плагин конфигураций для сервиса контент
  ],
}
```

## Добавление привилегий пользователю

Добавление правил происходит при помощи добавления пользовательских областей (Client Scopes). 
Название области должно соответствовать правилам, описанным выше. 

Внутри области добавлять параметры не нужно. 

После создания необходимо перейти во вкладку Области (Scopes) и добавить необходимые роли для этой области.

В клиенте эту область нужно указать как область по умолчанию.

![broken img]({{ "/img/configuration/access-management.png" | relative_url}})

## Структура JWT

В результате у пользователя с ролью, к которой была ассоциирована область, в payload части JWT токена в поле `scope` будет содержаться название этой области

```json
{
  "exp": 1668405511,
  "iat": 1668405211,
  "jti": "1d08b702-6d3e-442e-936b-64efa53767ab",
  "iss": "https://keycloak.farmperspektiva.aeroidea.ru/auth/realms/Test",
  "sub": "02057cbe-6eef-4918-bd0f-abc94755137f",
  "typ": "Bearer",
  "azp": "focus",
  "session_state": "401f86de-721e-43e4-85fe-adf4bcfe0f33",
  "acr": "1",
  "resource_access": {
    "focus": {
      "roles": [
        "focus admin"
      ]
    }
  },
  "scope": "services content catalog.models.store",
  "sid": "401f86de-721e-43e4-85fe-adf4bcfe0f33",
  "email_verified": false,
  "preferred_username": "focus-admin",
  "email": "focus@test.ru"
}
```


## Авторизация на фронте

Авторизация происходит по login / password

{: .note }
Регистрация пользователей происходит исключительно через интерфейс keyCloak, через интерфейс cms такой возможности нет.

## Запрос получения токена из keyCloak

```bash
curl --location --request POST 'https://{keycloak-endpoint}/auth/realms/{realm}/protocol/openid-connect/token' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'grant_type=password' \
--data-urlencode 'username={username}' \
--data-urlencode 'password={password}' \
--data-urlencode 'client_id={clientId}' \
--data-urlencode 'client_secret={clientSecret}'
```

Ответ сервиса:

```json
{
  "access_token":"{accessToken}",
  "expires_in":300,
  "refresh_expires_in":1800,
  "refresh_token":"{refreshToken}",
  "token_type":"Bearer",
  ...
}
```

## Запрос получения токена из keyCloak при помощи refresh токена

```bash
curl --location --request POST 'https://{keycloak-endpoint}/auth/realms/{realm}/protocol/openid-connect/token' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'grant_type=refresh_token' \
--data-urlencode 'refresh_token={refreshToken}' \
--data-urlencode 'client_id={clientId}' \
--data-urlencode 'client_secret={clientSecret}'
}'
```

Ответ сервиса:

```json
{
  "access_token":"{accessToken}",
  "expires_in":300,
  "refresh_expires_in":1800,
  "refresh_token":"{refreshToken}",
  "token_type":"Bearer",
  ...
}
```

