---
layout: page
title: Модели
permalink: /plugins/models/
parent: Плагины
nav_order: 3
has_children: true
---

![broken img]({{ "/img/models-preview.png" | relative_url}})

## Возможности

Плагин реализует апи, позволяющее управлять уже существующими сущностями: создавать новые элементы, редактировать их и удалять. Плагин может работать как вместе плагином медиа, так и отдельно от него.

## Установка

* `models/plugin` - основной плагин

```bash
go get github.com/aeroideaservices/focus/models/plugin
```

* `models/postgres` - адаптер для работы с базой данных postgres

```bash
go get github.com/aeroideaservices/focus/models/postgres
```

* `models/rest` - адаптер, реализующий REST API

```bash
go get github.com/aeroideaservices/focus/models/rest
```

* `models/aws-s3` - адаптер, реализующий интерфейс файлового хранилища, и работающий с s3-совместимыми хранилищами

```bash
go get github.com/aeroideaservices/focus/models/aws-s3
```

* `models/xlsx` - адаптер, реализующий экспорт в xlsx файлы *TODO*

```bash
go get github.com/aeroideaservices/focus/models/xlsx
```

{: .note-title }
> Кастомизация
>
> Любой из адаптеров можно заменить своей реализацией. Важно, чтобы реализация была совместима с интерфейсами плагина.

Определить основные переменные фокуса

```go
// https://github.com/aeroideaservices/focus-group/demo/-/blob/develop/internal/infrastructure/registry/services_definitions/focus.go#L17
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

Определить переменные, относящиеся к плагину моделей

```go
// https://github.com/aeroideaservices/focus-group/demo/-/blob/develop/internal/infrastructure/registry/services_definitions/models.go
var ModelsDefinitions = appendArr([]di.Def{
	{
		Name: "focus.models.registry.models",
		Build: func(ctn di.Container) (interface{}, error) {
			// регистрация кастомных полей формы
			if viewsExtrasInterface, err := ctn.SafeGet("focus.models.form.views.extra"); err == nil && viewsExtrasInterface != nil {
				viewsExtras := viewsExtrasInterface.(form.ViewsExtras)
				form.RegisterViewsExtras(viewsExtras)
			}

			return []any{
				&examples.Category{},
				&examples.Product{},
				&examples.Store{},
				&examples.Promo{},
			}, nil
		},
	},
	{
		Name: "focus.models.actions.menuItems.callbacks",
		Build: func(ctn di.Container) (interface{}, error) {
			callbacksTest := ctn.Get("focus.callbacks.test").(func(plugin, entity string) callbacks.Callbacks)

			return map[string]callbacks.Callbacks{
				"categories": callbacksTest("models", "categories"),
				"products":   callbacksTest("models", "products"),
				"stores":     callbacksTest("models", "stores"),
				"promos":     callbacksTest("models", "promos"),
			}, nil
		},
	},
	{
		Name: "focus.models.fileStorage.baseEndpoint",
		Build: func(ctn di.Container) (interface{}, error) {
			return env.AwsEndpoint + "/" + env.AwsBucket, nil
		},
	},
	{
		Name: "focus.models.form.views.extra",
		Build: func(ctn di.Container) (interface{}, error) {
			mediaUploadURI := "/media/files/upload"
			if env.EditorJSMediaFolderName != "" {
				folderId, err := getFolderIdOrCreateFolder(ctn, context.Background(), env.EditorJSMediaFolderName)
				if err != nil {
					return nil, err
				}
				mediaUploadURI += "?folderId=" + folderId.String()
			}
			var productMediaFolderId *uuid.UUID
			if env.ProductMediaFolderName != "" {
				var err error
				productMediaFolderId, err = getFolderIdOrCreateFolder(ctn, context.Background(), env.ProductMediaFolderName)
				if err != nil {
					return nil, err
				}
			}
			var storeMediaFolderId *uuid.UUID
			if env.StoreMediaFolderName != "" {
				var err error
				storeMediaFolderId, err = getFolderIdOrCreateFolder(ctn, context.Background(), env.StoreMediaFolderName)
				if err != nil {
					return nil, err
				}
			}
			var promoMediaFolderId *uuid.UUID
			if env.PromoMediaFolderName != "" {
				var err error
				promoMediaFolderId, err = getFolderIdOrCreateFolder(ctn, context.Background(), env.PromoMediaFolderName)
				if err != nil {
					return nil, err
				}
			}
			return form.ViewsExtras{
				"storeEditorJs": form.ViewExtras{
					"mediaUpload": map[string]string{
						"byFile": mediaUploadURI,
					},
					"productsHints": map[string]any{
						"identifier": "id",
						"display":    []string{"externalId", "name"},
						"request": form.Request{
							URI:  "/models-v2/product/elements/list",
							Meth: http.MethodPost,
							Body: map[string]any{
								"fields": []string{"externalId", "name"},
							},
							Paginated: true,
						},
					},
				},
				"promoEditorJs": form.ViewExtras{
					"mediaUpload": map[string]string{
						"byFile": mediaUploadURI,
					},
					"productsHints": map[string]any{
						"identifier": "id",
						"display":    []string{"externalId", "name"},
						"request": form.Request{
							URI:  "/models-v2/product/elements/list",
							Meth: http.MethodPost,
							Body: map[string]any{
								"fields": []string{"externalId", "name"},
							},
							Paginated: true,
						},
					},
				},
				"categorySelect": form.ViewExtras{
					"identifier": "id",
					"display":    []string{"name"},
					"request": form.Request{
						URI:  "/models-v2/categories/elements/list",
						Meth: http.MethodPost,
						Body: map[string]any{
							"fields": []string{"name"},
						},
						Paginated: true,
					},
				},
				"selectProducts": form.ViewExtras{
					"identifier": "id",
					"display":    []string{"externalId", "name"},
					"request": form.Request{
						URI:  "/models-v2/products/elements/list",
						Meth: http.MethodPost,
						Body: map[string]any{
							"fields": []string{"externalId", "name"},
						},
						Paginated: true,
					},
				},
				"selectKladrIds": form.ViewExtras{
					"identifier": "kladrId",
					"display":    []string{"value"},
					"request": form.Request{
						URI:  "http://geo-service/suggest/address",
						Meth: http.MethodPost,
						Body: map[string]any{
							"count":     20,
							"fromBound": "region",
							"toBound":   "settlement",
						},
					},
					"getRequest": form.Request{
						URI:  "http://geo-service/findById/address-list",
						Meth: http.MethodGet,
					},
				},
				"productGallery": form.ViewExtras{
					"folderId": productMediaFolderId,
				},
				"storesMedia": form.ViewExtras{
					"folderId": storeMediaFolderId,
				},
				"promosMedia": form.ViewExtras{
					"folderId": promoMediaFolderId,
				},
				"sluggableOnName": form.ViewExtras{
					"dependsOn": map[string]any{"code": "name", "generate": "sluggable"},
				},
				"sluggableOnTitle": form.ViewExtras{
					"dependsOn": map[string]any{"code": "title", "generate": "sluggable"},
				},
			}, nil
		},
	},
}, plugin.Definitions, postgres.Definitions, aws_s3.Definitions, rest.Definitions, xlsx.Definitions)
```

Если нужно, определить переводы к ошибкам

```go
// https://github.com/aeroideaservices/focus-group/demo/-/blob/develop/internal/infrastructure/registry/services_definitions/translations/translations.go#L7
var ErrTranslations = []et.Translation{
	{
		Tag:         "model-element.field.wrong",
		Translation: "Передано неверное значение поля \"{0}\".",
	},
	{
		Tag:         "model-element.field.unique",
		Translation: "Поле \"{0}\" должно быть уникальным.",
	},
 }
```

Регистрация сервисов в контейнере

```javascript
// https://github.com/aeroideaservices/focus-group/demo/-/blob/develop/internal/infrastructure/registry/container.go#L65
	if err = builder.Add(services_definitions.ModelsDefinitions...); err != nil {
		return nil, err
	}
```

Подключение роутов

```javascript
// https://github.com/aeroideaservices/focus-group/demo/-/blob/develop/internal/adapters/rest/router.go#L102
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
	r.focusModelsRouter.SetRoutes(focus)
	...

	return router
}
```

## Настройка плагина

Для того, чтобы в плагине конфигураций можно было использовать настройки типа медиа, необходимо подключить [Плагин Медиа]({{"/plugins/media/" | relative_url}}).

* `focus.models.fileStorage.baseEndpoint` - отвечает за относительный путь до файлового хранилища. Используется для построения полного пути до файла.
* `  focus.models.actions.models.callbacks` - добавляет основные колбэки для каждой модели отдельно: `postCreate`, `postUpdare` и `postDelete`\nПример:

```go
// https://github.com/aeroideaservices/focus-group/demo/-/blob/develop/internal/infrastructure/registry/services_definitions/models.go#L41
	{
		Name: "focus.models.actions.models.callbacks",
		Build: func(ctn di.Container) (interface{}, error) {
			callbacksTest := ctn.Get("focus.callbacks.test").(func(plugin, entity string) callbacks.Callbacks)

			return map[string]callbacks.Callbacks{
				"categories": callbacksTest("models", "categories"),
				"products":   callbacksTest("models", "products"),
				"stores":     callbacksTest("models", "stores"),
				"promos":     callbacksTest("models", "promos"),
			}, nil
		},
	},
```

## Дополнительно

[Примеры](https://github.com/aeroideaservices/focus-group/demo){: .btn .btn-purple }
[Все модули](https://github.com/aeroideaservices/focus/-/tree/develop/models){: .btn .btn-blue }