# Секционирование таблицы

Продолжаем пытаться как-то уживаться те что не нравиться - но активно используется.
После очистики старых данных(фактически более чем за 3 месяца они не нужны)
public | notifications      | table  | 18 GB      |

select count(*) from notifications;
count
---------
7517901
(1 row)

```sql
create table notifications (
    id bigserial primary key,
    distribution_id bigint references distributions(id),
    created_at timestamp default now() not null,
    title text not null,
    from_who text not null,
    body text not null,
    send_to text not null,
    channel numeric not null,
    receiver text references users(uid),
    status bigint not null references status_types(id),
    original_body json default '{}'::json not null,
    report text
);
```

После перехода с PostgreSQL 10.23 PostgreSQL 14.5 появился простой план:

- копируем данные из таблицы
- удаляем текущую
- создаем таблицу
- создаем секции по месяцам
- переносим данные (ну мы помним что надо 3 последних месяца - остальное отправляем в архив)
- в коде добавляем обязательное указание дат в запросах (с этмим можно справиться не сильно изменяя код приложения)


```sql
create table notifications_save as (select * from notifications);

drop table notifications;

create table notifications (
    id bigserial,
    distribution_id bigint references distributions(id),
    created_at timestamp default now() not null,
    title text not null,
    from_who text not null,
    body text not null,
    send_to text not null,
    channel numeric not null,
    receiver text references users(uid),
    status bigint not null references status_types(id),
    original_body json default '{}'::json not null,
    report text
) partition by range (created_at);

alter table notifications add constraint pk_notifications primary key (created_at,id);

CREATE TABLE notifications_2023_05 PARTITION OF notifications FOR VALUES FROM ('2023-05-01 00:00:00') to ('2023-06-01 00:00:00');
CREATE TABLE notifications_2023_06 PARTITION OF notifications FOR VALUES FROM ('2023-06-01 00:00:00') to ('2023-07-01 00:00:00');
CREATE TABLE notifications_2023_07 PARTITION OF notifications FOR VALUES FROM ('2023-07-01 00:00:00') to ('2023-08-01 00:00:00');
CREATE TABLE notifications_2023_08 PARTITION OF notifications FOR VALUES FROM ('2023-08-01 00:00:00') to ('2023-09-01 00:00:00');
CREATE TABLE notifications_2023_09 PARTITION OF notifications FOR VALUES FROM ('2023-09-01 00:00:00') to ('2023-10-01 00:00:00');
CREATE TABLE notifications_default PARTITION OF notifications DEFAULT;

```

```sql
notifications-test=# INSERT INTO notifications (SELECT * FROM notifications);
INSERT 0 3000641

notifications-test=# SELECT count(*) FROM notifications;
count
---------
3000641
(1 row)

notifications-test=# SELECT count(*) FROM notifications_save;
count
---------
3000641
(1 row)


INSERT INTO notifications (distribution_id, title, from_who, body, send_to, channel, receiver, status, original_body) VALUES (58109, 'Созданы', 'Reserve', 'Здравствуйте', 'm.simonenko@etpgpb.ru', 2, 'e2fc1ddf-9ef1-4553-97a7-08cc87a6935d', 3, '{}');
notifications-test=# SELECT count(*) FROM notifications_2023_08 WHERE title='Созданы';
count
-------
     2
(1 row)
```

