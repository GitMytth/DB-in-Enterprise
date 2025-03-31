# Лабораторная работа 1
### Выполнил студент группы 6133-010402D Калашников Дмитрий
Выбранная предметная область - сервис доставки еды.
Ниже представлена ER-диаграмма базы данных, созданная с помощью ERD-Tool в pgadmin.

![zxczxcСнимок asdasdasdasvxcvxczxczxcJPG](https://github.com/user-attachments/assets/5f66e896-1850-405d-905d-a8d2a9669f4e)

Есть таблцицы пользователей Users и таблица с историей заказов User_actions, пользователь может сделать заказ и отменить его. Поскольку пользователь может неоднократно совершать заказы, то связь между данными таблицами 1:N . Данные о заказе (id, время, список продуктов) хранятся в таблице Orders она связана отношениями 1:N с таблицами User_actions и Courier_actions. Данные о продуктах хранятся в таблице Products. Связь между таблицами Products и Orders M:N. Данные о курьерах хранятся в таблице Couriers, действия курьеров по отношению к заказам хранятся в таблице Courier_actions. Связь между данными таблицами 1:N. Также существует таблица для хранения данных о транспортных средствах курьеров. Одному курьеру может принадлежать только одно транспортное средство - связь 1:1.

Далее данные таблицы были создани с помощью SQL-скриптов. Ниже приведен пример скрипта, создающего таблицу User_actions.
```sql
CREATE TABLE IF NOT EXISTS public."User_actions"
(
    user_id bigint NOT NULL,
    action character varying(64) NOT NULL,
    order_id bigint NOT NULL,
    payment_type character varying(6) NOT NULL,
    PRIMARY KEY (user_id, action, order_id),
	FOREIGN KEY (order_id) REFERENCES public."Orders"(order_id),
	FOREIGN KEY (user_id) REFERENCES public."Users"(user_id)
);
```
Далее для столбцов, которые могут использоваться для поиска, были созданы индексы. Несколько примеров создания индекса приведены ниже.
```sql
CREATE INDEX IF NOT EXISTS idx_couriers_rate ON public."Couriers" (rating);

CREATE INDEX IF NOT EXISTS idx_couriers_age ON public."Couriers" (age);

CREATE INDEX IF NOT EXISTS idx_orders_date ON public."Orders" (creation_date);

CREATE INDEX IF NOT EXISTS idx_products_price ON public."Products" (price);

CREATE INDEX IF NOT EXISTS idx_users_age ON public."Users" (age);
```
Далее были разработаны типовые запросы в базу данных в количестве четырех штук. Ниже приведены описания запросов и соответствующий SQL-скрипт.

1. Получение данных для авторизации пользователей, у которых количество установленных  приложений больше среднего Подсчет количества уникальных пользователей сервиса, 
количества уникальных заказов и количество заказов на одного пользователя на одного пользователя
```sql
SELECT COUNT(DISTINCT user_id) as unique_users,
       COUNT(DISTINCT order_id) as unique_orders,
       ROUND(COUNT(DISTINCT order_id)::decimal / count(DISTINCT user_id), 2) as orders_per_user
FROM   public."User_actions"
```
2. Нахождение пользователей, которые совершили более 5 заказов
```sql
SELECT u.user_id, u.fname, u.sname, COUNT(ua.order_id) AS order_count
FROM public."Users" u
INNER JOIN public."User_actions" ua ON u.user_id = ua.user_id
GROUP BY u.user_id, u.fname, u.sname
HAVING COUNT(ua.order_id) > 5
ORDER BY order_count DESC
```
3. Определение количества отменённых заказов в таблице courier_actions, выяснение, есть ли в этой таблице заказы, которые были отменены пользователями, но при этом всё равно были доставлены.
```sql
with t1 as (SELECT order_id
FROM   public."Courier_actions"
WHERE  action = 'deliver_order'),
t2 as (SELECT order_id
FROM   public."User_actions"
WHERE  action = 'cancel_order')

SELECT COUNT(ua.order_id) FILTER (WHERE ua.action = 'cancel_order') as orders_canceled,
       COUNT(ca.order_id) FILTER (WHERE ca.action = 'deliver_order' and ua.action = 'cancel_order') as orders_canceled_and_delivered
FROM   public."Courier_actions" ca
INNER JOIN public."User_actions" ua
ON ca.order_id = ua.order_id
```
4. Подсчет количества товаров в каждом заказе, применение к этим значениям группировки и рассчет количества заказов в каждой группе. Учитываются только заказы, оформленные по будням. В результат включены только те размеры заказов, общее число которых превышает 2000. 
```sql
SELECT array_length(product_ids, 1) as order_size,
       COUNT(*) as orders_count
FROM   public."Orders"
WHERE  to_char(creation_date, 'Dy') not in ('Sat', 'Sun')
GROUP BY order_size having count(*) > 2000
ORDER BY order_size
```

Далее я занялся наполнением этой прекрасной базы данных. Я не стал делать хранимые процедуры, а обратился к языку Python. Числовые данные генерировались с помощью функций библиотеки random, тогда как текстовые данные - посредством библиотеки faker. С самими скриптами можно ознакомиться, если открыть файл [DB_data_generation.ipynb](https://github.com/GitMytth/DB-in-Enterprise/blob/main/ЛР%201/DB_data_generation.ipynb).

Далее эти данные были сохранены в базу данных посредством библиотеки psycopg2. Со скриптами записи данных в БД можно также ознакомиться, если открыть файл [DB_data_generation.ipynb](https://github.com/GitMytth/DB-in-Enterprise/blob/main/ЛР%201/DB_data_generation.ipynb).

Далее была протестирована работа всех запросов при стандартной конфигурации сервера. Были получены следующие показатели времени выполнения запросов (усредненные):

|Номер запроса|1|2|3|4|
|-------------|-|-|-|-|
|Время выполнения, мс|1005|3952|1120|571|

Запрос 2 оказался самым медленным, необходимо его оптимизировать.

Далее я приведу таблицу изменения параметров сервера и изменения времени выполнения запроса. Будем ориентироваться на второй запрос, но заодно и проследим за поведением остальных запросов.

|Эксперимент|Запрос 1 (мс)|Запрос 2 (мс)|Запрос 3 (мс)|Запрос 4 (мс)|random_page_cost|work_mem (Мб)|
|-----------|--------|--------|--------|--------|----------------|------------------|
|0|1005|3952|1120|571|4|4|
|1|508|1632|850|526|2|4|
|2|492|764|783|490|1.1|4|
|3|456|1370|653|478|1.1|64|

В таблице представлены результаты четырех экспериментов, демонстрирующих влияние настроек сервера PostgreSQL (random_page_cost и work_mem) на производительность выполнения запросов. Рассмотрим данные подробно, чтобы понять, как изменения параметров повлияли на время выполнения запросов.

1. Описание параметров
- random_page_cost :
  - Этот параметр определяет относительную стоимость произвольного чтения страницы с диска по сравнению с последовательным чтением. По умолчанию значение равно 4.
  - Чем ниже значение random_page_cost, тем больше оптимизатор PostgreSQL склоняется к использованию индексов и произвольного доступа к данным, предполагая, что это менее затратно.
- work_mem :
  - Этот параметр задает объем памяти, выделяемый для операций сортировки (ORDER BY, DISTINCT) и хэширования (JOIN, GROUP BY).
  - Увеличение значения work_mem позволяет выполнять более сложные операции в памяти, минимизируя использование временных файлов на диске.
Снижение random_page_cost до 2 значительно улучшило производительность всех запросов, особенно запроса 2. Это говорит о том, что оптимизатор начал активнее использовать индексы и произвольный доступ к данным, что оказалось более эффективным для данной рабочей нагрузки. Однако низкое значение work_mem все еще ограничивает производительность операций сортировки и хэширования.
Дальнейшее снижение random_page_cost до 1.1 продолжило улучшение производительности, особенно для запроса 2. Однако прирост стал менее значительным, что может указывать на достижение предела эффективности данного параметра для текущей рабочей нагрузки. 
Увеличение work_mem до 64 МБ немного улучшило производительность запросов 1, 3 и 4. Однако для запроса 2 увеличение work_mem привело к ухудшению времени выполнения, возможно, из-за перерасхода памяти или изменений в плане выполнения запроса. Это подчеркивает важность тщательного подбора значения work_mem под конкретную рабочую нагрузку.

Таким образом, после изменений конфигурации сервера postgres удалось получить следющие ускорения запросов:

||Запрос 1|Запрос 2|Запрос 3|Запрос 4|
|-----------|--------|--------|--------|--------|
|Ускорение|2.2|5.17|1.71|1.19|

Попробуем переписать запрос 2 более оптимально:
```sql
WITH UserOrderCounts AS (
    SELECT 
        user_id,
        COUNT(order_id) AS order_count
    FROM public."User_actions"
    GROUP BY user_id
    HAVING COUNT(order_id) > 5
)
SELECT 
    u.user_id,
    u.fname,
    u.sname,
    uoc.order_count
FROM public."Users" u
INNER JOIN UserOrderCounts uoc 
    ON u.user_id = uoc.user_id
ORDER BY uoc.order_count DESC;
```
Время выполнения запроса снизилось до ~500 мс
