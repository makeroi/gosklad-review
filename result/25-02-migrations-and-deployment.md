# 25.2. Проблемы с миграциями и разверткой окружений

## Результат
- **Статус:** ❌ Не пройдено
- **Severity:** S1
- **Краткое заключение:** миграции находятся в нестандартной папке, база создаётся через SQL-дампы, отсутствует удобная развертка окружений для dev/stage/test.

---

## 1. Структура миграций

**Проблема:** Миграции находятся в нестандартной папке и неполные.

**Текущее состояние:**
- Миграции в `database/_migrations/` (с подчеркиванием) — 13 файлов
- Миграции в `database/migrations/` (стандартная папка) — только 1 файл
- SQL-дамп `database/gs (2).sql` — полная структура базы (1425+ строк)

**Доказательства:**
```
database/_migrations/          # 13 миграций
  - 2014_10_12_000000_create_users_table.php
  - 2022_11_02_091424_create_accounts_table.php
  - 2022_11_07_151836_create_products_table.php
  - 2022_11_07_152235_create_orders_table.php
  - 2022_12_04_190750_create_docs_table.php
  - 2022_12_06_091617_create_leads_table.php
  - ... и другие

database/migrations/           # только 1 миграция
  - 2023_02_02_150453_create_import_configs_table.php

database/gs (2).sql           # SQL-дамп всей базы
```

**Проблемы:**
- ❌ Миграции в нестандартной папке `_migrations` не выполняются автоматически при `php artisan migrate`
- ❌ База создаётся через SQL-дамп, а не через миграции
- ❌ Невозможно воспроизвести базу на чистом окружении через `php artisan migrate`
- ❌ Нет связи между SQL-дампом и миграциями
- ❌ Неизвестно, какие миграции уже применены на продакшне
- ❌ Невозможно откатить изменения через `php artisan migrate:rollback`

**Рекомендации:**
1. Перенести все миграции из `database/_migrations/` в `database/migrations/`
2. Проверить, что все таблицы из SQL-дампа покрыты миграциями
3. Удалить SQL-дамп или использовать только для восстановления данных (не для структуры)
4. Создать скрипт для генерации миграций из существующей базы (если нужно)
5. Проверить порядок миграций (даты должны быть последовательными)

---

## 2. Отсутствие удобной развертки окружений

**Проблема:** Нет автоматизированной развертки для dev/stage/test окружений.

**Текущее состояние:**
- ❌ Нет Docker/Docker Compose
- ❌ Нет Makefile или скриптов для развертки
- ❌ Нет инструкций в README
- ❌ Нет `.env.example` с описанием переменных
- ❌ База создаётся вручную через SQL-дамп
- ❌ Нет разделения окружений (dev/stage/prod)

**Доказательства:**
- Отсутствие `Dockerfile`, `docker-compose.yml`
- Отсутствие `Makefile`
- `README.md` не содержит инструкций по развертке
- `composer.json` содержит `minimum-stability: dev` — нестабильные зависимости

**Проблемы:**
- ❌ Новые разработчики не могут быстро поднять окружение
- ❌ Невозможно воспроизвести базу на чистом сервере
- ❌ Разные версии зависимостей на разных окружениях
- ❌ Нет изоляции окружений (dev/stage/prod)
- ❌ Высокий риск ошибок при ручной настройке
- ❌ Невозможно быстро развернуть тестовое окружение

**Рекомендации:**

### 2.1. Создать Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - .:/var/www/html
    ports:
      - "8000:8000"
    depends_on:
      - mysql
      - redis
    environment:
      - APP_ENV=local
      - DB_HOST=mysql
      - REDIS_HOST=redis
    networks:
      - app-network

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: ${DB_DATABASE:-gs}
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD:-root}
      MYSQL_PASSWORD: ${DB_PASSWORD:-root}
      MYSQL_USER: ${DB_USERNAME:-gs}
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - app-network

volumes:
  mysql-data:
  redis-data:

networks:
  app-network:
    driver: bridge
```

### 2.2. Создать Dockerfile

```dockerfile
# Dockerfile
FROM php:8.1-fpm

# Установка зависимостей
RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    zip \
    unzip \
    && docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd

# Установка Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Рабочая директория
WORKDIR /var/www/html

# Копирование файлов
COPY . .

# Установка зависимостей
RUN composer install --no-dev --optimize-autoloader

# Права доступа
RUN chown -R www-data:www-data /var/www/html
RUN chmod -R 755 /var/www/html

EXPOSE 8000

CMD php artisan serve --host=0.0.0.0 --port=8000
```

### 2.3. Создать Makefile

```makefile
.PHONY: setup install migrate seed test clean up down

# Первоначальная настройка
setup:
	cp .env.example .env
	composer install
	php artisan key:generate
	make migrate
	make seed

# Установка зависимостей
install:
	composer install
	npm install

# Применение миграций
migrate:
	php artisan migrate

# Откат миграций
migrate-rollback:
	php artisan migrate:rollback

# Заполнение базы тестовыми данными
seed:
	php artisan db:seed

# Запуск тестов
test:
	php artisan test

# Очистка кэшей
clean:
	php artisan cache:clear
	php artisan config:clear
	php artisan route:clear
	php artisan view:clear

# Запуск через Docker
up:
	docker-compose up -d
	make migrate
	make seed

# Остановка Docker
down:
	docker-compose down

# Пересборка Docker
rebuild:
	docker-compose down
	docker-compose build --no-cache
	docker-compose up -d
	make migrate
	make seed
```

### 2.4. Обновить README.md

Добавить разделы:
- **Требования:** PHP 8.1+, MySQL 8.0+, Redis, Composer, Node.js
- **Быстрый старт:** инструкции по установке
- **Развертка через Docker:** команды для Docker Compose
- **Развертка вручную:** пошаговые инструкции
- **Переменные окружения:** описание `.env` файла
- **Миграции:** как применять и откатывать
- **Тестирование:** как запускать тесты

### 2.5. Создать `.env.example`

```env
APP_NAME=GoSklad
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://localhost

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=gs
DB_USERNAME=root
DB_PASSWORD=

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

# amoCRM/Kommo
AMOCRM_CLIENT_ID=
AMOCRM_CLIENT_SECRET=
AMOCRM_REDIRECT_URI=

# Bitrix24
BITRIX24_CLIENT_ID=
BITRIX24_CLIENT_SECRET=

# Telegram
TELEGRAM_BOT_TOKEN=

# Другие настройки
...
```

### 2.6. Настроить автоматическую миграцию

- В CI/CD автоматически применять миграции при деплое
- В Docker Compose запускать миграции при старте (через entrypoint script)
- Создать healthcheck для проверки состояния базы

---

## 3. Проблемы с качеством миграций

**Проблемы:**
- ⚠️ Не все миграции обратимы (нет метода `down()`)
- ⚠️ Нет индексов в некоторых миграциях
- ⚠️ Нет foreign keys для целостности данных
- ⚠️ Изменения данных выполняются через консольные команды, а не через миграции
- ⚠️ Нет seeders для тестовых данных

**Доказательства:**
- `app/Console/Commands/LogsClear.php` — очистка таблиц
- `app/Console/Commands/Fix*` — исправления данных
- Миграции в `_migrations` могут не иметь `down()`

**Рекомендации:**
1. Проверить все миграции на обратимость (добавить `down()` где нужно)
2. Добавить недостающие индексы для производительности
3. Добавить foreign keys для целостности данных
4. Перенести изменения данных в миграции или seeders
5. Создать seeders для тестовых данных (dev окружение)

---

## 4. Разделение окружений

**Проблема:** Нет четкого разделения между dev/stage/prod окружениями.

**Текущее состояние:**
- ❌ Один `.env` файл для всех окружений
- ❌ Нет конфигурации для разных окружений
- ❌ Нет отдельных баз данных для dev/stage

**Рекомендации:**
1. Создать `.env.dev`, `.env.stage`, `.env.prod` (или использовать переменные окружения)
2. Настроить разные базы данных для каждого окружения
3. Использовать Docker Compose с разными конфигурациями
4. Настроить CI/CD для автоматического деплоя на stage/prod

---

## Риски

1. **Развертка:**
   - Невозможно быстро поднять новое окружение
   - Высокий риск ошибок при ручной настройке
   - Невозможно воспроизвести базу на чистом сервере

2. **Разработка:**
   - Новые разработчики тратят много времени на настройку
   - Разные версии зависимостей на разных окружениях
   - Невозможно быстро развернуть тестовое окружение

3. **Деплой:**
   - Невозможно автоматизировать деплой
   - Высокий риск ошибок при ручном деплое
   - Невозможно откатить изменения

---

## Рекомендации (приоритетные)

### Немедленно (S0/S1):
1. ✅ Перенести все миграции из `_migrations` в `migrations`
2. ✅ Создать `.env.example` с описанием переменных
3. ✅ Обновить README с инструкциями по развертке

### В течение недели (S1):
4. ✅ Создать Docker Compose для развертки
5. ✅ Создать Dockerfile
6. ✅ Создать Makefile со скриптами
7. ✅ Проверить все миграции на обратимость

### В течение месяца (S2):
8. ✅ Добавить недостающие индексы в миграции
9. ✅ Добавить foreign keys для целостности данных
10. ✅ Создать seeders для тестовых данных
11. ✅ Настроить автоматическую миграцию в CI/CD
12. ✅ Настроить разделение окружений (dev/stage/prod)

---


## Полезные ссылки

- **Laravel Migrations:** https://laravel.com/docs/migrations
- **Docker Compose:** https://docs.docker.com/compose/
- **Laravel Sail:** https://laravel.com/docs/sail (альтернатива Docker Compose)


