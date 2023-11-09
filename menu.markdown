---
layout: page
title: Меню
permalink: /menu/
---

![broken img]({{ "/img/menu-preview.png" | relative_url}})

## Возможности

Плагин позволяет создавать и настраивать меню с различными уровнями пунктов меню, настраивать относительные или полные ссылки для каждого пункта меню, управлять порядком пунктов меню. Также для каждого пункта меню можно добавлять любые дополнительные параметры.

## Установка

* `menu/plugin` - основной плагин

```bash
go get gitlab.aeroidea.ru/internal-projects/focus/menu/plugin
```

* `menu/postgres` - адаптер для работы с базой данных postgres

```bash
go get gitlab.aeroidea.ru/internal-projects/focus/menu/postgres
```

* `menu/rest` - адаптер, реализующий REST API

```bash
go get gitlab.aeroidea.ru/internal-projects/focus/menu/rest
```

{: .note-title }
> Кастомизация
>
> Любой из адаптеров можно заменить своей реализацией. Важно, чтобы реализация была совместима с интерфейсами плагина.

Определить основные переменные фокуса

```go
// https://gitlab.aeroidea.ru/internal-projects/focus-group/demo/-/blob/develop/internal/infrastructure/registry/services_definitions/focus.go#L17
var FocusDefinitions = []di.Def{
	{
		Name: "focus.logger",
		Build: func(ctn di.Container) (interface{}, error) {
			logger := ctn.Get("logger").(*zap.SugaredLogger)
			return logger, nil
		},
	},
	{
		Name: "focus.validator",
		Build: func(ctn di.Container) (interface{}, error) {
			universalTranslator := ctn.Get("focus.universalTranslator").(*ut.UniversalTranslator)
			return validation.NewValidator(universalTranslator), nil
		},
	},
	{
		Name: "focus.errorHandler",
		Build: func(ctn di.Container) (interface{}, error) {
			logger := ctn.Get("logger").(*zap.SugaredLogger)
			errTrans := ctn.Get("focus.errorTranslator").(*et.Translator)
			errHandler := middleware.NewErrorHandler(logger).SetTranslator(errTrans)

			return errHandler, nil
		},
	},
	{
		Name: "focus.db",
		Build: func(ctn di.Container) (interface{}, error) {
			db := ctn.Get("db").(*gorm.DB)
			return db, nil
		},
	},
	{
		Name: "focus.universalTranslator",
		Build: func(ctn di.Container) (interface{}, error) {
			russian := ru.New()
			utTranslator := ut.New(russian, russian)

			return utTranslator, nil
		},
	},
	{
		Name: "focus.errorTranslator",
		Build: func(ctn di.Container) (interface{}, error) {
			uTranslator := ctn.Get("focus.universalTranslator").(*ut.UniversalTranslator)
			ru, _ := uTranslator.GetTranslator("ru")
			etTranslator := et.New(ru)

			if err := etTranslator.AddTranslation(translations.ErrTranslations...); err != nil {
				return nil, err
			}

			return etTranslator, nil
		},
	},
}
```

Определить переменные, относящиеся к плагину меню

```go
// https://gitlab.aeroidea.ru/internal-projects/focus-group/demo/-/blob/develop/internal/infrastructure/registry/services_definitions/menu.go#L11
var MenuDefinitions = appendArr([]di.Def{
	{
		Name: "focus.menu.maxMenuItemsDepth",
		Build: func(ctn di.Container) (interface{}, error) {
			return 10, nil
		},
	},
}, plugin.Definitions, postgres.Definitions, rest.Definitions)
```

Если нужно, определить переводы к ошибкам

```go
// https://gitlab.aeroidea.ru/internal-projects/focus-group/demo/-/blob/develop/internal/infrastructure/registry/services_definitions/translations/translations.go
var ErrTranslations = []et.Translation{
	{
		Tag:         "menu.conflict",
		Translation: "Меню с таким кодом уже существует.",
	},
	{
		Tag:         "menu-item.position-too-large",
		Translation: "Позиция пункта меню слишком большая.",
	},
	{
		Tag:         "menu-item.max-depth-exceeded",
		Translation: "Достигнута максимальная вложенность пунктов меню.",
	},
	{
		Tag:         "menu.not-found",
		Translation: "Меню не найдено.",
	},
	{
		Tag:         "menu-item.not-found",
		Translation: "Пункт меню не найден.",
	},
}
```

Регистрация сервисов в контейнере

```go
// https://gitlab.aeroidea.ru/internal-projects/focus-group/demo/-/blob/develop/internal/infrastructure/registry/container.go#L65
	if err = builder.Add(services_definitions.MenuDefinitions...); err != nil {
		return nil, err
	}
```

Подключение роутов

```go
// https://gitlab.aeroidea.ru/internal-projects/focus-group/demo/-/blob/develop/internal/adapters/rest/router.go#L102
func (r Router) Router() *gin.Engine {
	...

	// cms focus
	focus := v1.Group(r.apiSettings.FocusPath)
	focus.Group("health").Group("check").GET("", healthCheck)
	...
	r.focusMenuRouter.SetRoutes(focus)
	...

	return router
}
```

## Настройка

`focus.menu.maxMenuItemsDepth` - максимальная вложенность пунктов меню

## Дополнительно

[Примеры](https://gitlab.aeroidea.ru/internal-projects/focus-group/demo){: .btn .btn-purple }
[Все модули](https://gitlab.aeroidea.ru/internal-projects/focus/-/tree/develop/menu){: .btn .btn-blue }