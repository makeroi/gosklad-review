# 23.1. Быстрые команды проверки

## Результат
- **Статус:** ✅ Подготовлено
- **Severity:** S3
- **Краткое заключение:** список команд собран и использовался при аудите; пригодится для регулярных проверок.

## Шпаргалка
```bash
php artisan about
php artisan route:list
php artisan config:cache && php artisan route:cache && php artisan view:cache
php artisan migrate:status
php artisan horizon:status
composer outdated
composer audit
php -m
SHOW INDEX FROM <table>
EXPLAIN <sql>
```

## Рекомендации
- Расширить чеклист командами CI/CD и анализом логов.
- Хранить шпаргалку рядом с runbook'ами.
