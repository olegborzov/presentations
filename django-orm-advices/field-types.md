# Правильно выбирайте типы и параметры полей


### Используйте `DecimalField` (а не `FloatField` или `IntegerField`) для денежных типов


### Текстовые поля не должны быть `NULLABLE`


### Используйте `auto_now_add` и `auto_add` в Date[Time]Field, вместо `datetime.now`.


### Явно указывайте значение `on_delete` для `ForeignKey`

(станет обязательным в Django 2.0).
