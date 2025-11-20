# 25.1. Самописные библиотеки вместо официальных

## Результат
- **Статус:** ❌ Не пройдено
- **Severity:** S1
- **Краткое заключение:** используются самописные HTTP-клиенты и хелперы вместо официальных библиотек и встроенных средств Laravel.

---

## 1. amoCRM/Kommo API

**Проблема:** Вместо официальной библиотеки используется самописный HTTP-клиент на базе `curl`.

**Текущая реализация:**
- `app/Amo/Classes/Request.php` — самописный класс с методами `get()`, `post()`, `patch()`, `json()`, `delete()`
- `app/Amo/Classes/AmoAuth.php` — самописная авторизация
- `app/Amo/Tokens/*` — самописная система управления токенами

**Официальная альтернатива:**
- **`amocrm/amocrm-api-php`** — официальная PHP-библиотека от amoCRM
  - GitHub: https://github.com/amocrm/amocrm-api-php
  - Поддерживает OAuth 2.0, автоматический refresh токенов, rate limiting

**Доказательства:**
```php
// app/Amo/Classes/Request.php:184-216
public static function get($url, $params = [], $associativeJsonDecode = true, $wrapUrl = true)
{
    self::sleepIfRequestLimit();
    $ch = curl_init($url);
    curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, 0);
    curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);
    // ... ручная работа с curl
}
```

**Проблемы:**
- ❌ Отключена проверка SSL (CURLOPT_SSL_VERIFYPEER = false)
- ❌ Дублирование кода (множество методов с одинаковой логикой)
- ❌ Сложная система rate limiting через Redis (4 разных реализации в одном файле)
- ❌ Нет обработки ошибок API (429, 503, timeout)
- ❌ Нет автоматического refresh токенов
- ❌ Нет поддержки batch-запросов
- ❌ Нет типизации и валидации ответов

**Рекомендации:**
1. Установить `composer require amocrm/amocrm-api-php`
2. Заменить все вызовы `App\Amo\Classes\Request::*` на использование официальной библиотеки
3. Мигрировать токены в формат, поддерживаемый библиотекой
4. Включить проверку SSL

---

## 2. Bitrix24 API

**Проблема:** Используется самописная обёртка над `curl` вместо официальной библиотеки.

**Текущая реализация:**
- `app/Classes/Bitrix24.php` — самописный класс с hard-coded токенами
- `app/Classes/Bitrix24Local.php` — ещё одна самописная реализация
- `app/Bitrix/CRest.php` — похоже на официальную библиотеку, но модифицированная

**Официальная альтернатива:**
- **`andrey-tech/bitrix24-api-php`** — обёртка для Bitrix24 REST API
  - GitHub: https://github.com/andrey-tech/bitrix24-api-php
  - Поддерживает OAuth, автоматический refresh токенов, batch-запросы

**Доказательства:**
```php
// app/Classes/Bitrix24.php:96-130
private static function request($action, $params=[])
{
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);
    curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, false);
    // ... hard-coded токены в коде
    $params = array_merge($params, [
        'access_token' => 'ea105d680079c9970079c9920000000a000007f7f21a3d74a4491a2e5e01cc81a9cdaf',
        // ...
    ]);
}
```

**Проблемы:**
- ❌ Hard-coded токены в коде (строка 108-112)
- ❌ Отключена проверка SSL
- ❌ Две разные реализации (`Bitrix24` и `Bitrix24Local`)
- ❌ Нет обработки ошибок и refresh токенов
- ❌ `CRest.php` модифицирован, но не обновляется

**Рекомендации:**
1. Установить `composer require andrey-tech/bitrix24-api-php`
2. Заменить все вызовы `Bitrix24::*` и `Bitrix24Local::*` на использование библиотеки
3. Вынести токены в `.env` или БД
4. Унифицировать использование (удалить дубликаты)

---

## 3. Telegram API

**Проблема:** Используется самописный класс `TG` вместо Laravel Notification channel.

**Текущая реализация:**
- `app/Classes/TG.php` — самописный класс с `curl` и hard-coded токеном
- Используется в: `app/Models/Product.php`, `app/Services/AuthService.php`, `app/Http/Middleware/WidgetMiddleware.php`

**Официальная альтернатива:**
- **Laravel Notification Channel для Telegram**
  - Пакет: `laravel-notification-channels/telegram`
  - GitHub: https://github.com/laravel-notification-channels/telegram
  - Интегрируется с Laravel Notification system

**Доказательства:**
```php
// app/Classes/TG.php
class TG
{
    private const TOKEN = '6546757677:AAEZTdeJqreMc-lDV0myR79Iw7k-FXiZjGE'; // hard-coded!
    
    public static function send($chatId, $message)
    {
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, 'https://api.telegram.org/bot' . self::TOKEN . '/sendMessage');
        // ... ручная работа с curl
    }
}
```

**Проблемы:**
- ❌ Hard-coded токен в коде
- ❌ Использует `curl` напрямую вместо Laravel HTTP
- ❌ Нет интеграции с Laravel Notification system
- ❌ Нет обработки ошибок и retry-логики

**Рекомендации:**
1. Установить `composer require laravel-notification-channels/telegram`
2. Создать Notification классы для разных типов сообщений
3. Заменить все вызовы `TG::send()` на `$user->notify(new TelegramNotification($message))`
4. Вынести токен в `.env` (`TELEGRAM_BOT_TOKEN`)

---

## 4. Другие самописные HTTP-клиенты

**Найдены следующие самописные реализации:**

1. **`app/Helpers/helpers.php:740-763`** — функция `requestGet()`
   - Отключена проверка SSL
   - Используется вместо `Http::get()` или Guzzle

2. **`app/Services/DocumentService.php`** — метод `post()`
   - Отключена проверка SSL
   - Дублирует функциональность Guzzle

3. **`app/Classes/LoyaltyRequest.php`** — самописный класс
   - Hard-coded токен в коде (строка 7)
   - Отключена проверка SSL

4. **`app/Classes/Waba.php`** — самописный класс для 360Dialog API
   - Использует `curl` напрямую
   - Нет обработки ошибок

**Проблемы:**
- ❌ Guzzle уже установлен в `composer.json`, но не используется
- ❌ Laravel HTTP Facade (`Http::get()`, `Http::post()`) не используется
- ❌ Дублирование кода и логики
- ❌ Нет единообразия в обработке ошибок
- ❌ Отключена проверка SSL везде

**Рекомендации:**
1. Заменить все самописные HTTP-клиенты на Laravel HTTP Facade или Guzzle
2. Создать единый сервис `HttpClientService` с настройками по умолчанию
3. Включить проверку SSL везде
4. Добавить retry-логику и обработку ошибок

---

## 5. Самописные хелперы вместо Laravel-функций

**Проблема:** В `app/Helpers/helpers.php` определено 70+ самописных функций, многие из которых дублируют встроенные возможности Laravel.

### 5.1. Отладка и вывод

**Самописные функции:**
- `pre($a)` — выводит `print_r()` и вызывает `die`
- `vd($a)` — выводит `var_dump()` и вызывает `die`
- `pr($a)` — выводит `print_r()` без `die`
- `pret($a)`, `preta($a)`, `vdt($a)` — варианты с условиями

**Laravel альтернативы:**
- `dd($value)` — dump and die (встроено в Laravel)
- `dump($value)` — dump без die (встроено в Laravel)
- `ddd($value)` — dump, die, debug (Laravel Debugbar)

**Проблемы:**
- ❌ `die` в продакшне может прервать выполнение
- ❌ Нет форматирования и подсветки синтаксиса
- ❌ Нет интеграции с Laravel Debugbar

**Рекомендации:**
1. Заменить все `pre()`, `vd()`, `pr()` на `dd()` или `dump()`
2. Использовать Laravel Debugbar для отладки
3. Удалить функции из `helpers.php`

---

### 5.2. Логирование

**Самописные функции:**
- `lg($fileName, $message)` — пишет в файл через `file_put_contents`
- `mkLog($message)` — пишет в `storage_path('mk_log.txt')`
- `tst($message)` — пишет JSON в `storage_path('tst.json')`
- `ql($is, $line)` — условное логирование в файл

**Laravel альтернативы:**
- `Log::info($message, $context)` — стандартное логирование Laravel
- `Log::channel('custom')->info($message)` — кастомные каналы
- `Log::stack(['single', 'daily'])->info($message)` — множественные каналы

**Проблемы:**
- ❌ Прямая запись в файлы без ротации
- ❌ Нет структурированного формата
- ❌ Нет централизованного сбора логов
- ❌ Нет уровней логирования (debug, info, warning, error)

**Рекомендации:**
1. Заменить все `lg()`, `mkLog()`, `tst()` на `Log::info()`, `Log::debug()`, etc.
2. Настроить каналы в `config/logging.php`
3. Использовать структурированное логирование с контекстом

---

### 5.3. HTTP-запросы

**Самописные функции:**
- `requestGet($url, $bearerToken)` — самописный GET через `curl`
- `app/Services/DocumentService::post()` — самописный POST

**Laravel альтернативы:**
- `Http::get($url, $headers)` — Laravel HTTP Facade
- `Http::post($url, $data, $headers)` — Laravel HTTP Facade
- `Http::withToken($token)->get($url)` — с токеном
- `Http::withHeaders(['Authorization' => 'Bearer ' . $token])->get($url)`

**Проблемы:**
- ❌ Отключена проверка SSL
- ❌ Нет обработки ошибок
- ❌ Нет retry-логики

**Рекомендации:**
1. Заменить на `Http::get()`, `Http::post()`, etc.
2. Использовать `Http::retry(3, 100)` для повторных попыток
3. Включить проверку SSL (по умолчанию включена)

---

### 5.4. Работа с базой данных

**Самописные функции:**
- `withTransaction($callback)` — обёртка над транзакцией
- `sfja($code, $subCode)` — SQL для извлечения JSON-полей
- `sfj($code, $subCode)` — SQL для извлечения JSON-значений

**Laravel альтернативы:**
- `DB::transaction(function() { ... })` — встроенные транзакции
- `DB::table()->whereJsonContains('fields->code', $value)` — JSON-запросы
- `$model->where('fields->code', $value)` — JSON-запросы в Eloquent

**Проблемы:**
- ❌ `withTransaction()` возвращает `false` при успехе, что нелогично
- ❌ SQL-инъекции в `sfja()` и `sfj()` (если не экранировать)
- ❌ Нет использования Eloquent для JSON-полей

**Рекомендации:**
1. Заменить `withTransaction()` на `DB::transaction()`
2. Использовать Eloquent JSON-запросы вместо сырого SQL
3. Использовать `whereJsonContains()`, `whereJsonLength()`, etc.

---

### 5.5. Работа с файлами

**Самописные функции:**
- `fgcJA($filePath)` — `json_decode(file_get_contents($filePath), true)`
- `fpcJA($filePath, $data)` — `file_put_contents($filePath, json_encode($data))`
- `clearFileName($fileName)` — очистка имени файла

**Laravel альтернативы:**
- `Storage::get($path)` — чтение файла
- `Storage::put($path, $content)` — запись файла
- `Storage::json($path)` — чтение JSON
- `Storage::putJson($path, $data)` — запись JSON

**Проблемы:**
- ❌ Прямая работа с файловой системой вместо Storage
- ❌ Нет поддержки разных дисков (local, s3)
- ❌ Нет проверки прав доступа

**Рекомендации:**
1. Заменить на `Storage::get()`, `Storage::put()`, `Storage::json()`
2. Использовать `Storage::disk('s3')` для облачного хранилища
3. Использовать `Storage::exists()`, `Storage::delete()`, etc.

---

### 5.6. Генерация токенов и строк

**Самописные функции:**
- `gen_token()` — генерация токена через `openssl_random_pseudo_bytes()`
- `getGoodPass($length)` — генерация пароля с проверкой на плохие слова

**Laravel альтернативы:**
- `Str::random($length)` — генерация случайной строки
- `Str::uuid()` — генерация UUID
- `Hash::make($password)` — хеширование паролей

**Проблемы:**
- ❌ `gen_token()` дублирует `Str::random()`
- ❌ `getGoodPass()` использует файл `bad-words.txt` (не найден)

**Рекомендации:**
1. Заменить `gen_token()` на `Str::random()`
2. Использовать `Str::random()` для паролей или `Hash::make()` для хеширования

---

### 5.7. Работа с датами

**Самописные функции:**
- `daysLeft($date)`, `hoursLeft($date)` — вычисление оставшегося времени
- `diffDateSeconds()`, `diffDateDays()` — разница между датами
- `date_micro()` — форматирование даты с микросекундами

**Laravel/Carbon альтернативы:**
- `Carbon::parse($date)->diffForHumans()` — человекочитаемый формат
- `Carbon::parse($date)->diffInDays($otherDate)` — разница в днях
- `Carbon::parse($date)->diffInSeconds($otherDate)` — разница в секундах
- `Carbon::now()->format('Y-m-d H:i:s.u')` — форматирование с микросекундами

**Проблемы:**
- ❌ Дублирование функциональности Carbon
- ❌ Менее гибкие методы

**Рекомендации:**
1. Использовать методы Carbon напрямую
2. Использовать `diffForHumans()` для человекочитаемого формата

---

### 5.8. Валидация и форматирование

**Самописные функции:**
- `isValidEmail($email)` — проверка email через `filter_var()`
- `phoneFormat()`, `phoneFormat7()`, `phoneFormat10()`, `phoneWabaFormat()` — форматирование телефонов
- `isEmail($string)` — пустая функция

**Laravel альтернативы:**
- `'email' => 'required|email'` — валидация в FormRequest
- `Validator::make($data, ['email' => 'email'])` — валидация
- Можно использовать пакеты для форматирования телефонов

**Проблемы:**
- ❌ `isEmail()` пустая функция
- ❌ Дублирование валидации Laravel

**Рекомендации:**
1. Использовать встроенную валидацию Laravel
2. Использовать FormRequest для валидации
3. Оставить функции форматирования телефонов (если специфичные для проекта)

---

### 5.9. JSON-ответы

**Самописные функции:**
- `json_response($success, $error)` — формирование JSON-ответа
- `apiResponse($data, $message, $success)` — API-ответ
- `apiResponseError($message)` — ошибка API

**Laravel альтернативы:**
- `response()->json($data, $status)` — стандартный JSON-ответ
- `response()->json(['success' => true, 'data' => $data])` — структурированный ответ
- Можно создать базовый Response класс для единообразия

**Проблемы:**
- ❌ Дублирование `response()->json()`
- ❌ Нет единого формата ответов

**Рекомендации:**
1. Использовать `response()->json()` напрямую
2. Создать базовый API Resource класс для единообразия
3. Использовать Laravel Resources для форматирования ответов

---

## Риски

1. **Безопасность:**
   - Отключенная проверка SSL → уязвимость к MITM-атакам
   - Hard-coded токены → утечка секретов
   - SQL-инъекции в самописных функциях

2. **Поддержка:**
   - Самописные библиотеки требуют поддержки и обновлений
   - Официальные библиотеки обновляются автоматически
   - Нет документации для самописных функций

3. **Производительность:**
   - Самописные библиотеки могут быть неоптимальными
   - Нет кэширования и пулинга соединений
   - Дублирование кода увеличивает размер приложения

4. **Тестируемость:**
   - Самописные функции сложно тестировать
   - Нет моков для самописных HTTP-клиентов
   - Laravel-функции легко мокаются

---

## Рекомендации (приоритетные)

### Немедленно (S0/S1):
1. ✅ Вынести все токены из кода в `.env`
2. ✅ Включить проверку SSL во всех HTTP-клиентах
3. ✅ Заменить `pre()`, `vd()`, `pr()` на `dd()` или `dump()`

### В течение недели (S1):
4. ✅ Установить `amocrm/amocrm-api-php` и заменить самописный клиент
5. ✅ Установить `andrey-tech/bitrix24-api-php` и заменить самописный клиент
6. ✅ Установить `laravel-notification-channels/telegram` и заменить `TG::send()`
7. ✅ Заменить все `curl` на Laravel HTTP Facade или Guzzle
8. ✅ Заменить `lg()`, `mkLog()` на `Log::info()`, `Log::debug()`

### В течение месяца (S2):
9. ✅ Заменить `withTransaction()` на `DB::transaction()`
10. ✅ Заменить `fgcJA()`, `fpcJA()` на `Storage::json()`, `Storage::putJson()`
11. ✅ Заменить `gen_token()` на `Str::random()`
12. ✅ Заменить `json_response()` на `response()->json()`
13. ✅ Рефакторинг остальных хелперов

---


## Ссылки на официальные библиотеки

- **amoCRM:** https://github.com/amocrm/amocrm-api-php
- **Bitrix24:** https://github.com/andrey-tech/bitrix24-api-php
- **Telegram (Laravel):** https://github.com/laravel-notification-channels/telegram
- **Laravel HTTP:** https://laravel.com/docs/http-client
- **Guzzle:** https://docs.guzzlephp.org/en/stable/ (уже установлен)

