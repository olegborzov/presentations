1. https://www.braintreepayments.com/blog/safe-operations-for-high-volume-postgresql/
2. apps.get_model - для получения текущей модели
3. Миграция всегда выполняется в транзакции (любая неперехваченная ошибка приводит к откатыванию миграции)
4. pg_activity
