# Следуйте Django coding style

Пример сферического кода, который вы могли бы увидеть в каком-нибудь простом Django-проекте (Python 2.7):

```python
# -*- coding: utf-8 -*-
from django.db import models

class Books(models.Model):
    class Meta:
        verbose_name = u'Книга'
        verbose_name_plural = u'Книги'

    title = models.CharField(u'Название', max_length=100)
    authors = models.ManyToManyField(Author, verbose_name=u'Авторы')
```

### Проблемы

* Все модели в коде Django (User, Group, Permission и т.д.) используют в названии существительное в единственном числе, так как экземпляр модели является отражением строки в таблице и реализует шаблон проектирования [ActiveRecord](http://www.martinfowler.com/eaaCatalog/activeRecord.html). В данном случае модель названа "во множественном числе", что некорректно с точки зрения конвенций наименования моделей в Django.
* Код "засорен" литералами `u''` для обозначения строк в Unicode формате.
* Отсутствует полезный метод `__unicode__`, позволяющий выводить более понятное строковое представление экземляра модели, нежели `Book object` (str, unicode) или `<Book: Book object>` (repr).

### Вариант решения

Воспользуемся официальным [руководством Django](https://docs.djangoproject.com/en/1.9/internals/contributing/writing-code/coding-style/) для разработчиков и конструкцией `unicode_literals` в Python 2.7, чтобы переписать код:

```python
# -*- coding: utf-8 -*-
from __future__ import unicode_literals
from django.db import models

class Book(models.Model):
    title = models.CharField('Название', max_length=100)
    authors = models.ManyToManyField(Author, verbose_name='Авторы')

    class Meta:
        verbose_name = 'Книга'
        verbose_name_plural = 'Книги'

    def __unicode__(self):
        return self.title
```

Теперь при отладке экземпляр книги будет отображаться в более понятном и красивом варианте `<Book: Тихий Дон>`.

:exclamation: Стоит учесть, что переименование уже существующей модели может нести за собой достаточно сложный рефакторинг кода, а также обязательную миграцию схемы данных.

:exclamation: Импортирование `unicode_literals` может привести к различным [негативным эффектам](http://python-future.org/unicode_literals.html#drawbacks) в уже существующем коде, рекомендуется его использование только в новых файлах.

### Дополнительно

* [Философия архитектурного дизайна Django](https://docs.djangoproject.com/en/1.9/misc/design-philosophies/)
* [Python \_\_future\_\_ package](https://docs.python.org/2/library/__future__.html)
