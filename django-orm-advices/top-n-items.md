# Выборка первых N сгруппированных элементов

### Проблема

Существует очень популярная задача в web-разработке: получение нескольких отсортированных строк, сгруппированных по какому-либо полю. Например:

* Для каждой темы получить N последних опубликованных статей (сортируем по дате, группируем по теме)
* N лучших учеников для каждого класса (сортируем по среднему баллу, группируем по классу)
* N самых покупаемых товаров в каждой категории (сортируем по количеству покупок, группируем по категории)

Обычно данная проблема решается разработчиками "в лоб" – на каждую группу делаем по одному запросу.

Например получим 5 последних статей для каждой категории:

```python
latest_articles_by_category = {}, N = 5
categories = Category.objects.filter(...)

# Количество запросов к БД будет равно количеству категорий
for category in categories:
    latest_articles_by_category[category] = (
        Article.objects
        .filter(category=category)
        .order_by('-date_published')[:N]
    )
```

Для случая с небольшим количеством категорий это решение вполне может подойти – все запросы должны выполнится быстро (есть индекс на `category` и `date_published`), но что если понадобилось сгруппировать по 100 категориям? Можно ли сократить количество запросов?

### `WINDOW` функции спешат на помощь!

[Оконные функции](http://postgresql.ru.net/manual/tutorial-window.html) появились в PostgreSQL 9.1 и позволяют выполнять вычисления над строками, которые находятся перед и после текущий строки результата запроса.

Напишем сначала соответствующий SQL-запрос:

```sql
SELECT * from (
  SELECT
    a.*,
    row_number()
    OVER (PARTITION BY a.category_id ORDER BY a.date_published DESC) as row
  FROM article AS a
    JOIN category AS c ON a.category_id = c.id AND c.id IN (1, 2, 3, 4)
) as s
WHERE s.row <= N;
```
Теперь напишем сначала подзапрос с оконной функцией на ORM:

```python
from django.db.models.expressions import RawSQL

articles_subquery = (
    Article.objects
    .filter(category_id__in=(1, 2, 3, 4))
    .annotate(row=RawSQL(
        'ROW_NUMBER() OVER (PARTITION BY %s ORDER BY %s DESC)',
        [
            Article._meta.get_field('category').column,
            Article._meta.get_field('date_published').column,
        ]
    ))
)
```

Осталась одна проблема – в Django 1.9 вложенные подзапросы не поддерживаются, поэтому придется воспользоваться методом `raw`:

```
N = 5
articles = Article.objects.raw(
    'SELECT * FROM ({}) AS s WHERE s.row <= %s'.format(articles.query), [n]
)

for article in articles:
    print article.title
```

Нюансы:
* `raw` возвращает объект `RawQuerySet`, который значительно отличается от `QuerySet` (некоторые методы у него недоступны).
* нетривиально достать какие-либо поля из таблицы категорий

### `LATERAL JOIN`

В PostgreSQL 9.3 был добавлен новый тип JOIN – LATERAL, он позволяет выполнить второй подзапрос в поле `FROM` для каждого элемента из первого подзапроса, т.е. эмулирует вариант "в лоб" в SQL. Для нашей задачи он выглядит так:

```sql
SELECT sa.* FROM
  (SELECT * FROM category WHERE id IN (1, 2, 3, 4)) as c
  JOIN LATERAL
  (
    SELECT * FROM article as a
    WHERE a.category_id = c.id
    ORDER BY a.date_published DESC, a.id DESC LIMIT 5
  ) as sa ON TRUE;
```

Перевод его в ORM формат делается по аналогии с предыдущим примером.

> Сравнение производительности разных подходов можно посмотреть в этой статье:
http://charlesleifer.com/blog/querying-the-top-n-objects-per-group-with-peewee-orm/
