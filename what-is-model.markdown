---
layout: page
title: Определение модели
permalink: /models/what-is-model
parent: Модели
nav_order: 2
---

Модель в CMS focus - это структура для управления данными при помощи UI focus. Такой структурой может быть представлена обычная сущность.

Для того, чтобы структура стала моделью, она должна реализовать интерфейс `focus.Focusable`

```go
type Focusable interface {
	TableName() string
	ModelTitle() string
}
```

Метод `TableName()` - определяет, к какой таблице в базе данных привязана эта модель (аналогично для gorm)

Метод `ModelTitle()` - определяет название модели, которое будет отображаться в UI.

Для настройки полей модели используется [тег focus]({{ "/models/focus" | relative_url }}).


