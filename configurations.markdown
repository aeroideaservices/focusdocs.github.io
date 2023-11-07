---
layout: page
title: Настройки
permalink: /configurations/
---

Примеры подключения всех плагинов есть в сервисе демо <https://gitlab.aeroidea.ru/internal-projects/focus-group/demo>

Все модули плагина находятся тут <https://gitlab.aeroidea.ru/internal-projects/focus/-/tree/develop/configurations>

## Возможности

Плагин конфигураций позволяет создавать конфигурации и их настройки. Плагин может работать как вместе плагином медиа, так и отдельно от него.

## Установка

```bash
go get gitlab.aeroidea.ru/internal-projects/focus/configurations/plugin
go get gitlab.aeroidea.ru/internal-projects/focus/configurations/postgres
go get gitlab.aeroidea.ru/internal-projects/focus/configurations/rest
```

* `configurations/plugin` - основной плагин
* `configurations/postgres` - адаптер для работы с базой данных postgres
* `configurations/rest` - адаптер, реализующий REST API

Любой из адаптеров можно заменить своей реализацией. Важно, чтобы реализация была совместима с интерфейсами плагина.

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

Определить переменные, относящиеся к плагину конфигураций (в примере определяется хендлер с публичным методом получения настроек по коду конфигурации, не обязательно)

```go
// https://gitlab.aeroidea.ru/internal-projects/focus-group/demo/-/blob/develop/internal/infrastructure/registry/services_definitions/configurations.go#L15
var ConfigurationsDefinitions = appendArr([]di.Def{
	{
		Name: "optionsHandler",
		Build: func(ctn di.Container) (interface{}, error) {
			options := ctn.Get("focus.configurations.actions.options").(*actions.Options)
			validator := ctn.Get("focus.validator").(*validation.Validator)
			return handlers.NewOptionsHandler(options, validator), nil
		},
	},
}, plugin.Definitions, postgres.Definitions, rest.Definitions)
```

Если нужно, определить переводы к ошибкам

```go
// https://gitlab.aeroidea.ru/internal-projects/focus-group/demo/-/blob/develop/internal/infrastructure/registry/services_definitions/translations/translations.go
var ErrTranslations = []et.Translation{
	{
		Tag:         "conf.exists",
		Translation: "Конфигурация с таким символьным кодом уже существует.",
	},
	{
		Tag:         "opt.exists",
		Translation: "Настройка с таким символьным кодом уже существует.",
	},
	{
		Tag:         "opt.linked-to-another",
		Translation: "Настройка привязана к другой конфигурации.",
	},
	{
		Tag:         "field-not-updatable",
		Translation: "Поле {0} нельзя обновлять.",
	},
}
```

Регистрация сервисов в контейнере

```go
// https://gitlab.aeroidea.ru/internal-projects/focus-group/demo/-/blob/develop/internal/infrastructure/registry/container.go#L65
	if err = builder.Add(services_definitions.ConfigurationsDefinitions...); err != nil {
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
	r.focusConfigurationsRouter.SetRoutes(focus)
	...

	return router
}
```

## Настройка

Для того, чтобы в плагине конфигураций можно было использовать настройки типа медиа, необходимо подключить [Плагин Медиа](https://wiki.aeroidea.ru/doc/plagin-media-4XjOy3JqLK).

## Примеры использования

Примеры использования можно найти в сервисе демо <https://gitlab.aeroidea.ru/internal-projects/focus-group/demo>
