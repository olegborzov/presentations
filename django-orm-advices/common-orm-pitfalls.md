# Прочитайте документацию по ORM полностью, а не по диагонали

Самые странные и глупые ошибки при работе с Django-ORM случаются при отсутствии детального понимания его возможностей и принципов работы. Я рекомендую к детальному изучению как минимум два раздела: [описание моделей](https://docs.djangoproject.com/en/1.9/topics/db/models/) и API для работы с [QuerySet'ами](https://docs.djangoproject.com/en/1.9/ref/models/querysets/). 

Глупости, которых можно избежать:

### QuerySet'ы выполняют запросы в "ленивом" режиме:

```python
start_time = time.time()
try:
	books = Books.objects.all() # <-- здесь запроса еще не происходит
except:
    # отдаем данные из кэша! <-- O_O
    ...
finally:
	end_time = time.time()
	measure_request_time(start_time - end_time)
```

### Проблема N+1 запросов, или зачем нужен **select_related**
```python
class Musician(models.Model):
    name = models.CharField(max_length=250)

class Album(models.Model):
    artist = models.ForeignKey(Musician)
    name = models.CharField(max_length=100)
    release_date = models.DateField()

# выводим список всех музыкантов и их альбомов
# N+1 запрос к бд
for album in Album.objects.all():
	print(album.name, album.artist.name)

# 1 запрос к бд
for album in Album.objects.select_related('artist'):
    print(album.name, album.artist.name)
```

### **Count** вместо **exists**
```python
# Есть ли у нас пользователи с балансом больше 10?
print(bool(User.objects.filter(balance__gt=10)).count())
# или
print(User.objects.filter(balance__gt=10).exists())
```
Пример анализа запросов на таблице из ~20'000 элементов:

 `EXPLAIN ANALYSE SELECT COUNT(*) FROM "user" WHERE "user"."balance" > 10`

```
Aggregate
  ->  Index Only Scan using idx_balance on user
		 Index Cond: (balance > 10::numeric)
         Heap Fetches: 0
Planning time: 0.108 ms
Execution time: 4.009 ms
```

`EXPLAIN ANALYSE SELECT (1) from "user" WHERE "user"."balance" > 10 LIMIT 1;`

```
Limit
  ->  Seq Scan on user
        Filter: (balance > 10::numeric)
        Rows Removed by Filter: 1
Planning time: 0.094 ms
Execution time: 0.026 ms
```

Разница в 150 раз!
