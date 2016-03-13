# Проставляйте очевидные индексы (indexes) и ограничения (сonstraints) при проектировании схемы данных

### Индексы

Индексы ([CREATE INDEX](http://www.postgresql.org/docs/current/static/sql-createindex.html)) позволяют ускорить доступ к данным.

[db_index](https://docs.djangoproject.com/en/1.9/ref/models/fields/#db-index) – создает индекс поумолчанию на конкретное поле, [index_together](https://docs.djangoproject.com/en/1.9/ref/models/options/#index-together) – на список полей.

Лучшие кандидаты для индексирования – поля, по которым будет осуществляться поиск, выборка данных или сортировка, например дата публикации новости.

> Django автоматически создает индекс для полей типа ForeignKey и SlugField

### Ограничения

Ограничения ([constraints](http://www.postgresql.org/docs/current/static/ddl-constraints.html)) позволяют обеспечить целостность данных. Чаще всего самыми очевидными являются ограничения на уникальность данных, например [unique](https://docs.djangoproject.com/en/1.9/ref/models/fields/#unique) и [unique_together](https://docs.djangoproject.com/en/1.9/ref/models/options/#unique-together).

Лучшие кандидаты на уникальность – это идентификаторы чего-либо, например электронная почта пользователя (если она используется для логина), номер телефона или поля типа SlugField. Уникальность для нескольких полей может может быть использована, например, для фамилии имени и отчества.

> Django автоматически проставляет уникальность на первичный ключ и OneToOneField, а также автоматически создаст индекс на уникальное поле.
