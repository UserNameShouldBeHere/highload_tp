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

### L3 балансировка
В приложении будет L3 балансировка, основанная на перенаправлении трафика в зависимости от IP пользователя. Будет использоваться балансировка по алгоритму IP hashing, что позволит направлять пользователя каждый раз на один и тот же ближайший сервер. Для отказоустойчивости будет использоваться VRRP маршрутизатор

### L7 балансировка
Для L7 балансировки будет использоваться Nginx. Алгоритм балансировки будет least connections. Также Nginx будет регулярно проводить health check и, в случае отказа какого то сервиса, перенаправлять трафик

### SSL termination
Проверка SSL сертификатов будет выполнять Nginx, чтобы снизить нагрузку на сервер

## 5. Логическая схема БД

![image](https://github.com/user-attachments/assets/8b949655-1b81-4903-9f97-c455d62de3cc)

#### Таблица пользователей (user):
|поле|описание|размер|
|-|-|-|
|id|первичный ключ|8Б|
|name|имя пользователя|100Б|
|address|название почтового ящика|40Б|
|age|возраст пользователя|4Б|
|registered_at|дата регистрации|8Б|
|avatar|ссылка на аватар в S3|150Б|

Примерное кол-во записей: 150 млн

#### Таблица писем (mail):
|поле|описание|размер|
|-|-|-|
|id|первичный ключ|8Б|
|from|отправитель|8Б|
|to|получатель|8Б|
|title|название письма|100Б|
|text|текст письма|10КБ|
|attachment|ссылки на приложенные файлы в S3|n*150Б|
|is_checked|прочитано ли письмо|1Б|
|sent_at|когда отправлено|8Б|

Примерное кол-во записей: 30 трлн

#### Таблица черновых писем (draft_mail):
|поле|описание|размер|
|-|-|-|
|id|первичный ключ|8Б|
|from|отправитель|8Б|
|to|получатель|8Б|
|title|название письма|100Б|
|text|текст письма|10КБ|
|attachment|ссылки на приложенные файлы в S3||

Примерное кол-во записей: 10 млн

#### Таблица контактов (contact_book):
|поле|описание|размер|
|-|-|-|
|id|первичный ключ|8Б|
|user|владелец адресной книжки|8Б|
|contact_list|список ящиков|40Б|

Примерное кол-во записей: 150 млн

#### Таблица папок (folder):
|поле|описание|размер|
|-|-|-|
|id|первичный ключ|8Б|
|owner|владелец папки|8Б|
|name|название папки|50Б|

Примерное кол-во записей: 750 млн

#### Таблица писем в папках (folder_mail):
|поле|описание|размер|
|-|-|-|
|folder|папка|8Б|
|mail|письма, хранящиеся в папке|n*8Б|

Примерное кол-во записей: 30 трлн
