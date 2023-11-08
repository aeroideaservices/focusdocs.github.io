---
layout: page
title: Медиа
permalink: /media/
---

{: .note }
Примеры подключения всех плагинов есть в сервисе демо <https://gitlab.aeroidea.ru/internal-projects/focus-group/demo>

{: .note }
Все модули плагина находятся тут <https://gitlab.aeroidea.ru/internal-projects/focus/-/tree/develop/media>



## Возможности

Плагин реализует работу с файловыми хранилищами, например, s3-совместимые хранилища. Плагин позволяет управлять папками и файлами: создавать, переименовывать, перемещать и удалять.

## Установка

```bash
go get gitlab.aeroidea.ru/internal-projects/focus/media/plugin
go get gitlab.aeroidea.ru/internal-projects/focus/media/postgres
go get gitlab.aeroidea.ru/internal-projects/focus/media/rest
go get gitlab.aeroidea.ru/internal-projects/focus/media/aws-s3
```

* `media/plugin` - основной плагин
* `media/postgres` - адаптер для работы с базой данных postgres
* `media/rest` - адаптер, реализующий REST API
* `media/aws-s3` - адаптер, реализующий интерфейс файлового хранилища, и работающий с s3-совместимыми хранилищами

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
	{ // AWS S3 client
		Name: "focus.awsS3.client",
		Build: func(ctn di.Container) (interface{}, error) {
			cfg := aws.Config{
				Region:      env.AwsRegion,
				Credentials: credentials.NewStaticCredentialsProvider(env.AwsAccessKeyID, env.AwsSecretAccessKey, ""),
				EndpointResolverWithOptions: aws.EndpointResolverWithOptionsFunc(func(service, region string, options ...interface{}) (aws.Endpoint, error) {
					if service == s3.ServiceID {
						return aws.Endpoint{
							PartitionID:       "aws",
							URL:               env.AwsEndpoint,
							SigningRegion:     env.AwsRegion,
							HostnameImmutable: true,
						}, nil
					}
					return aws.Endpoint{}, &aws.EndpointNotFoundError{}
				}),
			}
			client := s3.NewFromConfig(cfg)
			return client, nil
		},
	},
	{ // AWS S3 bucket
		Name: "focus.awsS3.bucketName",
		Build: func(ctn di.Container) (interface{}, error) {
			return env.AwsBucket, nil
		},
	},
}
```

Определить переменные, относящиеся к плагину медиа

```go
// https://gitlab.aeroidea.ru/internal-projects/focus-group/demo/-/blob/develop/internal/infrastructure/registry/services_definitions/media.go#L16
var MediaDefinitions = appendArr([]di.Def{
	{
		Name: "focus.media.baseUrl",
		Build: func(ctn di.Container) (interface{}, error) {
			return url.Parse(env.AwsEndpoint + "/" + env.AwsBucket)
		},
	},
}, plugin.Definitions, postgres.Definitions, aws_s3.Definitions, rest.Definitions)
```

Если нужно, определить переводы к ошибкам

```go
// https://gitlab.aeroidea.ru/internal-projects/focus-group/demo/-/blob/develop/internal/infrastructure/registry/services_definitions/translations/translations.go#L7
var ErrTranslations = []et.Translation{
	{
		Tag:         "folder.conflict",
		Translation: "Папка с таким названием уже существует.",
	},
	{
		Tag:         "folder.recursion",
		Translation: "Невозможно сделать дочернюю папку родительской.",
	},
	{
		Tag:         "folder.moving-same-folder",
		Translation: "Папка уже лежит в этой папке.",
	},
	{
		Tag:         "folder.renaming-same-name",
		Translation: "У папки уже такое название.",
	},
	{
		Tag:         "media.renaming-same-name",
		Translation: "У медиа уже такое название.",
	},
	{
		Tag:         "media.moving-same-folder",
		Translation: "Медиа уже лежит в этой папке.",
	},
	{
		Tag:         "media.conflict",
		Translation: "Медиа с таким названием уже существует.",
	},
	{
		Tag:         "folder.not-found",
		Translation: "Папка не найдена.",
	},
	{
		Tag:         "media.not-found",
		Translation: "Медиа не найдено.",
	},
	{
		Tag:         "media.not-exists",
		Translation: "Один из медиа файлов не найден.",
	},
	{
		Tag:         "media.file.size",
		Translation: "Размер медиа файла не может превышать 10МБ.",
	},
 }
```

Регистрация сервисов в контейнере

```javascript
// https://gitlab.aeroidea.ru/internal-projects/focus-group/demo/-/blob/develop/internal/infrastructure/registry/container.go#L65
	if err = builder.Add(services_definitions.MediaDefinitions...); err != nil {
		return nil, err
	}
```

Подключение роутов

```javascript
// https://gitlab.aeroidea.ru/internal-projects/focus-group/demo/-/blob/develop/internal/adapters/rest/router.go#L102
func (r Router) Router() *gin.Engine {
	gin.SetMode(r.ginMode)
	router := gin.New()
	// обработка ошибок page not found и method not allowed
	router.NoRoute(func(c *gin.Context) {
		c.JSON(http.StatusNotFound, gin.H{"applicationErrorCode": http.StatusText(http.StatusNotFound), "message": "page not found"})
	})
	router.NoMethod(func(c *gin.Context) {
		c.JSON(http.StatusMethodNotAllowed, gin.H{"applicationErrorCode": http.StatusText(http.StatusMethodNotAllowed), "message": "method not allowed"})
	})

	healthCheck := func(c *gin.Context) { c.JSON(http.StatusOK, gin.H{"status": "ok"}) }

	api := router.Group(r.apiSettings.Root)
	v1 := api.Group(r.apiSettings.Version)

	// internal
	...

	// cms focus
	focus := v1.Group(r.apiSettings.FocusPath)
	focus.Group("health").Group("check").GET("", healthCheck)
	...
	r.focusMediaRouter.SetRoutes(focus)
	...

	return router
}
```

## Настройка

Пока что можно лишь настраивать переменную `focus.media.baseUrl` - отвечает за относительный путь до файлового хранилища. Используется для построения полного пути до файла.

## Примеры использования

Примеры использования можно найти в сервисе демо <https://gitlab.aeroidea.ru/internal-projects/focus-group/demo>

Тут <https://gitlab.aeroidea.ru/internal-projects/focus/-/blob/develop/models/examples/promo.go#L17>

И тут <https://gitlab.aeroidea.ru/internal-projects/focus/-/blob/develop/models/examples/product.go#L15>



