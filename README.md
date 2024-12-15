# Gmail

## 1. Тема и целевая аудитория

### Тема
Почтовый сервис

### Целевая аудитория
Количество активных пользователей в месяц 21 млн
1) США - 13%
2) Бразилия - 12.55%
3) Россия - 10.77%
4) Мексика - 4.4%

### MVP функционал
- отправка и получение писем
- просмотр/поиск писем
- добавление файлов к письму
- сортировка писем
- создание папок
- блокировка нежелательных писем/спама

### Продуктовые решения
- размер почтового ящика 5 ГБ
- удаление писем старше 1 года
- ограничение на отправку файлов выше 25 МБ
- предпросмотр файлов перед скачиванием
- возможность подключения сторонних почтовых сервисов, чтобы все письма приходили в одно место

### Источники
- https://reklama.rambler-co.ru/sites/rambler-mail/
- https://www.mailbutler.io/blog/email/email-statistics-trends/

## 2. Расчет нагрузки

### Продуктовые метрики
- MAU 21 млн
- DAU 14 млн

### Хранилище пользователя
Данные пользователя (личные данные, пароль, двойная аутентификация и т.д.) - до 5 КБ
Ящик писем - 5 ГБ (максимальное кол-во писем - 400)
адресная книга - 50 КБ

### Среднее количество запросов в день для пользователя
- Отправка письма
- Получение страницы писем
- Просмотр содержимого письма
- Загрузка файла
- Скачивание файла
- Поиск по письмам
- Блокировка писем
- Отметить письмо прочитанным

### Технические метрики (RPS)
|Тип запроса|Средний RPS|Максимальный RPS|
|-|-|-|
|Отправка письма|5672|10210|
|Получение страницы писем|3240|5832|
|Просмотр содержимого письма|7000|12600|
|Загрузка файла|2836|5105|
|Скачивание файла|1750|3150|
|Поиск по письмам|1620|2916|
|Блокировка писем|487|877|
|Отметить письмо прочитанным|7000|12600|

Максимальный RPS = Средний RPS * 1.8

|Тип запроса|Средняя нагрузка, Гбит/с|
|-|-|
|Отправка письма|141|
|Получение страницы писем|21.8|
|Просмотр содержимого письма|18.7|
|Загрузка файла|133|
|Скачивание файла|68|
|Поиск по письмам|10.9|
|Блокировка писем|0.004|
|Отметить письмо прочитанным|0.06|

Итого: средняя нагрузка ~ 325.3 Гбит/с; максимальная нагрузка ~ 586 Гбит/с

## 3. Глобальная балансировка нагрузки

### Функциональное разбиение по доменам (при наличии, нужно для следующего пункта)
- mail.rambler.com
- help.rambler.com

### Расположения ДЦ (влияние на продуктовые метрики)
ДЦ расположены так, чтобы покрывать максимальную территорию в странах с наибольшим количеством активных пользователей
- США: Нью-Йорк, Мехико, Лос-Анджелес
- Россия: Москва, Новосибирск, Владивосток
- Стамбул
- Бразилия

### DNS балансировка
Для балансировки будет использоваться метод, позволяющий находить сервер с наименьшим временем отклика

### Схема Anycast балансировки
Будет использоваться Anycast. ДЦ в одной стране будут делить один IP адрес. Протокол BGP позволит отправлять данные в ближайший ДЦ 

## 4. Локальная балансировка нагрузки

### L3 отказоустойчивость
Для отказоустойчивости маршрутизаторов будем использовать протокол VRRP

### L7 балансировка
Для L7 балансировки будет использоваться Nginx. Алгоритм балансировки будет least connections. Также Nginx будет регулярно проводить health check и, в случае отказа какого то сервиса, перенаправлять трафик

### SSL termination
Проверка SSL сертификатов будет выполнять Nginx, чтобы снизить нагрузку на сервер

## 5. Логическая схема БД

![image](https://github.com/user-attachments/assets/ef47b5ac-1c4b-4700-a6ba-436d47fce0bb)

#### Таблица пользователей (user):
|поле|описание|размер|
|-|-|-|
|id|первичный ключ|8Б|
|name|имя пользователя|100Б|
|address|название почтового ящика|40Б|
|age|возраст пользователя|4Б|
|avatar|ссылка на аватар в S3|150Б|
|registered_at|дата регистрации|8Б|
|add_data|дополнительная информация о пользователе||
|last_online|последняя дата входа|8Б|

Примерное кол-во записей: 150 млн

Объем: 45 ГБ

Чтение - 3.3 МБ/с

#### Таблица писем (mail):
|поле|описание|размер|
|-|-|-|
|id|первичный ключ|8Б|
|from|отправитель|8Б|
|to|получатель|n*8Б|
|title|название письма|100Б|
|headers|служебные заголовки письма|4КБ|
|content|ссылка на текст письма|n*8Б|
|is_checked|прочитано ли письмо|1Б|
|sent_at|когда отправлено|8Б|
|to_another_domain|находится ли получатель в другом почтовом домене|1Б|

Примерное кол-во записей: 2.6 трлн

Объем: 10.4 ПБ

Чтение - 1.45 ГБ/с

Запись - 59 МБ/с

#### Таблица писем (content):
|поле|описание|размер|
|-|-|-|
|text|текст письма|10КБ|
|attachment|ссылки на приложенные файлы|n*120Б|

Примерное кол-во записей: 2.6 трлн

Объем: 23.6 ПБ

Чтение: 1.1 МБ/с

Запись: 1.8 ГБ/с

#### Таблица вложений (attachment):
|поле|описание|размер|
|-|-|-|
|id|первичный ключ|8Б|
|content|отправитель|4МБ|

Примерное кол-во записей: 770 млрд

Объем: 2.8 ПБ

Чтение: 8.8 ГБ/с

Запись: 25 ГБ/с

#### Таблица черновых писем (draft_mail):
|поле|описание|размер|
|-|-|-|
|id|первичный ключ|8Б|
|from|отправитель|8Б|
|to|получатель|n*8Б|
|title|название письма|100Б|
|text|текст письма|10КБ|
|attachment|приложенные файлы|n*150Б|
|created_at|когда было создано|8Б|

Примерное кол-во записей: 150 млн

Объем: 1.5 ТБ

Чтение - 15 МБ/с

Запись - 6 МБ/с

#### Таблица контактов (contact_book):
|поле|описание|размер|
|-|-|-|
|id|первичный ключ|8Б|
|user|владелец адресной книжки|8Б|
|contact_list|список ящиков|50КБ|

Примерное кол-во записей: 150 млн

Объем: 23 ТБ

#### Таблица папок (folder):
|поле|описание|размер|
|-|-|-|
|id|первичный ключ|8Б|
|owner|владелец папки|8Б|
|name|название папки|50Б|

Примерное кол-во записей: 750 млн

Объем: 47 ГБ

Чтение: 0.5 ГБ/с

## 6. Физическая схема БД

![image](https://github.com/user-attachments/assets/5f719dab-dfe3-41f8-9b50-e6003826da06)

|Таблица|Запросы|
|-|-|
|user|- select ... from user where id = ... <br> - update table user set avatar = ... where id = ...|
|contact_book|- select ... from contact_book where user = ...|
|mail|- select ... from mail where from = '...' order by sent_at desc limit 30 offset 0 <br> - select ... from mail where from = '...' and title = '...' order by sent_at desc limit 30 offset 0 <br> - select ... from mail where id = ... <br> - update table mail set is_checked = ... where id = ... <br> - select ... from mail where from = '...' and id in (select ... from folder_mail where folder = ...) order by sent_at desc limit 30 offset 0|
|draft_mail|- select ... from draft_mail where from = '...' limit 10 offset 0 <br> - update table draft_mail set ... where id = ...|
|folder|- select ... from folder where owner = ... order by name asc|

Для каждого пользователя будет использоваться своя БД sqlite. Будер организована папка пользоватея с файлом БД и файлом конфигурации. Это сделано для ускорения чтения/записи данных профиля пользователя. В БД пользователя лежат входящие и исходящие письма

Так же, будет сделан демон, который организовывает запросы ко всем данным. Для этого он будет знать, какие пользователи имеются в конкретном шарде.

Из за того, что в sqlite нет многопоточной записи, запись будет происходить пачками. В течении какого то времени письма будут скапливаться, а затем записываться за одну транзакцию.

### Индексы
- user.id, user.address
- contact_book.user
- inbox_mail.id, inbox_mail.title, inbox_mail.sent_at, inbox_mail.folders
- outbox_mail.id, outbox_mail.title, outbox_mail.sent_at
- attachment.id
- folder.owner, folder.name
- to_user_mail.user, to_user_mail.mail

### Выбор СУБД
- для "user", "folder", "inbox_mail", "outbox_mail", "draft_mail", "contact_book" - sqlite3
- для "statistics" - Clickhouse, т.к. колонычные БД лучше всего подходят для аналитики больших данных
- для вложений будем использовать готовые объектные хранилища (S3)
- для кэширования и таблицы "session" - Redis

### Шардирование
  Шардироваться данные будут по папкам пользователей, т.к. один пользователь содержит всю свою информацию: настройки, БД со входящими, контактами и т.д. Примерное количество шардов - 1 000

### Клиентские библиотеки
- для sqlite3: https://pkg.go.dev/github.com/mattn/go-sqlite3
- для Clickhouse: https://pkg.go.dev/github.com/ClickHouse/clickhouse-go
- для S3: https://pkg.go.dev/github.com/minio/minio-go/v7
- для Redis: https://pkg.go.dev/github.com/redis/go-redis/v9

## 7. Алгоритмы

### Поиск писем
В БД пользователя будут вноситься входящие и исходящие письма. Для этого используется специально созданный демон, в котором лежат данные о пользователях текущего шарда, а также данные о других шардах. Если необходимого пользователя нет в текущих БД, то будет сделан запрос к нужному шарду

### Сохранение писем
Для сохранения полученного письма, оно будет разбито на заголовки и содержимое. От содержимого будут отделены вложения, отправлены в объектное хранилище S3, а ссылки на них положены в письмо. Само тело также будет отправлено в S3. Далее заголовки и ссылка на тело письма будут положены в БД нужного пользователя

### Получение письма
Для получения письма сначала будут взяты заголовки и ссылка на тело из БД. Затем получено само тело, В котором могут быть вложения. На месте вложений будут ссылки на S3. Заголовки и тело письма в виде единого целого уже будет отправлено пользователю

## 8. Технологии

|Технология|Область применения|Мотивационная часть|
|-|-|-|
|Go|бэкэнд|статически типизированный ЯП, хорошо поддерживающий многопоточную разработку, с простым синтаксисом, что ускорит разработку|
|gRPC|способ общения микросервисов|позволяет организовать общение между различными сервисами вне зависимости от ЯП + очень хорошо сжимает передаваемые данные благодаря protocol buffers|
|sqlite3|основная БД для хранения всей информации|использование легковесной БД для каждого пользователя|
|Clickhouse|аналитика|быстрая, расширяемая колоночная БД, хорошо подходящая для аналитики|
|Kafka|общение между микросервисами + сброс статистических данных в clickhouse|kafka сейчас является лидеров в брокерах сообщений, имеет большое community и хорошую поддержку|
|S3|хранение прикрепляемых файлов и тел писем|внешний сервис для хранения файлов пользователей позволит делегировать задачи хранения|
|Victoria metrics|хранение + обработка метрик|victoria metrics лучше сжимает данные, имеет большую пропусную способность и потребляет меньше ресурсов|
|Redis|хранение сессионных данных|высокопроизводительная in-memory база данных key-value типа|
|Nginx|балансировка нагрузки + выдача статики для клиентов|быстрая раздача статики для пользователей, возможность настроить ssl termination, L7 балансировка|
|TypeScript|клиентская сторона|необходим для разработки клиентской части приложения|
|Kotlin|разработка мобильного приложения под Android|необходим для разработки клиентской части для мобильных приложений под Android|

## 9. Обеспечение надёжности

|Технология|Способ резервирования|
|-|-|
|sqlite3|для резервного копирования каждый шард будет связан с еще одним с помощью kafka. При каждом обновлении БД новые записи будут отправляться другому шарду, что позволит сделать копии данных. Для проверки доступности будут проводиться healthcheck'и, и при отказе какого то сервера запросы будут направляться на связанный с ним. Однако, возможны потери последних данных в случае|
|clickhouse|данные в clickhouse будут отправляться через kafka, поэтому добавим еще несколько подписчиков, которые будут сохранять данные|
|minio s3|резервное копирование с помощью server-side bucket replication между другими сервисами|
|victoria metrics|встроенные средствоа позволяют сделать резервную копию для любого узла|
|Redis|RDB для создания скриншотов с определенным интервалом|

# 10. Схема проекта

![image](https://github.com/user-attachments/assets/07d3133a-65b4-4686-85f3-b45f1c5301eb)

# 11. Список серверов
|Сервис|Цилевая нагрузка, rps|CPU|RAM, GB|Сеть, Гбит/с|
|-|-|-|-|-|
|sso|1000|10|22|0.0029|
|gateway (проверка сессии)|53000|64|64|394|
|mail|53000|106|128|394|
|sqlite3|5000|64|9|
|clickhouse|50|16|32|23|
|postgres|1000|32|32|0.0029|
|redis|53000|16|16|22|
|kafka|65000|64|32|31|
|minio s3|8000|64|64|201|

|Сервис|Хостинг|Конфигурация|Количество|Стоимость за единицу, евро|
|-|-|-|-|-|
|kubenode sso|собственный|1xAMD EPYC 7313P / 16GB / 1xNVMe256GB / 10GB/s|1|2400|
|kubenode gateway|собственный|1xAMD EPYC 7313P / 32GB / 1xNVMe256GB / 25Gb/s|2|2800|
|kubenode mail|собственный|1xAMD EPYC 7313P / 32GB / 1xNVMe256GB / 25Gb/s|4|2800|
|kubenode sqlite3|собственный|1xAMD EPYC 7313P / 32GB / 1xHDD18TB / 10Gb/s|2|3800|
|kubenode clickhouse|собственный|1xAMD EPYC 7313P / 16GB / 1xNVMe4TB / 10Gb/s|3|2800|
|kubenode postgres|собственный|1xAMD EPYC 7313P / 32GB / 1xNVMe256GB / 10Gb/s|2|2500|
|kubenode redis|собственный|1xAMD EPYC 7313P / 16GB / 1xNVMe256GB / 10Gb/s|3|2400|
|kubenode kafka|собственный|1xAMD EPYC 7313P / 16GB / 1xNVMe512GB / 10Gb/s|2|2500|
|kubenode minio|собственный|1xAMD EPYC 7313P / 64GB / 24xHDD20TB / 25Gb/s|45|13000|
