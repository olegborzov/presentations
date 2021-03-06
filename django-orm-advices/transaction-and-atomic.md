# Своевременно используйте транзакции и атомарные операции

### Транзакции

Главное правило работы с [транзакциями](https://ru.wikipedia.org/wiki/%D0%A2%D1%80%D0%B0%D0%BD%D0%B7%D0%B0%D0%BA%D1%86%D0%B8%D1%8F_%28%D0%B8%D0%BD%D1%84%D0%BE%D1%80%D0%BC%D0%B0%D1%82%D0%B8%D0%BA%D0%B0%29) в Django: если в коде вы выполняете несколько связанных операций `INSERT`, `UPDATE` или `DELETE` подряд, то оборачивайте их в [отдельную транзакцию](https://docs.djangoproject.com/en/1.9/topics/db/transactions/#django.db.transaction.atomic) – это поможет избежать неконсистентности данных, если одна или несколько операции завершатся с ошибкой.

Пример:

```python
from django.db import transaction

@transaction.atomic  # <-- оборачиваем функцию в транзакцию
def create_user(login, phone):
    user = User(login=login)
    profile = Profile(user=user, phone=phone)

    user.save()
    # если пользователь с таким же номером телефона
    # уже существует и стоит ограничение на его уникальность, то
    # мы получим на следующей строчке IntegrityError
    profile.save()
```

Если бы мы не обернули эту функцию в транзакцию, то могли бы получить логическое противоречие данных, когда запись о пользователе есть, а о его профиле – нет.

Даже если вы используете чистый SQL  совместно с Django-ORM запросами, `atomic` сработает корректно.


### Атомарные изменения полей

Часто необходимо изменить значение какого-нибудь числового поля в БД с помощью математических операций с его текущим значением. Классический пример – изменение баланса пользователя.

Неправильно:

```python
user = User.objects.get()  # 1
user.balance += 100  # 2
user.save(update_fields=('balance',))
```

В данном случае мы можем столкнуться с известной проблемой "[состояния гонки](https://ru.wikipedia.org/wiki/%D0%A1%D0%BE%D1%81%D1%82%D0%BE%D1%8F%D0%BD%D0%B8%D0%B5_%D0%B3%D0%BE%D0%BD%D0%BA%D0%B8)" – на сервер почти одновременно могут прийти два запроса на изменение баланса у одного и того же пользователя. В результате строка `1` выполнится в одно и то же время и в строке `2` произойдет увеличение одного и того же значения и в БД в обоих случаях сохранится одно и то же число.

Правильное решение – использование конструкции [F()](https://docs.djangoproject.com/es/1.9/ref/models/expressions/#f-expressions):

```python
from django.db.models import F

user = User.objects.get()  # 1
user.balance = F('balance') + 100  # 2
user.save(update_fields=('balance',))

# или
User.objects.filter(...).update(balance=F('balance') + 100)
```
В данном случае Django-ORM подставит математическую операцию в сам SQL запрос и БД выполнит его в соответствии с уровнем изоляции транзакций.

Подобный подход так же часто используется в счетчиках.

:exclamation: Будьте осторожны: если в одной транзакции вы выполните несколько раз `save`, то изменения поля с `F()` выполнятся столько же раз.
