---
layout: page
title: Установка через docker-compose
permalink: /installation/docker-compose/
parent: Установка
has_children: false
nav_order: 1
---
Описанный вариант установки является самым простым и удобным для систем: Windows/Linux/OSX. 
Для работы вам понадобятся:
- [git](https://git-scm.com/)
- [docker](https://www.docker.com/get-started/)
- [docker-compose](https://docs.docker.com/compose/)

{: .warning }
> Для OS Windows следующие шаги рекомендуется выполнять с использованием утилиты git bash. Она поставляется вместе с git.

## 1. Загрузка исходников с github
```bash 
git clone https://github.com/aeroideaservices/focus-demo.git
```
## 2 Запуск контейнеров
```bash 
docker-compose up -d
```
## 3. Восстановление БД keycloak из дампа
```bash
docker-compose exec -T postgres_keycloak psql -U keycloak < keycloak/dump_keycloak.sql
```
## 4. Загрузка демо данных
```bash
docker-compose exec -it dms1 /cli run_fixtures
```
## 5. Готово! Пробуем войти
Переходим по [ссылке](//localhost:3000)
- Логин: ```dms1```
- Пароль: ```dms1pass```

[Обучающее видео]({{"/#демонстрация-возможностей" | relative_url}}){: .btn .btn-purple }