
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

   <img width="1632" height="4138" alt="Untitled (3)" src="https://github.com/user-attachments/assets/f6889b7f-9a14-408a-b2ca-fd96de1e7368" />

 
## 6. Физическая схема БД
   ### 6.1 Распределение таблиц по базам данных
   
   | База данных | Тип | Назначение | Таблицы |
   |-------------|-----|------------|---------|
   | **PostgreSQL (OLTP)** | Реляционная, ACID | Основные бизнес-данные, требующие транзакционности | `users`, `user_sessions`, `api_clients`, `api_keys`, `api_sessions`, `subscriptions`, `payment_methods`, `invoices`, `model_families`, `model_versions`, `model_deployments`, `fine_tuning_jobs`, `training_datasets`, `user_attachments`, `token_usage`, `usage_quotas`, `message_embeddings`, `search_index_jobs` |
   | **Cassandra** | NoSQL, Wide-Column | Сообщения и чаты (огромная нагрузка на запись/чтение) | `chats`, `user_messages`, `assistant_responses`, `message_contexts`, `search_snapshots` |
   | **Redis** | In-Memory Key-Value | Кеш контекста, сессии, rate limiting | `hot counters (usage:*)`, `сессии (session:*)`, `pub/sub стримов (stream:resp:*)`, `кеш контекста (ctx:msg:*)`, `prefix_cache моделей`|
   | **ClickHouse** | Колоночная OLAP | Аналитика, DWH | `dwh_session_events`, `dwh_chat_events`, `dwh_message_events`, `dwh_token_events`, `dwh_click_events` |
   | **Qdrant** | Векторная БД | Семантический поиск | `message_embeddings` |
   | **Elasticsearch** | Поисковая БД | Полнотекстовый поиск | `полнотекстовый индекс сообщений и ответов` |
   | **S3 / Object Storage** | Объектное хранилище | Медиафайлы, веса моделей | `media_S3`, `weight_S3`|

   ### 6.2 Индексы

   | Таблица | Состав индекса | Тип индекса | Пояснение |
   |---------|----------------|-------------|-----------|
   | **users** (`auth_db`) | 1) `email`<br>2) `phone`<br>3) `id` | 1) UNIQUE B-Tree<br>2) UNIQUE B-Tree<br>3) PRIMARY KEY B-Tree | Поиск по email и телефону при аутентификации, primary key для join'ов |
   | **user_sessions** (`auth_db`) | 1) `token_hash`<br>2) `user_id, expires_at`<br>3) `expires_at` | 1) UNIQUE B-Tree<br>2) Composite B-Tree<br>3) B-Tree | Lookup сессии по токену, активные сессии пользователя, batch-очистка просроченных |
   | **api_clients** (`auth_db`) | 1) `id`<br>2) `owner_user_id`<br>3) `status, last_used_at` | 1) PRIMARY KEY B-Tree<br>2) B-Tree<br>3) Composite B-Tree | Аутентификация клиента, список клиентов разработчика, выявление неактивных |
   | **api_keys** (`auth_db`) | 1) `key_hash`<br>2) `api_client_id, status`<br>3) `expires_at` | 1) UNIQUE B-Tree<br>2) Composite B-Tree<br>3) B-Tree | Аутентификация по ключу, активные ключи клиента, очистка просроченных |
   | **api_sessions** (`auth_db`) | 1) `api_key_id, started_at`<br>2) `user_id, started_at` | 1) Composite B-Tree<br>2) Composite B-Tree | Аудит сессий по ключу и по пользователю |
   | **subscriptions** (`billing_db`) | 1) `user_id, status`<br>2) `end_date`<br>3) `user_id, start_date` | 1) Composite B-Tree<br>2) B-Tree<br>3) Composite B-Tree | Активная подписка пользователя, обработка истекающих, аналитика конверсий |
   | **payment_methods** (`billing_db`) | 1) `user_id`<br>2) `user_id` WHERE `is_default=true`<br>3) `subscription_id` | 1) B-Tree<br>2) Partial B-Tree<br>3) B-Tree | Способы оплаты, основной метод, связь с подпиской |
   | **token_usage** (`billing_db`) | 1) `user_id, billing_period`<br>2) `billing_period`<br>3) `api_key_id, billing_period` | 1) Composite B-Tree<br>2) B-Tree (партиция)<br>3) Composite B-Tree | Агрегация для инвойсов, отчёты периода, биллинг по ключу. Таблица партиционирована по `billing_period` |
   | **usage_quotas** (`billing_db`) | 1) `user_id, quota_period, period_start`<br>2) `is_exceeded` WHERE `is_exceeded=true` | 1) Composite UNIQUE B-Tree<br>2) Partial B-Tree | Текущая квота пользователя, быстрая выборка превысивших лимит |
   | **user_attachments** (`files_db`) | 1) `user_message_id`<br>2) `user_id, created_at`<br>3) `file_hash_sha256`<br>4) `av_scan_status` WHERE `av_scan_status='pending'`<br>5) `expires_at` | 1) B-Tree<br>2) Composite B-Tree<br>3) B-Tree<br>4) Partial B-Tree<br>5) B-Tree | Файлы сообщения, файлы пользователя, дедупликация, AV-очередь, GDPR-удаление |
   | **model_families** (`model_db`) | 1) `family_code`<br>2) `is_public` WHERE `is_public=true` | 1) UNIQUE B-Tree<br>2) Partial B-Tree | Поиск по коду семейства, выборка публичных моделей для UI |
   | **model_versions** (`model_db`) | 1) `version_code`<br>2) `family_id, status`<br>3) `base_version_id` | 1) UNIQUE B-Tree<br>2) Composite B-Tree<br>3) B-Tree | Lookup версии, production-версии семейства, дерево fine-tune происхождения |
   | **model_deployments** (`model_db`) | 1) `model_version_id, status`<br>2) `region, status` | 1) Composite B-Tree<br>2) Composite B-Tree | Активные deployments версии, deployments в регионе для маршрутизации |
   | **fine_tuning_jobs** (`model_db`) | 1) `owner_user_id, created_at`<br>2) `status` WHERE `status IN ('queued','running')`<br>3) `base_version_id` | 1) Composite B-Tree<br>2) Partial B-Tree<br>3) B-Tree | История заданий пользователя, активные задания для scheduler, потомки базовой модели |
   | **message_embeddings** (`search_meta_db`) | 1) `source_type, source_id`<br>2) `chat_id`<br>3) `user_id, created_at` | 1) Composite UNIQUE B-Tree<br>2) B-Tree<br>3) Composite B-Tree | Lookup эмбеддинга по источнику, эмбеддинги чата для RAG, индексация per-user |
   | **search_index_jobs** (`search_meta_db`) | 1) `status, enqueued_at` WHERE `status='queued'`<br>2) `source_type, source_id` | 1) Partial B-Tree<br>2) Composite B-Tree | Очередь индексации, retry для конкретного источника |
   | **chats** (`chat_keyspace`) | Partition: `user_id`<br>Clustering: `last_activity_at DESC, chat_id` | Compound (Cassandra) | Список чатов пользователя, отсортированных по последней активности. Чтение одной партиции = один узел |
   | **user_messages** (`chat_keyspace`) | Partition: `chat_id`<br>Clustering: `created_at, id` | Compound (Cassandra) | История переписки чата, упорядоченная по времени. Write-once, без UPDATE |
   | **assistant_responses** (`chat_keyspace`) | Partition: `user_message_id`<br>Clustering: `version DESC` | Compound (Cassandra) | Ответы на сообщение, включая альтернативы (regenerate). Primary version — первая в выдаче |
   | **message_contexts** (`chat_keyspace`) | Partition: `user_message_id` | Compound (Cassandra) | Снимок контекста, отправленного в модель. 1-к-1 с user_message |
   | **dwh_message_events** (`dwh_db`) | `PARTITION BY toYYYYMMDD(event_time)`<br>`ORDER BY (event_type, event_time, user_id)` | MergeTree (ClickHouse) | Партиционирование по дням для эффективного TTL и дроп старых партов; сортировка под типовые `GROUP BY event_type` |
   | **dwh_token_events** (`dwh_db`) | `PARTITION BY toYYYYMM(event_time)`<br>`ORDER BY (user_id, event_time)` | AggregatingMergeTree | Месячное партиционирование для биллинга; pre-aggregation materialized view сверху для invoice generation |
   | **msg_embeddings** (Qdrant) | Vector index | HNSW (M=16, ef_construct=200) | Approximate nearest neighbor по семантике, recall@10 ≈ 98% при latency P99 5–15 мс на шарде |
   | **messages-YYYY-MM** (Elasticsearch) | Inverted index по `text`<br>Filter index по `user_id` (routing) | Lucene inverted + keyword | Полнотекстовый поиск с фильтром по пользователю. Routing по `user_id` направляет запрос на один шард |

   ### 6.3 Шардирование
   
   | СУБД / БД | Таблица | Ключ шардирования | Пояснение |
   |-----------|---------|-------------------|-----------|
   | **PostgreSQL — `auth_db`** | `users` | `id` | Все запросы к профилю, сессиям, API-ключам выполняются в контексте `user_id`. Шардирование по `id` локализует все связанные данные пользователя на одном шарде. |
   | **PostgreSQL — `auth_db`** | `user_sessions` | `user_id` | Сессии запрашиваются вместе с пользователем (refresh-токены, аудит). Шард совпадает с шардом users — нет cross-shard join'ов. |
   | **PostgreSQL — `auth_db`** | `api_clients` | `owner_user_id` | API-клиенты разработчика лежат рядом с самим разработчиком. При rotation ключей и листинге клиентов — один шард. |
   | **PostgreSQL — `auth_db`** | `api_keys` | `api_client_id` | Ключи живут рядом со своим клиентом. Lookup по `key_hash` использует глобальный consistent-hash routing на уровне приложения. |
   | **PostgreSQL — `auth_db`** | `api_sessions` | `api_key_id` | Аудит сессий ключа — локально |
   | **PostgreSQL — `billing_db`** | `subscriptions` | `user_id` | Подписки всегда запрашиваются вместе с пользователем. Co-location с биллинговыми данными для транзакционности при апгрейде/даунгрейде тарифа. |
   | **PostgreSQL — `billing_db`** | `payment_methods` | `user_id` | Способы оплаты привязаны к пользователю. Транзакция «списать с карты + продлить подписку» проходит в рамках одного шарда. |
   | **PostgreSQL — `billing_db`** | `invoices` | `user_id` | Инвойсы пользователя локализованы. Биллинговые отчёты строятся по шардам параллельно. |
   | **PostgreSQL — `billing_db`** | `usage_quotas` | `user_id` | Квоты пользователя на одном шарде с подпиской — атомарная проверка лимита при обработке запроса. |
   | **PostgreSQL — `billing_db`** | `token_usage` | `user_id`, доп. партиционирование по `billing_period` | Шардирование по `user_id` + локальное партиционирование по месяцу. Это даёт быстрый месячный rollup для инвойса без cross-shard scan. |
   | **PostgreSQL — `files_db`** | `user_attachments` | `user_id` | Файлы пользователя локализованы для GDPR-удаления и list-операций. |
   | **PostgreSQL — `model_db`** | `model_families`, `model_versions`, `model_deployments` | **Без шардирования** | Объём данных мал (десятки тысяч строк). Полная репликация на все PG-узлы + read-through кеш в `model_cache` (Redis). Любой Inference Scheduler читает локально без round-trip. |
   | **PostgreSQL — `model_db`** | `fine_tuning_jobs` | `owner_user_id` | Задания клиента локализованы для list-API и квот на одновременное обучение. |
   | **PostgreSQL — `search_meta_db`** | `message_embeddings`, `search_index_jobs` | `user_id` | Совпадает с шардом всех пользовательских данных. Поиск всегда per-user. |
   | **Cassandra — `chat_keyspace`** | `chats` | Partition: `user_id` | Все чаты одного пользователя — на одной партиции (физически на одном узле в рамках replication set). Список диалогов с сортировкой по `last_activity_at DESC` читается одним запросом. |
   | **Cassandra — `chat_keyspace`** | `user_messages` | Partition: `chat_id` | Все сообщения одного чата — на одной партиции. История чата выгружается одним range-scan'ом по clustering key `created_at`. Consistent hashing распределяет чаты по узлам кластера равномерно. |
   | **Cassandra — `chat_keyspace`** | `assistant_responses` | Partition: `user_message_id` | Все версии ответа на одно сообщение (regenerate) — рядом. Чтение primary version и альтернатив — один запрос. |
   | **Cassandra — `chat_keyspace`** | `message_contexts` | Partition: `user_message_id` | Контекст лежит рядом с сообщением и ответом. Для regenerate всё доступно локально. |
   | **Cassandra — `chat_keyspace`** | `search_snapshots` | Partition: `user_message_id` | Снимок web-search идёт в паре с сообщением. |
   | **Redis — `auth_cache`** | `session:{token_hash}` | Hash slot от `token_hash` | Redis Cluster делит 16384 hash slots между мастер-узлами. Routing по slot выполняется клиентской библиотекой без proxy. |
   | **Redis — `quota_counters`** | `usage:user:{id}:*`, `quota:user:{id}` | Hash slot от `user:{id}` (hash tag) | Все счётчики одного пользователя через `{user:id}` hash tag попадают в один slot — атомарные multi-key операции (`MULTI/EXEC`) работают локально. |
   | **Redis — `chat_cache`** | `ctx:msg:{user_message_id}`, `stream:resp:{response_id}` | Hash slot от ключа | Pub/sub-канал стриминга и кеш контекста — на одном slot'е через hash tag по `chat_id`, чтобы subscriber Chat Service и publisher vLLM попадали на один узел. |
   | **Redis — `model_cache`** | `model:family:*`, `model:version:*`, `model:deployments:active` | Hash slot (но фактически full replication) | Данных мало (~МБ), удобнее реплицировать на каждый узел кластера и читать локально. Invalidation через pub/sub `model_config_changed`. |
   | **Redis — `inf_cache`** | `prefix_cache:{prompt_hash}` | Hash slot от `prompt_hash` | Равномерное распределение горячих префиксов системных промптов между узлами. |
   | **ClickHouse — `dwh_db`** | все `dwh_*_events` | `cityHash64(user_id) % N_shards` | Distributed-таблица поверх локальных `MergeTree`. Запросы аналитиков обычно фильтруют по диапазону дат + опционально `user_id`. Партиционирование по дате (внутри шарда) + сортировка по `(event_type, event_time)` дают эффективный pruning. |
   | **Qdrant — `vector_index`** | collection `msg_embeddings` | `user_id` (sharding key в payload) | HNSW-индекс per-shard. Запрос с фильтром `user_id == X` маршрутизируется на один шард — без fan-out по всему кластеру. |
   | **Elasticsearch — `fulltext_index`** | индексы `messages-YYYY-MM` | `user_id` (custom routing) | Routing parameter `?routing=user_id` направляет запрос на один шард. Без этого ES выполнил бы scatter-gather по всем 200 шардам индекса — недопустимо при таком объёме. |
   | **S3** | `user_uploads_bucket`, `model_weights_bucket` | Префикс ключа (`/uploads/{user_id}/...`) | Шардирование выполняется самим S3 на уровне префиксов ключа. Хорошо распределённые префиксы по `user_id` или `file_hash` дают равномерную нагрузку без hot partitions. |

   ### 6.4 Резервирование
   
   | СУБД / БД | Схема резервирования | Формула | Пояснение |
   |-----------|---------------------|---------|-----------|
   | **PostgreSQL** (все 5 инстансов: `auth_db`, `billing_db`, `files_db`, `model_db`, `search_meta_db`) | Мастер + 1 синхронная реплика + 1 асинхронная реплика, разнесённые по 3 ДЦ | N+2 | Запись подтверждается мастером после fsync на синхронной реплике (`synchronous_commit=on`, `synchronous_standby_names`) — данные не теряются при падении мастера. Асинхронная реплика — для чтений и для disaster recovery в удалённом регионе. Failover через Patroni + etcd: автопереключение за 10–30 секунд. |
   | **Cassandra — `chat_keyspace`** | Multi-master, Replication Factor = 3, Consistency Level = QUORUM (для write и read) | RF=3, кворум 2 из 3 | Каждая партиция реплицируется на 3 узла |
   | **Redis** (все 5 инстансов: `auth_cache`, `quota_counters`, `chat_cache`, `model_cache`, `inf_cache`) | Redis Cluster: мастер + 1 реплика на каждый из 16384 hash slots, минимум 6 узлов на кластер (3 master + 3 replica) | N+1 на каждый slot | Реплики асинхронные — `WAIT` команда позволяет дождаться репликации при критичных операциях. При падении мастера sentinel-протокол кластера выбирает новую replica мастером за 5–15 секунд. Acceptable trade-off: возможна потеря последних миллисекунд write'ов — приемлемо для кешей и счётчиков (квоты восстанавливаются из PG через Quota Flusher). |
   | **ClickHouse — `dwh_db`** | ReplicatedMergeTree, RF = 2, координация через ClickHouse Keeper | N+1 на каждый шард | Каждый шард имеет 2 реплики, координация через встроенный Keeper (Raft). Запись через Distributed-таблицу: данные шардируются и реплицируются прозрачно. При падении одной реплики чтения автоматически переключаются на другую. |
   | **Qdrant — `vector_index`** | Replication Factor = 2 на каждый shard, Raft consensus | N+1 на каждый shard | Шарды реплицируются между узлами. Запись через Raft гарантирует консистентность HNSW-индексов между репликами. При падении узла поиск переключается на здоровую реплику автоматически. |
   | **Elasticsearch — `fulltext_index`** | 1 primary + 1 replica на каждый shard, кросс-зональное размещение | N+1 на каждый shard | Master-eligible nodes — 3 штуки для quorum'а при выборах. Реплика принимает на себя поисковый трафик параллельно с primary, что также даёт scale-out на чтение. При падении primary реплика автоматически промоутится. |
   | **S3** | Cross-region replication, 3 региона (US, EU, Азия) | N+2 | S3 внутри одного региона уже даёт хорошую durability за счёт хранения на 3+ зонах доступности. Cross-region replication защищает от регионального инцидента и даёт edge-доступ к весам моделей и пользовательским файлам из ближайшего ДЦ. |
   | **Kafka** | `replication.factor=3`, `min.insync.replicas=2`, разнесены по 3 ДЦ | N+1 | Запись подтверждается после ack от 2 ISR-реплик из 3. Переживает падение одного брокера без потери сообщений и без downtime. Producer'ы используют `acks=all` для сохранности данных. |
   
   ### 6.5 Библиотеки для работы с БД

   | СУБД | Библиотека |
   |------|------------|
   | PostgreSQL | pgx |
   | Cassandra | gocql | 
   | Redis | go-redis |
   | ClickHouse | clickhouse-go |
   | S3 | aws sdk |
   
## 7. Алгоритмы

   ### 7.1 Архитектура Transformer
   
   Chat gpt использует глубокое обучение на основе нейронных сетей, в частности, архитектуры Transformer. Эта архитектура позволяет модели обрабатывать текстовые данные параллельно, учитывая контекст каждого слова в предложении. Модель обучается на массивах информации, включая книги, статьи, веб-сайты и диалоги, что дает ей понимание различных стилей и жанров.

   ### 7.2 Предварительное и углубленное обучение

   Модель проходит двухступенчатый процесс обучения, состоящий из предварительной фазы и дообучения.
   На этапе предварительного обучения происходит изучение фундаментальных закономерностей языка, включая взаимосвязи между словами и фразами. Затем, на стадии дообучения, модель настраивается на выполнение конкретных задач, таких как генерация текстового контента или имитация диалога. Такой подход позволяет повысить ее эффективность в решении разнообразных задач, улучшить качество ответов и способность создавать осмысленные тексты.

   ### 7.3 Токенизация
   
   Когда пользователь отправляет запрос, GPT анализирует его с помощью токенизации — разбиения текста на отдельные слова или части слов. Затем эти токены преобразуются в векторные представления, которые являются числовыми значениями, отражающими смысл слов. Модель использует эти векторы, чтобы понять контекст запроса и сгенерировать наиболее подходящий ответ. Этот процесс занимает доли секунды. GPTunneL берет плату за использование моделей на основе использованных токенов.

   ### 7.4 Предсказание ответов
   
   В процессе генерации ответа, Chat gpt не просто копирует фрагменты из своей базы данных. Она создает новый текст, опираясь на усвоенные языковые закономерности и связи. Модель предсказывает последовательность слов, которая наиболее вероятно соответствует контексту запроса. Такой подход позволяет создавать нешаблонные ответы, а также адаптироваться к различным стилям общения.

## 8. Технологии

| Технология | Область применения | Мотивационная часть |
|------------|--------------------|---------------------|
| Go | Backend | Низкое потребление памяти, нативные легковесные горутины для обработки десятков тысяч одновременных стриминговых соединений |
| Python | Backend для инфереса | Обязательное требование экосистемы ML. Все основные фреймворки для трансформеров (PyTorch, Transformers, vLLM) написаны на Python/CUDA |
| PyTorch / CUDA | Вычислительное ядро для нейросетей | Стандарт для загрузки и выполнения моделей GPT-подобной архитектуры. Использование NVIDIA GPU для матричных вычислений |
| FastAPI (Python) | REST/SSE эндпоинты воркера инференса | Позволяет держать соединение открытым, пока модель генерирует ответ | 
| TypeScript + React | Frontend | React идеален для построения сложного SPA-интерфейса, а TypeScript обеспечивает статическую типизацию |
| Nginx | L7-балансировщик, SSL Termination | Высокая производительность при отдаче статики и проксирования соединений |
| gRPC | Взаимодействие между внутренними сервисами | Высокая производительность бинарного протокола HTTP/2, строгая контрактная система (Protobuf). Снижает накладные расходы на сериализацию по сравнению с JSON | 
| Swagger / OpenAPI | Документирование публичного REST API | Стандарт индустрии для описания HTTP эндпоинтов | 
| PostgreSQL | Основное OLTP хранилище: пользователи, подписки | Нужна реляционная БД для контроля пользователей и их подписок, ACID-транзакции |
| Apache Cassandra | Хранение истории чатов | Высокая горизонтальная масштабируемость. Выдерживает экстремальную нагрузку на запись истории сообщений |
| Redis | Кеш контекста диалога, хранение сессий | In-memory скорость. Ключ: значение | 
| S3 | Объектное хранилище для артефактов и весов моделей | Высокая надежность хранения бинарных данных без нагрузки на основные СУБД |
| ClickHouse | Аналитика и Data Warehouse | Колоночное хранение для агрегации по использованию токенов | 
| Cloudflare | Глобальная балансировка, DDoS Protection, DNS | Обеспечивает Anycast сеть и кеширование на Edge, снижая нагрузку на ядро системы |
| Docker | Контейнеры | Изоляция зависимостей и гарантия идентичности окружения на локальной машине разработчика и в продакшене | 
| Kubernetes (k8s) | Оркестрация контейнеров | Автоматическое развертывание, горизонтальное автомасштабирование (HPA) по метрикам RPS и CPU, самовосстановление упавших нод |
| Helm | Управление релизами в Kubernetes | Шаблонизация и версионирование манифестов, упрощение развертывания в разные окружения (Dev/Staging/Prod) |
| Prometheus | Сбор метрик | Стандарт для метрик | 
| Grafana | Визуализация метрик | Гибкие дашборды для анализа состояния инфраструктуры и бизнес-показателей (Tokens Per Second) |
| Jaeger | Распределенный трейсинг | Позволяет визуализировать путь запроса от DNS-балансировки до инференса модели | 
| GitHub Actions | CI/CD пайплайн | Запуск тестов, линтеров (golangci-lint), сборка Docker образов и деплой | 
| Testify (Go) | Модульное тестирование | Удобные ассерты и моки для проверки бизнес-логики сервисов |
| Pytest / Unittest | Модульное тестирование Python кода | Проверка корректности токенизации и подготовки промптов |
| Testcontainers | Интеграционное тестирование | Библиотека, которая запускает реальные БД и сервисы в Docker-контейнерах прямо во время теста |

   ## 10. Cхема проекта

   <img width="4652" height="3441" alt="СХЕМА Проекта vers3" src="https://github.com/user-attachments/assets/e14ea79b-dec4-445e-93fc-622c9b69a832" />

   ### Auth Service Domain

Домен отвечает за идентификацию и авторизацию пользователей и внешних API-клиентов. Хранит учётные записи, выпускает и валидирует сессионные токены, управляет жизненным циклом API-ключей с поддержкой rotation без downtime. Все запросы во все остальные сервисы предварительно проходят проверку авторизации через этот домен (Gateway вызывает Auth Service на каждом запросе).

| Компонент | Назначение |
|-----------|------------|
| **Auth Service (Go)** | Stateless-сервис аутентификации и авторизации. Регистрация и логин пользователей, валидация сессионных токенов, выпуск и проверка API-ключей. Делает write в PostgreSQL и read через Redis-кеш. Вызывается API Gateway на каждом запросе для проверки identity. |
| **PostgreSQL (`auth_db`)** | OLTP-хранилище. Содержит таблицы `users` (учётные записи), `user_sessions` (refresh-токены), `api_clients` (приложения-разработчики), `api_keys` (хеши ключей, никогда не сами ключи), `api_sessions` (короткоживущие сессии API-вызовов). Шардирование по `user_id` для горизонтальной масштабируемости. |
| **Redis (`auth_cache`)** | Hot-path кеш для проверки идентичности на каждом запросе. Хранит `session:{token_hash}` с TTL равным времени жизни сессии и `key_lookup:{key_hash}` для быстрой валидации API-ключа. При cache miss происходит fallback на PostgreSQL с последующим warm-up кеша. |

---

### Billing & Subscription Domain

Домен управляет монетизацией: подписками, платежами, квотами и биллингом по токенам. Критически важен принцип real-time rate-limiting — пользователь, превысивший квоту, должен получить отказ в течение миллисекунд, а не через минуты ETL-пайплайна. Поэтому домен включает hot counters в Redis и периодический flusher для их персистентности.

| Компонент | Назначение |
|-----------|------------|
| **Subscribe Service (Go)** | Управление подписками и платёжными методами. CRUD над тарифами, обработка callback'ов от платёжных шлюзов (Stripe, YooKassa, Apple Pay), активация/деактивация подписок, проверка квот перед запросом инференса. Эмитит события в `topic: token_billed` для аналитики. |
| **PostgreSQL (`billing_db`)** | OLTP-хранилище финансовых данных с строгими ACID-гарантиями. Таблицы: `subscriptions`, `payment_methods` (токенизированные референсы на платёжный шлюз, не сырые данные карт), `invoices`, `usage_quotas` (персистентное состояние квот), `token_usage` (детализация per response, партиционирование по `billing_period`). |
| **Redis (`quota_counters`)** | Hot counters для real-time rate-limiting. Хранит счётчики вида `usage:user:{id}:m:{minute}` (TTL 2 минуты), `usage:user:{id}:day:{date}` (TTL 48 часов), `quota:user:{id}` (закешированный лимит из PG). Атомарные операции `INCR`/`INCRBY` дают sub-millisecond latency при проверке. |
| **Billing Aggregator** | Асинхронный воркер, запускается раз в сутки по cron. Читает таблицу `token_usage` за прошедший день, агрегирует по `user_id` и `billing_period`, генерирует записи в `invoices`. Отдельный сервис вынесен потому, что batch-агрегация миллиардов строк не должна блокировать OLTP-нагрузку Subscribe Service. |
| **Quota Flusher** | Singleton-воркер (leader election через etcd), запускается каждые 30 секунд. Читает счётчики из Redis `quota_counters` и батчем обновляет `usage_quotas` в PostgreSQL для персистентности. Это решает проблему долговременного хранения квот без нагрузки PG на каждый INCR. |

---

### Chat Domain

Сердце системы — хранилище и обработка диалогов между пользователем и моделью. Главные требования: write-heavy нагрузка (~2,6 млрд сообщений в день), append-only паттерн (никаких UPDATE на горячем пути), низкая latency стриминга ответа от GPU к клиенту. Чат-сервис принимает запрос, строит контекст для модели, ставит inference-задачу в очередь и поддерживает SSE-соединение с клиентом до завершения генерации ответа.

| Компонент | Назначение |
|-----------|------------|
| **Chat Service (Go)** | Stateless-сервис обработки диалогов. Принимает пользовательское сообщение, строит `message_context` (история чата + RAG-чанки + system prompt), персистит сообщение и контекст в Cassandra, публикует задание в `topic: inference_requests`, открывает SSE-соединение с клиентом и подписывается на Redis pub/sub канал `stream:resp:{response_id}` для проброса токенов от vLLM обратно клиенту. |
| **Cassandra (`chat_keyspace`)** | Wide-column хранилище под write-heavy append-only нагрузку. Таблицы: `chats` (партиция по `user_id`, сортировка по `last_activity_at DESC`), `user_messages` (партиция по `chat_id`, write-once), `assistant_responses` (партиция по `user_message_id`, поддержка версий для regenerate), `message_contexts` (снимок контекста для воспроизводимости), `search_snapshots` (результаты web-поиска для запроса). Replication Factor 3, Consistency Level QUORUM. |
| **Redis (`chat_cache`)** | Два назначения. Первое — кеш контекстов сообщений (`ctx:msg:{user_message_id}` с TTL 1 час) для быстрой ре-генерации без повторного построения. Второе — pub/sub каналы (`stream:resp:{response_id}`) для стриминга токенов от vLLM-воркера к Chat Service в реальном времени. Использует sub-millisecond latency Redis для time-to-first-token < 200 мс. |
| **Chat Summarizer** | Асинхронный воркер, запускается раз в час. Для чатов длиннее 50 сообщений генерирует сжатое резюме через специальную модель, обновляет поле `summary` в таблице `chats`. Это позволяет вместо отправки всей истории в модель отправлять резюме + последние K сообщений, экономя контекстное окно и токены. |

---

### Files & Attachments Domain

Домен управляет пользовательскими вложениями: загрузка изображений, PDF, документов для подачи в модель. Имеет отличный от остальных доменов жизненный цикл — каждый файл проходит цепочку проверок и обработок: антивирусный скан, извлечение текста, GDPR-привязка к пользователю. Метаданные хранятся в PostgreSQL, бинарные блобы — в S3, что разгружает основные СУБД.

| Компонент | Назначение |
|-----------|------------|
| **File Service (Go)** | Управление загрузкой и доступом к файлам. Выдаёт presigned URL клиенту для прямой загрузки в S3 (минуя backend для эффективности), регистрирует метаданные в `user_attachments`, ставит задачу на AV-скан в Kafka, возвращает download-URL по запросу с проверкой прав доступа. |
| **PostgreSQL (`files_db`)** | OLTP-метаданные. Таблицы: `user_attachments` (имя, размер, MIME, SHA-256 для дедупликации, `av_scan_status`, `expires_at` для GDPR), `av_scan_results` (детальные результаты антивирусной проверки). Шардирование по `user_id` — все файлы пользователя на одном шарде для list-операций и удаления. |
| **S3 (`user_uploads_bucket`)** | Объектное хранилище для блобов файлов. Префиксы: `/uploads/{user_id}/{file_hash}` для оригинальных файлов, `/extracted/{file_hash}/text.txt` для извлечённого текста PDF/DOCX. Cross-region replication для георезервирования. |
| **AV Scanner** | Асинхронный воркер на основе ClamAV. Читает задания из `topic: av_scan_queue`, скачивает файл из S3, проверяет по hash-базе сигнатур и эвристическому анализу, обновляет `av_scan_status` в PG. Файлы со статусом `infected` помечаются на удаление и блокируются для скачивания. |
| **Text Extractor** | Асинхронный воркер на основе Apache Tika. Запускается после успешного AV-скана. Извлекает текст из PDF, DOCX, PPTX, поддерживает OCR для сканов через интеграцию с Tesseract. Сохраняет результат в S3 в директорию `/extracted/`, обновляет метаданные в PG. Извлечённый текст используется Chat Service для построения контекста сообщений с вложениями. |

---

### Search Domain

Домен полнотекстового и семантического поиска по истории чатов пользователя. Простой LIKE-запрос по миллиардам сообщений невозможен, поэтому используется двухуровневая стратегия: Elasticsearch для точных совпадений ключевых слов и Qdrant для семантического поиска по эмбеддингам. Метаданные эмбеддингов отделены от самих векторов для удобства operational-задач (переиндексация, миграция между моделями эмбеддингов).

| Компонент | Назначение |
|-----------|------------|
| **Search Service (Go)** | API поиска. Принимает запрос пользователя, параллельно опрашивает Elasticsearch (full-text) и Qdrant (semantic) с фильтром `user_id`, мерджит результаты с reranking, возвращает топ-N с подсветкой совпадений. Используется также внутренне Chat Service для RAG-инъекций — поиск релевантных кусков из истории для подмешивания в контекст. |
| **PostgreSQL (`search_meta_db`)** | Метаданные индексации. Таблицы: `message_embeddings` (для каждого сообщения — какой моделью посчитан вектор, в какой коллекции Qdrant лежит, какой point_id), `search_index_jobs` (очередь индексации со статусами для retry и аудита). Не enforced FK на `chat_keyspace.user_messages` — это логическая ссылка между разными хранилищами. |
| **Qdrant (`vector_index`)** | Векторная БД с HNSW-индексом. Collection `msg_embeddings` (vector_dim=1536), шардирование по `user_id` для filtered queries без fan-out. HNSW параметры M=16, ef_construct=200 дают recall@10 ≈ 98% при P99 latency 5–15 мс на шарде. Hot tier хранит последние 90 дней, более старые эмбеддинги вытесняются в S3 cold storage. |
| **Elasticsearch (`fulltext_index`)** | Inverted index Lucene. Индексы по месяцам (`messages-YYYY-MM`), routing по `user_id` для направления запроса на один shard. Поддерживает fuzzy matching, синонимы, подсветку результатов. Старые месячные индексы (>6 месяцев) переводятся в frozen tier для экономии. |
| **Search Indexer** | Асинхронный воркер. Читает задания из `topic: search_index_jobs`, вызывает Embedding Worker для расчёта вектора нового сообщения, параллельно записывает: вектор → Qdrant, текст → Elasticsearch, метаданные → PostgreSQL. Обновляет статус задания в `search_index_jobs`. Поддерживает retry при сбоях downstream-сервисов. |
| **Embedding Worker (Python + ONNX Runtime)** | Воркер расчёта эмбеддингов через модель `text-embedding-3-small`. Использует ONNX Runtime на CPU — для batch inference эмбеддингов это дешевле GPU. Обрабатывает batch'и до 1000 документов одновременно, throughput ~50 тысяч документов в секунду на узел. |

---

### Model Registry Domain

Домен управления жизненным циклом моделей. Реализует трёхуровневую иерархию: семейство (что выбирает пользователь, например GPT-4o) → версия (конкретный checkpoint, immutable после релиза) → deployment (runtime-привязка версии к пулу GPU с traffic_weight для canary). Каждый ответ ассистента ссылается на конкретный `model_version_id` для аудита и воспроизводимости. Отдельная подсистема fine-tuning'а позволяет enterprise-клиентам обучать персональные LoRA-адаптеры.

| Компонент | Назначение |
|-----------|------------|
| **Model Registry (Go)** | Управление каталогом моделей. CRUD над семействами, версиями, deployments. Принимает запросы на fine-tuning от пользователей, регистрирует задания в `fine_tuning_jobs` и публикует событие в `topic: training_jobs`. Управляет canary-выкаткой новых версий через изменение `traffic_weight` в `model_deployments`. Эмитит invalidation-события в Redis pub/sub `model_config_changed` при изменении конфигурации. |
| **PostgreSQL (`model_db`)** | OLTP-каталог. Таблицы: `model_families` (gpt-4o, gpt-4, gpt-3.5-turbo и т. д.), `model_versions` (конкретные checkpoints с ссылками на веса в S3), `model_deployments` (runtime-привязки с traffic_weight), `fine_tuning_jobs` (статусы задач обучения), `training_datasets` (версионированные датасеты для воспроизводимости). Объём данных мал — десятки тысяч строк, без шардирования, полная репликация. |
| **Redis (`model_cache`)** | Read-through кеш для горячего чтения конфигурации. Inference Scheduler читает `model:deployments:active` на каждый запрос — без кеша это создало бы 90 тыс. RPS на PostgreSQL. Ключи: `model:family:{code}`, `model:version:{id}`, `model:deployments:active` с TTL 30–60 секунд. Invalidation через pub/sub при изменении в Registry. |
| **S3 (`model_weights_bucket`)** | Объектное хранилище весов. Префиксы: `/weights/{version_id}/` для полных весов базовых моделей (~1,5 ТБ на checkpoint), `/lora/{version_id}/` для LoRA-адаптеров fine-tuned моделей (~50–500 МБ), `/datasets/{dataset_id}/` для training-датасетов. Cross-region replication для возможности горячей загрузки весов в любой GPU-кластер. |

---

### Inference Domain

Домен исполнения LLM-инференса на GPU. Принципиально вынесен из Kubernetes на bare-metal — для GPU критичен прямой доступ к железу и сети InfiniBand 100 Гбит/с между узлами для тензор-параллелизма больших моделей. Использует vLLM с PagedAttention для эффективного управления KV-cache и continuous batching для увеличения throughput.

| Компонент | Назначение |
|-----------|------------|
| **Inference Scheduler (Go)** | Stateless-сервис маршрутизации inference-запросов. Читает задания из `topic: inference_requests`, для каждого: получает список активных deployments из Redis (`model_cache`), применяет consistent hashing по `chat_id` для sticky-routing (переиспользование warm KV-cache), выбирает deployment с учётом `traffic_weight` (canary), отправляет запрос на vLLM-воркер по gRPC. Логирует выбранный `model_deployment_id` в Cassandra для audit trail. |
| **Redis (`inf_cache`)** | Internal-кеш Inference Scheduler'а. Хранит `prefix_cache:{prompt_hash}` — закешированные KV-cache для часто встречающихся системных промптов (значительно ускоряет prefill-фазу при повторном использовании одного и того же system prompt'а), а также `deployment_config` для runtime-параметров. |
| **vLLM Pool 1 (gpt-4o production)** | Основной production-пул GPU-узлов с моделью GPT-4o. Каждый узел — 8× NVIDIA A100 80GB SXM с InfiniBand. Использует PagedAttention для эффективного управления KV-cache (снижение фрагментации в 4 раза), continuous batching для увеличения throughput в 3–8 раз vs naive batching. Получает 95% production-трафика. |
| **vLLM Pool 2 (gpt-4o canary)** | Канареечный пул для новых версий GPT-4o. Получает 5% трафика для проверки регрессий перед полным выкатыванием. При обнаружении проблем `traffic_weight` устанавливается в 0, и трафик мгновенно возвращается на Pool 1. |
| **vLLM Pool 3 (gpt-3.5-turbo)** | Cheap tier для дешёвого/быстрого инференса. Используется для бесплатных пользователей, для simple-запросов, для генерации заголовков чатов и других вспомогательных задач. |

---

### Training & Fine-tuning Domain

Полностью изолированная инфраструктура обучения и дообучения моделей. Изоляция от inference критична: training — это batch-нагрузка с длительностью часы и дни, нельзя позволять ей конкурировать за GPU с inference, у которого жёсткие SLO. Использует отдельный GPU-пул, отдельный scheduler, отдельный queue. После успешного прохождения evaluation новая версия автоматически попадает в Model Registry и через canary раскатывается в production.

| Компонент | Назначение |
|-----------|------------|
| **Training Scheduler** | Управляет очередью заданий обучения. Читает из `topic: training_jobs`, выделяет свободные GPU-слоты из training-пула (отдельного от inference), запускает Fine-tune Worker, отслеживает прогресс, обновляет `fine_tuning_jobs.status` в `model_db`. Поддерживает приоритеты для enterprise-клиентов и cancellation запущенных задач. |
| **Fine-tune Workers** | GPU-воркеры обучения. Загружают базовую модель из `model_weights_bucket`, тренировочный датасет из `training_datasets`, выполняют LoRA fine-tuning (Low-Rank Adaptation — обучается не вся модель, а маленький адаптер ~50–500 МБ, что в 100 раз дешевле полного обучения). Сохраняют веса LoRA в S3, регистрируют новую версию в Model Registry. |
| **Evaluation Suite** | Сервис автоматической проверки качества свежеобученной модели. Прогоняет batch стандартных бенчмарков (MMLU, HumanEval) и внутренний eval-сет, сравнивает с baseline. При прохождении порогов качества переводит `model_version.status` из `staging` в `production` и инициирует canary-деплой через Model Registry. При регрессии — помечает версию как `failed` и не допускает в production. |

---

### Analytics / DWH Domain

Домен сырых событий и аналитики. Реализован по принципу event sourcing: ClickHouse принимает только append-only события, никаких UPDATE и счётчиков. Все агрегаты считаются аналитиками на лету через GROUP BY поверх миллиардов строк — это и есть назначение колоночных СУБД. Объём — около 2,5 ТБ событий в сутки, retention 5 лет с переходом старых партов на cold storage.

| Компонент | Назначение |
|-----------|------------|
| **ClickHouse (`dwh_db`)** | Колоночное OLAP-хранилище. Таблицы: `dwh_session_events` (логин, логаут, рефреш токена), `dwh_chat_events` (создание/архивирование/удаление чатов), `dwh_message_events` (отправка сообщений, генерация ответов, regenerate, ~3 млрд событий в сутки), `dwh_token_events` (биллинговые события для AggregatingMergeTree materialized view'ов), `dwh_click_events` (UX-аналитика: thumbs up/down, copy, share). Партиционирование по дням или месяцам, сжатие LZ4HC. |
| **DWH Ingester** | Асинхронный воркер. Читает события из Kafka-топиков `dwh_ingest` и `token_billed`, батчит по 10 тысяч событий каждые 5 секунд, делает массовую вставку в соответствующие таблицы ClickHouse. Батчинг критичен — мелкие insert'ы в ClickHouse создают много партов и замедляют merge'и. |

---

### Edge & API Gateway Domain

Внешний периметр системы. Принимает весь входящий трафик, защищает от DDoS, выполняет TLS termination, маршрутизирует запросы по доменным сервисам, применяет глобальные политики (аутентификация, rate limiting, квоты). API Gateway сам не имеет своей БД — это полностью stateless-слой, опирающийся на кеши других сервисов.

| Компонент | Назначение |
|-----------|------------|
| **Cloudflare** | Edge-слой с Anycast-маршрутизацией на ближайший дата-центр, WAF, защитой от DDoS, edge-кешированием статики (~30–40% запросов не доходят до бэкенда). Также делает геолокационную маршрутизацию для соблюдения требований data residency. |
| **L4 Balancer (LVS)** | Уровень транспортной балансировки. Алгоритм least-connection, режим Direct Server Return (ответ от backend идёт клиенту напрямую, минуя балансировщик — снижает нагрузку на L4 в 5–10 раз). Распределяет TCP-соединения между пулом Nginx-узлов. |
| **L7 Balancer (Nginx)** | Уровень прикладной балансировки. TLS termination (внутри кластера трафик идёт по HTTP без шифрования для производительности), HTTP/2, mTLS для API-tier (взаимная аутентификация для enterprise-клиентов). Маршрутизация по hostname'ам и path'ам на соответствующие upstream'ы. |
| **API Gateway (Go)** | Stateless-сервис единой точки входа. На каждом запросе: проверяет аутентификацию через Auth Service (lookup в `auth_cache`), применяет rate limiting через `quota_counters` в Redis, проверяет квоту по токенам через Subscribe Service, маршрутизирует запрос на соответствующий доменный сервис (Chat, Search, File). Не имеет собственной БД — все данные хранятся в кешах других доменов. |

---

### Event Backbone (Kafka)

Не отдельный домен, а cross-cutting инфраструктура, развязывающая синхронный, ML и асинхронный контуры обработки. Каждый топик имеет одного primary consumer'а (single responsibility) и партиционируется под характер нагрузки. Replication factor 3, min.insync.replicas 2 для durability.

| Топик | Producer'ы | Consumer | Партиционирование |
|-------|-----------|----------|-------------------|
| `inference_requests` | Chat Service | Inference Scheduler | по `chat_id` — гарантирует FIFO в рамках чата и sticky-routing на warm GPU |
| `dwh_ingest` | Chat Service, vLLM, Auth Service | DWH Ingester | по `event_type` — параллельная обработка разных типов событий |
| `token_billed` | vLLM, Subscribe Service | DWH Ingester | по `user_id` — co-location событий пользователя для агрегации |
| `search_index_jobs` | Chat Service | Search Indexer | по `user_id` — равномерное распределение нагрузки индексации |
| `av_scan_queue` | File Service | AV Scanner | по `user_id` |
| `training_jobs` | Model Registry | Training Scheduler | round-robin (заданий мало, важна параллельность) |
| `token_stream_fallback` | vLLM | Chat Service | по `chat_id` — резервный канал стриминга при недоступности Redis |

---

### Observability Domain

Cross-cutting инфраструктура наблюдаемости. Не имеет собственных бизнес-данных, но критически важна для эксплуатации highload-системы. Все сервисы экспортируют метрики (Prometheus), логи (через stdout, собираются Fluent Bit), и trace'ы (через OpenTelemetry SDK в Jaeger).

| Компонент | Назначение |
|-----------|------------|
| **Prometheus** | Time-series база метрик. Pull-модель: scraper'ы каждые 15 секунд опрашивают `/metrics` endpoint'ы всех сервисов. Хранит метрики 30 дней локально, долговременная аналитика — через remote_write в долгое S3-storage (Thanos/Mimir). |
| **Grafana** | Визуализация. Дашборды по группам сервисов, бизнес-метрики (Tokens Per Second, MAU, активные подписки), SLO-индикаторы. Alerting через Prometheus AlertManager в Slack/PagerDuty. |
| **Jaeger** | Distributed tracing. Каждый входящий запрос получает trace_id, который пробрасывается через все downstream-вызовы (Gateway → Chat → Kafka → Scheduler → vLLM). Позволяет визуализировать полный путь запроса и находить bottlenecks. Sampling 1% для production-трафика, 100% для error-traces. |

   ## 11. Список серверов

### 11.1 Модель развёртывания

Используется гибридная модель:

- **Kubernetes (bare-metal узлы)** — для stateless-сервисов: API Gateway, Auth, Chat, Subscribe, L7 Nginx. Даёт автомасштабирование, self-healing, высокую плотность упаковки.
- **Bare-metal без оркестрации** — для баз данных (PostgreSQL, Cassandra, Redis, ClickHouse) и GPU-серверов инференса. БД страдают от CPU throttling и виртуальной сети k8s; GPU-серверы требуют прямого доступа к железу без overhead контейнерной сети.
- **Хостинг:** собственное железо для GPU и Kubernetes-узлов (амортизация 5 лет), аренда bare-metal Hetzner/Hostkey для вспомогательных сервисов.

---

### 11.2 Расчёт ресурсов по сервисам

#### Stateless-сервисы (Go, ~5 000 RPS/ядро)

| Сервис | Пиковый RPS | CPU (ядра) | RAM |
|--------|-------------|------------|-----|
| API Gateway | 90 377 | 19 | 10 ГБ |
| Chat service | 90 377 | 19 | 19 ГБ |
| Auth service | 101 | 1 | 1 ГБ |
| Subscribe service | 101 | 1 | 1 ГБ |
| **Итого** | | **40** | **31 ГБ** |

#### L7 Nginx (SSL termination, ~500 CPS/ядро)

Новые TLS-соединения: 90 377 RPS / 10 (keep-alive) ≈ 9 038 CPS → 9 038 / 500 ≈ **19 ядер**.

#### Inference service (vLLM + NVIDIA A100)

- Пиковый RPS: 90 276
- Средний ответ: ~420 токенов
- Пиковая нагрузка: 90 276 × 420 ≈ **37,9 млн токенов/сек**
- Производительность 1× A100 80GB в vLLM (GPT-4-класс, continuous batching): ~12 000 токенов/сек
- Требуется GPU: 37 900 000 / 12 000 ≈ 3 158 + 20% резерв = **≈ 3 792 GPU**
- GPU на сервер: 8× A100 SXM → **≈ 474 GPU-сервера**

---

### 11.3 Конфигурации серверов

#### Собственное железо

| Название | Назначение | Конфигурация | Ядра | Кол-во | Стоимость (5 лет амортизация) |
|----------|-----------|--------------|------|--------|-------------------------------|
| kube-node | Kubernetes worker (stateless-сервисы + Nginx) | 2× Intel Xeon Gold 6338 / 16× 32 ГБ DDR4 / 2× NVMe 4 ТБ / 2× 25 Гбит/с | 64 | 6 | $14 500 / шт → $87 000 |
| gpu-node | Inference (vLLM) | 2× AMD EPYC 7763 / 16× 64 ГБ DDR4 / 8× NVIDIA A100 80GB SXM / 2× 100 Гбит/с InfiniBand | 128 | 474 | ~$400 000 / шт → ~$189,6 млн |
| pg-node | PostgreSQL (шарды + реплики) | 2× Intel Xeon Gold 5320 / 8× 32 ГБ DDR4 / 4× NVMe 4 ТБ / 2× 25 Гбит/с | 52 | 9 | $8 000 / шт → $72 000 |
| cassandra-node | Apache Cassandra | 2× Intel Xeon Silver 4314 / 8× 32 ГБ DDR4 / 4× NVMe 16 ТБ / 2× 25 Гбит/с | 32 | 6 | $12 000 / шт → $72 000 |
| redis-node | Redis (sessions + context) | 1× Intel Xeon Gold 5318Y / 16× 32 ГБ DDR4 / 2× NVMe 1 ТБ / 2× 25 Гбит/с | 24 | 6 | $7 000 / шт → $42 000 |
| ch-node | ClickHouse (DWH) | 2× Intel Xeon Gold 6338 / 16× 32 ГБ DDR4 / 8× NVMe 8 ТБ / 2× 25 Гбит/с | 64 | 2 | $18 000 / шт → $36 000 |

#### Аренда bare-metal (Hetzner/Hostkey)

| Название | Назначение | Конфигурация | Ядра | Кол-во | Аренда/мес |
|----------|-----------|--------------|------|--------|------------|
| lvs-node | L4 LVS балансировщик | Intel i9-13900 / 128 ГБ DDR5 / 2× NVMe 512 ГБ / 2× 10 Гбит/с | 24 | 4 | $120 / шт → $480 |
| monitoring | Prometheus + Grafana + Jaeger | Intel i5-13500 / 64 ГБ DDR4 / 2× NVMe 512 ГБ / 1× 10 Гбит/с | 14 | 2 | $60 / шт → $120 |
| ci-node | CI/CD (GitHub Actions runner) | Intel i5-13500 / 64 ГБ DDR4 / 2× NVMe 512 ГБ / 1× 10 Гбит/с | 14 | 2 | $60 / шт → $120 |

---

### 11.4 Kubernetes: аллокация ресурсов контейнеров

Развёртывание stateless-сервисов на `kube-node` (6 узлов × 64 ядра = 384 ядра доступно).

| Сервис | CPU request | CPU limit | RAM request | RAM limit | Реплики |
|--------|-------------|-----------|-------------|-----------|---------|
| API Gateway | 2 | 4 | 1 ГБ | 2 ГБ | 10 |
| Chat service | 2 | 4 | 2 ГБ | 4 ГБ | 10 |
| Auth service | 0.5 | 1 | 512 МБ | 1 ГБ | 2 |
| Subscribe service | 0.5 | 1 | 512 МБ | 1 ГБ | 2 |
| L7 Nginx | 2 | 4 | 256 МБ | 512 МБ | 10 |


---

### 11.5 Схема резервирования

| Группа серверов | Схема | Формула |
|----------------|-------|---------|
| kube-node | 3 ДЦ × 2 узла | N+1 на ДЦ |
| gpu-node | 3 ДЦ, равномерно | N+1 группами по 8 |
| pg-node | Мастер + 1 синхронная реплика + 1 асинхронная | N+2 |
| cassandra-node | Replication Factor = 3, QUORUM | RF=3 |
| redis-node | Redis Cluster, мастер + реплика на слот | N×2 |
| ch-node | Мастер + реплика | N+1 |
| lvs-node | Active-Passive пара | N+1 |

---

### 11.6 Итоговая сводка

| Тип | Кол-во серверов | Хостинг | Суммарные ядра | Суммарная RAM |
|-----|----------------|---------|----------------|---------------|
| kube-node | 6 | Собственное | 384 | 3 ТБ |
| gpu-node | 474 | Собственное | 60 672 | ~225 ТБ |
| pg-node | 9 | Собственное | 468 | 2,3 ТБ |
| cassandra-node | 6 | Собственное | 192 | 768 ГБ |
| redis-node | 6 | Собственное | 144 | 3 ТБ |
| ch-node | 2 | Собственное | 128 | 1 ТБ |
| lvs-node | 4 | Hetzner | 96 | 512 ГБ |
| monitoring | 2 | Hetzner | 28 | 128 ГБ |
| ci-node | 2 | Hetzner | 28 | 128 ГБ |
| Итого | 511 | | ~62 140 | ~235 ТБ |

### 11.7 Сравнение стоимости владения: покупка vs аренда
 
#### Вариант А — собственное железо (CAPEX)
 
| Сервер | Кол-во | Цена/шт | Итого |
|--------|--------|---------|-------|
| gpu-node (8× NVIDIA A100 80GB) | 474 | $400 000 | $189 600 000 |
| kube-node | 6 | $14 500 | $87 000 |
| pg-node | 9 | $8 000 | $72 000 |
| cassandra-node | 6 | $12 000 | $72 000 |
| redis-node | 6 | $7 000 | $42 000 |
| ch-node | 2 | $18 000 | $36 000 |
| lvs / ci / monitoring (Hetzner) | 8 | $60–120/мес | $720/мес |
| **CAPEX итого** | | | **$190 509 000** |
 
| Статья | Сумма |
|--------|-------|
| Амортизация / год (5 лет) | $38 101 800 |
| Амортизация / мес | $3 175 150 |
| Электричество 5 лет (~5 МВт × $0.10/кВтч) | ~$26 280 000 |
| Аренда Hetzner 5 лет | $43 200 |
| **TCO 5 лет** | **~$216 832 200** |
 
> Средняя стоимость в пересчёте на месяц: **~$3,6 млн/мес**
 
---
 
#### Вариант Б — полная аренда (чистый OPEX)
 
Аренда GPU-серверов у Hostkey (~$25 000/мес за сервер с 8× A100), CPU-серверов у Hetzner.
 
| Сервер | Кол-во | Аренда/мес (шт) | Итого/мес |
|--------|--------|----------------|-----------|
| gpu-node (8× A100, Hostkey) | 474 | $25 000 | $11 850 000 |
| kube-node | 6 | $400 | $2 400 |
| pg-node | 9 | $300 | $2 700 |
| cassandra-node | 6 | $350 | $2 100 |
| redis-node | 6 | $200 | $1 200 |
| ch-node | 2 | $400 | $800 |
| lvs / ci / monitoring | 8 | $60–120 | $720 |
| OPEX итого / мес | | | $11 859 920 |
 
| Статья | Сумма |
|--------|-------|
| OPEX / год | $142 319 040 |
| CAPEX | $0 |
| Электричество | включено в аренду |
| TCO 5 лет | ~$711 596 200 |
 
> Средняя стоимость в месяц: **$11,9 млн/мес** — в **3,3× дороже** покупки за горизонт 5 лет
 
---
 
#### Сводное сравнение
 
| Показатель | Вариант А (покупка) | Вариант Б (аренда) |
|-----------|--------------------|--------------------|
| Стартовые вложения | $190 509 000 | $0 |
| Стоимость в месяц | ~$3 600 000 | ~$11 860 000 |
| TCO 5 лет | **~$216 832 200** | ~$711 596 200 |
| Точка безубыточности | — | ~16 месяцев |
| Гибкость масштабирования | Низкая (закупка = месяцы) | Высокая (дни) |
| Риск | Устаревание железа | Зависимость от провайдера |
 
#### Вывод
 
Аренда выгоднее только на горизонте до ~16 месяцев — пока накопленные платежи не превысят стоимость покупки. На горизонте 5 лет собственное железо экономит ~$494,8 млн.
