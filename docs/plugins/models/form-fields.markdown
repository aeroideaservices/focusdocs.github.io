---
layout: page
title: Поля формы моделей
permalink: /plugins/models/fields
grand_parent: Плагины
parent: Модели
nav_order: 2
---
# Поля формы

Описание полей формы получается в [методе получения описания модели](https://gitlab.aeroidea.ru/internal-projects/focus/-/blob/develop/models/rest/openapi.yaml#/Models/get_models__model_code_). Для формы создания и обновления настройки могут различаться.

### Простые поля

| Название | Тип данных | Может быть множ. | Описание | Пример                                                         |
|----|----|----|----|----------------------------------------------------------------|
| `textInput` | `string` | да | Текстовое поле | ![broken img]({{ "/img/models-fields/text-input.png"           | relative_url}}) |
| `numericInput` | `int` | да | Целочисленное поле | ![broken img]({{ "/img/models-fields/num-input.png"            | relative_url}}) |
| `uintInput` | `uint` | да | Целочисленное поле с min=0 | ![broken img]({{ "/img/models-fields/uint-input.png"           | relative_url}}) |
| `floatInput` | `float` | да | Поле для ввода числа с плавающей точкой | ![broken img]({{ "/img/models-fields/float-input.png"          | relative_url}}) |
| `emailInput` | `string` | да | Текстовое поле с автоподстановкой почты | ![broken img]({{ "/img/models-fields/email-input.png"          | relative_url}}) |
| `phoneInput` | `string` | да | Текстовое поле с маской телефона | ![broken img]({{ "/img/models-fields/phone-input.png"          | relative_url}}) |
| `textarea` | `string` | нет | Многострочное текстовое поле | ![broken img]({{ "/img/models-fields/textarea-input.png"       | relative_url}}) |
| `checkbox` | `bool` | нет | Чекбокс | ![broken img]({{ "/img/models-fields/checkbox-input.png"       | relative_url}}) |
| `datePickerInput` | `string` в формате `2012-01-02T00:00:00Z` | нет | Выбор даты из календаря | ![broken img]({{ "/img/models-fields/datepicker-input.png"     | relative_url}}) |
| `dateTimePicker` | `string` в формате `2012-01-02T15:30:54Z` | нет | Выбор даты и времени из календаря | ![broken img]({{ "/img/models-fields/datetimepicker-input.png" | relative_url}}) |

### Другие поля

| Название | Тип данных | Может быть множ. | Описание |
|----|----|----|----|
| `wysiwyg` | `string` | нет | Многострочное текстовое поле с html-разметкой |
| `editorJs` | `string` | нет | Многострочное текстовое поле с улучшенной html-разметкой. Сохраняется в виде json |
| `media` | `string` в формате `uuid` | да | Поле выбора медиа из медиа-библиотеки (должен быть установлен плагин media) |
| `select` | `string` `int` `uint` `float` `bool` `object?` | да | Поле с выбором из списка. Список может быть как статичным, так и задаваться через доп запрос |


