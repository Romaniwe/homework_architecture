# Проектирование высоконагруженного аналога chat GPT
## 1. Тема и целевая аудитория
   ### 1.1 Описание сервиса
   ChatGPT — это разговорный ИИ-ассистент на основе большой языковой модели (Large Language Model, LLM), разработанный OpenAI. Сервис предназначен для массового конечного пользователя и решает широкий спектр задач: от ответов на вопросы и написания текстов до генерации изображений и анализа данных
   ### 1.2 Целевая аудитория
   * Количество пользователей в месяц(MAU): 5.9 млрд
   * Количество пользоватеелй в день(DAU): 193 млн
   * Географическое положение: Весь мир
   ### 1.3 Ключевой функционал сервиса (MVP)
   * Регистрация и аутентификация
   * Обучение модели
   * Отправка текстового запроса
   * Потоковая выдача ответа (Streaming)
   * Сохранение истории диалогов
   * Контекст диалога
   * Разграничение бесплатного и платного доступа
   ### Источники
   * https://www.demandsage.com/chatgpt-statistics/
## 2. Расчет нагрузки
   ### 2.1 Продуктовые метрики
   | Метрика | Значение | Как считал? |
   | ------- | -------- | ----------- |      
   | MAU     | 5.9 млрд | 1 пункт | 
   | DAU     | 193 млн  | 1 пункт |
   | Среднее число сообщений в день на активного пользователя | 13.5 сообщений | 2600000000 / 193000000=13,47 |
   | Среднее число символов для ответа пользователю | 1686 символов | [Источник](https://www.demandsage.com/chatgpt-statistics/) |
   | Средний размер пользовательского сообщения | 78 символов | [Источник](https://www.demandsage.com/chatgpt-statistics/) |
   | Среднее число регистраций за месяц | 87.5 млн | WAU за февраль 2025 = 400 млн; WAU за Апрель-июнь уже 700-800 млн => Рост около 350 млн за 4 месяца => 350/4=87.5 млн в месяц новых юзеров [Источник](https://thunderbit.com/blog/chatgpt-stats-usage-growth-trends?utm_source=chatgpt.com)| 
   | Срок хранения чатов | 30 дней | - |
 
   ### 2.2 Технические метрики
   #### Расчёт объёма хранения
   Расчет производится с учетом того, что чаты хранятся 30 дней
   | Тип данных | Расчёт | Итоговый обём |
   | ---------- | ------ | ------------- |  
   | Инференс | 2600000000 x (78 + 1686) x 2 байта x 30 дней | 250,28 ТБ |
   | Сервис регистрации и аутентификации | 87500000 х 1,5 КБ | 125.169 ГБ |
   #### Расчет RPS
   | Тип запроса | Общее кол-во запросов в сутки | Средний RPS (Общ. кол-во / 86400) | Пиковый RPS (Средний RPS x 3) |
   | ----------- | ----------------------------- | --------------------------------- | ----------------------------- |
   | Инференс    | 2600000000                    | 30092                             | 90276                         |
   | Сервис регистрации и аутентификации | 87500000 / 30 = 2916666,7 | 33,7 | 101,3 |
   | Итого | 2602916666 | 30125,7 | 90377,3 |
   #### Сетевой трафик
   | Тип трафика | Объем данных за сутки (Тбайт/сут) | Средний трафик (Гбит/с) | Пиковый трафик ( Средний трафик x 3 ) |
   | ----------- | --------------------------------- | ----------------------- | ------------------------------------- |
   | Инференс | 2600000000 x (78 + 1686) x 2 байта = 8,34 | 0,772 | 2,316 |
   | Сервис регистрации и аутентификации | 87500000 х 1,5 КБ / 30 = 0,00271 | 0,00025 | 0,00075 |

## 3. Глобальная балансировка нагрузки
   ### 3.1 Функциональное разбиение по доменам
   При исследовании разбиения Chatgpt на домены были выделены следующие
   | Доменное имя | Назначение | 
   | ------------ | ---------- |
   | ab.chatgpt.com | аб тесты |
   | chatgpt.com | основной интерфейс |
   | api.chatgpt.com | API | 
   ### 3.2 Расположение дата-центров
   Virginia, Arizona, Texas, Washington D.C, California - Здесь находятся датацентры Microsoft Asure, которые арендует openai [Источник](https://observer.com/2024/06/elon-musk-xai-big-tech-data-center-location/#:~:text=Microsoft%20and%20OpenAI:%20OpenAI%20has,Texas%2C%20Washington%20D.C.%20and%20California.)
   Сингапур - для покрытия Азии ( а людей из Азии на нашей планете очень много ) Через Сингапур проходят десятки подводных оптоволоконных магистралей, связывающих Азию с остальным миром.
   Франкфурт - для покрытия Европы
   ### 3.3 Распределение запросов по ДЦ
   * Северная Америка ( 30% )
   * Азия ( 40% )
   * Европа ( 20% )
   * Остальной мир ( 10% )
   ### 3.4 Схема DNS балансировки
   OpenAI использует Anycast DNS (преимущественно через инфраструктуру Cloudflare и Microsoft Azure DNS)
   User - > Запрос на chatgpt.com - >Cloudflare/Azure DNS -> Анализ IP (Геолокация: Германия) -> DNS Response -> Возвращает IP Frankfurt -> Edge Server -> Проверяет нагрузку на бэкенд и пробрасывает запрос на кластер Azure West находящийся в Европе

   ### 3.4 Схема Anycast
   Используется один IP. Например, у chatgpt.com есть конкретные IP-адреса (например, принадлежащие Cloudflare). Эти адреса анонсируются (объявляются) одновременно из сотен дата-центров по всей планете через протокол BGP.
## 4. Локальная балансировка нагрузки
   ### 4.1 Схема балансировки
   Двух уровневая схема балансировки
   ### L4 балансировщик:
   * LVS (linux virtual server) распределяет трафик на L7 балансировщики ( а вот уже ответ от бека пойдет напрямую пользователю, а не на балансироващик обратно ).
   * Least connection смотрит, на список подключений и выбирает менее загруженный.
   ### L7 балансировщики (Nginx):
   Занимаются SSL Termination и отправляют запросы на бэкенды по http уже.  
## 5. Логическая схема БД
   ### 5.1 Схема БД
   <img width="985" height="782" alt="image" src="https://github.com/user-attachments/assets/c3bdafb5-73e4-4ef5-892a-cfbd5fddb59f" />
## 6. Физическая схема БД
   ### 6.1 Распределение таблиц по базам данных

| База данных | Тип | Назначение | Таблицы |
|-------------|-----|------------|---------|
| **PostgreSQL (OLTP)** | Реляционная, ACID | Основные бизнес-данные, требующие транзакционности | `users`, `user_sessions`, `subscriptions`, `payment_methods`, `api_clients`, `api_keys`, `models` |
| **Cassandra** | NoSQL, Wide-Column | Сообщения и чаты (огромная нагрузка на запись/чтение) | `chats`, `messages` |
| **Redis** | In-Memory Key-Value | Кеш контекста, сессии, rate limiting | `chat_context` (кэш), `user_sessions` (кэш), `rate_limit_counters` |
| **ClickHouse** | Колоночная OLAP | Аналитика, DWH | `dwh_usage_facts`, `dwh_daily_aggregates`, `dwh_moderation_analytics`, `dwh_model_performance`, `dwh_billing_facts`, `dwh_retention_cohorts` |
| **S3 / Object Storage** | Объектное хранилище | Медиафайлы, веса моделей | `media_S3` (метаданные), `weight_S3` (метаданные) |
   ### 6.2 Индексы
   | Таблица | Индекс | Поля | Тип индекса | Назначение |
|---------|--------|------|-------------|------------|
| **users** | `users_pkey` | `id` | PRIMARY KEY B-Tree | Идентификация пользователя |
| | `idx_users_email` | `email` | UNIQUE B-Tree | Поиск по email при аутентификации |
| | `idx_users_phone` | `phone` | UNIQUE B-Tree | Поиск по телефону |
| | `idx_users_created_at` | `created_at` | B-Tree | Аналитика регистраций |
| | `idx_users_status` | `status` | B-Tree | Фильтрация по статусу |
| **user_sessions** | `user_sessions_pkey` | `id` | PRIMARY KEY B-Tree | Идентификация сессии |
| | `idx_user_sessions_token` | `token` | UNIQUE B-Tree | Поиск сессии по токену |
| | `idx_user_sessions_user_id` | `user_id` | B-Tree | Получение всех сессий пользователя |
| | `idx_user_sessions_expires_at` | `expires_at` | B-Tree | Очистка просроченных сессий |
| **api_clients** | `api_clients_pkey` | `id` | PRIMARY KEY B-Tree | Идентификация API клиента |
| | `idx_api_clients_client_id` | `client_id` | UNIQUE B-Tree | Аутентификация по client_id |
| | `idx_api_clients_owner_user_id` | `owner_user_id` | B-Tree | Получение клиентов разработчика |
| | `idx_api_clients_status` | `status` | B-Tree | Фильтрация активных клиентов |
| | `idx_api_clients_last_used_at` | `last_used_at` | B-Tree | Очистка неиспользуемых клиентов |
| **api_keys** | `api_keys_pkey` | `id` | PRIMARY KEY B-Tree | Идентификация API ключа |
| | `idx_api_keys_key_hash` | `key_hash` | UNIQUE B-Tree | Аутентификация по ключу |
| | `idx_api_keys_user_id` | `user_id` | B-Tree | Получение всех ключей пользователя |
| | `idx_api_keys_user_id_status` | `user_id, status` | Composite B-Tree | Получение активных ключей пользователя |
| | `idx_api_keys_expires_at` | `expires_at` | B-Tree | Очистка просроченных ключей |
| **subscriptions** | `subscriptions_pkey` | `id` | PRIMARY KEY B-Tree | Идентификация подписки |
| | `idx_subscriptions_user_id` | `user_id` | B-Tree | Получение подписок пользователя |
| | `idx_subscriptions_user_id_status` | `user_id, status` | Composite B-Tree | Получение активной подписки |
| | `idx_subscriptions_end_date` | `end_date` | B-Tree | Обработка истекающих подписок |
| | `idx_subscriptions_start_date` | `start_date` | B-Tree | Аналитика новых подписок |
| **payment_methods** | `payment_methods_pkey` | `id` | PRIMARY KEY B-Tree | Идентификация способа оплаты |
| | `idx_payment_methods_user_id` | `user_id` | B-Tree | Получение способов оплаты пользователя |
| | `idx_payment_methods_subscription_id` | `subscription_id` | B-Tree | Связь подписки со способом оплаты |
| | `idx_payment_methods_user_id_default` | `user_id, is_default` | Partial B-Tree | Получение основного способа оплаты |
| **models** | `models_pkey` | `id` | PRIMARY KEY B-Tree | Идентификация модели |
| | `idx_models_model_code` | `model_code` | UNIQUE B-Tree | Поиск модели по коду |
| | `idx_models_is_active` | `is_active` | B-Tree | Получение активных моделей |
| **media_S3** | `media_S3_pkey` | `url` | PRIMARY KEY B-Tree | Поиск медиа по URL |
| | `idx_media_S3_file_hash` | `file_hash` | B-Tree | Дедупликация файлов |
| | `idx_media_S3_created_at` | `created_at` | B-Tree | Очистка старых файлов |
| **weight_S3** | `weight_S3_pkey` | `id` | PRIMARY KEY B-Tree | Идентификация весов |
| | `idx_weight_S3_model_id` | `model_id` | B-Tree | Получение весов модели |
| | `idx_weight_S3_version` | `model_id, version` | Composite UNIQUE B-Tree | Уникальность версии модели |


