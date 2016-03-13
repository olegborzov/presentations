# Избегайте преждевременной денормализации данных

> "Преждевременная оптимизация — корень всех (или большинства) проблем в программировании" **Дональд Кнут**

При проектировании [схемы данных](https://ru.wikipedia.org/wiki/%D0%A1%D1%85%D0%B5%D0%BC%D0%B0_%D0%B1%D0%B0%D0%B7%D1%8B_%D0%B4%D0%B0%D0%BD%D0%BD%D1%8B%D1%85) обязательно стоит ознакомиться с базовой теорией реляционной модели данных и её применимости для выбранной СУБД. Ваша задача при проектировании схемы базы данных – избежать логической избыточности, соблюдая известные [нормальные формы](https://ru.wikipedia.org/wiki/%D0%9D%D0%BE%D1%80%D0%BC%D0%B0%D0%BB%D1%8C%D0%BD%D0%B0%D1%8F_%D1%84%D0%BE%D1%80%D0%BC%D0%B0).

Пример преждевременной денормализации:

```python
class Category(models.Model):
    title = models.CharField()
    parent = models.ForeignKey('self')

class Article(models.Model):
    title = models.CharField()  
    category = models.ForeignKey(Category)
    parent_category = models.ForeignKey(Category)
    counter = models.IntegerField(default=0)
    date_published = models.DateTimeField()

    def save(self, *args, **kwargs):
        self.parent_category = self.category.parent
        super(Article, self).save(*args, **kwargs)

# выбираем статьи с "хлебными крошками"
for article in Article.objects.select_related('parent_category', 'category'):
    print article.parent_category.title
    print article.category.title
    print article.title        
```

В данном случае нарушена [3-я нормальная форма](https://ru.wikipedia.org/wiki/%D0%A2%D1%80%D0%B5%D1%82%D1%8C%D1%8F_%D0%BD%D0%BE%D1%80%D0%BC%D0%B0%D0%BB%D1%8C%D0%BD%D0%B0%D1%8F_%D1%84%D0%BE%D1%80%D0%BC%D0%B0):  зависимость *статья* → *родительская категория* является транзитивной. Поэтому уберем лишнее поле в модели и обновим код получения необходимых данных:

```python
class Category(models.Model):
    title = models.CharField()
    parent = models.ForeignKey('self')

class Article(models.Model):
    title = models.CharField()  
    category = models.ForeignKey(Category)

# выбираем статьи с "хлебными крошками"
for article in Article.objects.select_related('category__parent', 'category'):
    print article.category.parent.title
    print article.category.title
    print article.title
```

Нормализовав данные, мы убрали лишнюю избыточность и уменьшили количество кода.

Главное правило: JOIN'ы – это не плохо, это очевидный и быстрый способ получать связанные данные в реляционных системах хранения. Если замерами удалось доказать, что денормализация приведет к значительному улучшению производительности на текущем количестве данных, можно перейти к следующему шагу.

[Денормализация](https://ru.wikipedia.org/wiki/%D0%94%D0%B5%D0%BD%D0%BE%D1%80%D0%BC%D0%B0%D0%BB%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D1%8F) может быть полезна, если необходимо сократить время выполнения тяжелых JOIN запросов, но даже в этом случае рекомендуется использовать функционал [материализованных представлений](https://habrahabr.ru/post/188802/).
