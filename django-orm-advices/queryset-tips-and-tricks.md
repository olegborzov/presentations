# Правила хорошего тона при работе с моделями и QuerySet'ами

### БД всегда (почти) быстрее чем Python

Ищите компромиссы между сложностью запроса и количеством операций выполняемых на стороне СУБД. Предпочтительно выполнять агрегацию, сортировку и удаление дубликатов ([DISTINCT](http://www.postgresql.org/docs/9.0/static/sql-select.html#SQL-DISTINCT)) выполнять в SQL.

### Экономьте запросы

Старайтесь выполнить минимальное число запросов для решения конкретной задачи. Используйте [select_related](https://docs.djangoproject.com/en/1.9/ref/models/querysets/#select-related) (ForeignKey, OneToOne) и [prefetch_related](https://docs.djangoproject.com/en/1.9/ref/models/querysets/#prefetch-related) (ManyToMany, ManyToOne) при необходимости. Используйте объекты [Prefetch](https://docs.djangoproject.com/en/1.9/ref/models/querysets/#django.db.models.Prefetch), чтобы кастомизировать связанные запросы.

### Экономьте память

Старайтесь максимально точно указывать список данных, которые вы хотите получить из СУБД – если для конкретной задачи вам необходимо только два поля из всех таблицы, укажите их через [only](https://docs.djangoproject.com/en/1.9/ref/models/querysets/#django.db.models.query.QuerySet.only) (либо ненужные поля через [defer](https://docs.djangoproject.com/en/1.9/ref/models/querysets/#django.db.models.query.QuerySet.defer)) – сэкономите не только память, но и время запроса.

Обязательно используйте [values_list](https://docs.djangoproject.com/en/1.9/ref/models/querysets/#values-list), если необходимо получить только список конкретных значений:

```python
# BORING
users_id = [user.id for user in User.objects.filter(...)]

# GOOD
users_id = User.objects.filter(...).values_list('id', flat=True)
```

Используйте [values](https://docs.djangoproject.com/en/1.9/ref/models/querysets/#values), чтобы получить в виде словаря только необходимые поля – экономит память и дает дополнительные удобства для передачи данных в какие-нибудь внешние источники или API. Поддерживается автоматический join, если запрашиваются поля из разных моделей, например:

```python
# стандартно, загружает все поля из двух таблиц за один запрос
for article in Article.objects.select_related('category')[:10]:
    print article.title, article.category.name

# отлично, загружаем два поля из двух таблиц за один запрос и
# не создаем сложных Python-объектов (экземпляров моделей)
for article in Article.objects.values('title', 'category__name')[:10]:
    print article['title'], article['category__name']
```

При каждой возможности используйте метод QuerySet'a [iterator](https://docs.djangoproject.com/en/1.9/ref/models/querysets/#iterator) в циклах – он позволяет не создавать экземпляры объектов для всех полученных из БД кортежей данных, а только для одного, например:

```python
# 10 экземпляров Article в памяти
for article in Article.objects.all()[:10]:
    print article.title

# 1 экземпляр Article в памяти
for article in Article.objects.iterator()[:10]:
    print article.title
```

### Не бойтесь использовать чистый SQL

ORM не должен вас ограничивать – если в нем нет поддержки необходимого функционала, воспользуйтесь методами [extra](https://docs.djangoproject.com/en/1.9/ref/models/querysets/#extra), [raw](https://docs.djangoproject.com/en/1.9/ref/models/querysets/#raw) или в крайнем случае напишите SQL запрос в текстовом виде.

 Не стоит отказываться от решений, которые ORM не поддерживает. Главное помнить, что параметры в запрос нужно передавать только через `params`!

### У вас всегда есть внешний ключ!

Не забывайте, что при получении экземпляра модели у вас всегда есть значения внешний ключей на другие модели, называется он `<ИМЯ ПОЛЯ>_id`:

```python
article = Article.object.get(...)

# дополнительный запрос
category_id = article.category.id

# правильное решение
category_id = article.category_id
```

Если вы организуете RESfull API, то дополнительных запросов можно не делать и сразу отдавать "айдишники".

Классическая ошибка, получив `id` какого-нибудь объекта в запросе, сначала достать объект полностью из БД, а потом выполнить запрос за связанным объектом:

```python
category_id = 1

# неправильно
category = Category.objects.get(id=category_id)
articles = Article.objects.filter(category=category)

# правильно
articles = Article.objects.filter(category_id=category_id)
```

### Используйте подзапросы

Важно помнить, что в качестве значений полей можно передавать QuerySet'ы и Django-ORM самостоятельно оформит их как подзапросы, например:

```python
# лишнее
categories_id = Category.objects.filter(...).values_list('id', flat=True)
articles = Article.objects.filter(category_id__in=categories_id)

# тот же результат
articles = Article.objects.filter(category_in=Category.objects.filter(...))
```

### Избегайте случайной сортировки (`ORDER BY RANDOM()`)

Иногда вам требуется либо получить случайный элемент из таблицы, либо набор элементов, отсортированный в случайном порядке. Django предлагает использовать следующий вид сортировки:

```python
random_article = Articles.objects.order_by('?').first()
```

Проанализируем запрос:

```
Limit (actual time=2.508..2.508 rows=1 loops=1)
  ->  Sort (actual time=2.508..2.508 rows=1 loops=1)
        Sort Key: (random())
        Sort Method: top-N heapsort  Memory: 25kB
        ->  Seq Scan on article (actual time=0.013..1.249 rows=5000 loops=1)
Planning time: 0.111 ms
Execution time: 2.550 ms
```

БД делает полный скан таблицы и для каждой строки генерирует случайное число, по которому затем сортирует таблицу и выдает первую строку. Представим насколько это будет медленно на больших таблицах и на больших выборках.

Есть решения лучше:

Выбор случайного* элемента за константное время: запросим минимальный и максимальный первичный ключ (работает только если он у нас числовой), выберем случайное число между ними, запросим из таблицы строку с `id` >= выбранному:

```python
agg_info = Article.objects.aggregate(max_id=Max('id'), min_id=Min('id'))
random_id = random.randint(agg_info['min_id'], agg_info['max_id'])
random_article = Article.objects.filter(id__gte=random_id).first()  # order_by('id')?
```
**Pro**: 3 IndexScan'a, а значит что время получения можно считать константным.

**Contra**: неравномерное распределение, зависит от степени разряженности айдишников.

Для получения случайной выборки можно использовать несколько вариантов, но все они имеют свои ограничения:

* `select * from table where random() < percent` и [другие варианты](http://stackoverflow.com/questions/8674718/best-way-to-select-random-rows-postgresql)
* PosgreSQL 9.5: [TABLESAMPLE SYSTEM](http://www.postgresql.org/docs/9.5/static/sql-select.html#SQL-FROM)


### Избегайте выполнения запросов к БД во время инициализации WEB-приложения

Очень часто появляется соблазн закэшировать какие-нибудь редко меняющиеся данные в момент инициализации web-приложения, в результате в глобальной области модуля могут появится записи вида:

```python
categories = list(Category.objects.all())
```

Чем это плохо:

* Если запрос по каким-то причинам вызвал ошибку и вы не используете lazy-инициализацию, то приложение останется в неработоспособном состоянии.
* Запрос в БД происходит вне HTTP запроса, что усложняет профилирование.
* Нет возможности централизованно сбросить кэш для всех воркеров.

Одно из решений – использовать Django LocMemCache с ограниченным временем хранения.

### Инкапсулируйте логику в "толстых" моделях или менеджерах

Вместо развесистых и сложных view,  лучше сосредоточить логику в отдельных классах либо в классах моделей или менеджеров. Для определенных случаев иногда лучше писать свои собственные классы QuerySet'ов и создавать на их основе менеджеры при помощи [from_queryset](https://docs.djangoproject.com/en/1.9/topics/db/managers/#from-queryset).  Более подробно об этой стратегии организации кода написано [здесь](http://redbeacon.github.io/2014/01/28/Fat-Models-a-Django-Code-Organization-Strategy/).
