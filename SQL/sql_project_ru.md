# SQL проект по анализу авиаперевозок
## **Общая информация**
Данный анализ являлся итоговой работой по курсу SQL.

Для выполнения работы были использованы PostgreSQL и DBeaver.

База данных используемая в проектеявляется демонстрационной базой данных для СУБД PostgreSQL. В качестве предметной области в ней выбраны авиаперевозки по России.
Полное описание базы данных можно найти [здесь](https://www.postgrespro.ru/education/demodb).

## **Описание использованных таблиц**

Для получения общего представления о структуре таблиц, необходимых для выполнения заданий, были использованы документация базы данных и следующий запрос:
```postgresql
select column_name, data_type 
from information_schema.columns 
where table_name = table_name
```

### **Диаграмма схемы данных**
'#' показывает первичный ключ.

![Диаграмма схемы данных](https://repo.postgrespro.ru/doc//std/10.23.1/en/html/demodb-bookings-schema.svg)

***

### seats
Места определяют схему салона каждой модели. Каждое место определяется своим номером (*seat_no*) и имеет закрепленный за ним класс обслуживания (*fare_conditions*): *Economy*, *Comfort* или *Business*.

| Column          |     Type    | Modifiers    |      Description    |
|-----------------|-------------|--------------|---------------------|
| aircraft_code   | char(3)     | not null     | Aircraft code, IATA |
| seat_no         | varchar(4)  | not null     | Seat number         |
| fare_conditions | varchar(10) | not null     | Travel class        |

***

### flights
Рейс всегда соединяет две точки — аэропорты вылета (*departure_airport*) и прибытия (*arrival_airport*). 

У каждого рейса есть запланированные дата и время вылета (*scheduled_departure*) и прибытия (*scheduled_arrival*). Реальные время вылета (*actual_departure*) и прибытия (*actual_arrival*) могут отличаться.

Статус рейса (*status*) может принимать одно из следующих значений: 
- *Scheduled* - Рейс доступен для бронирования
- *On Time* - Рейс доступен для регистрации и не задержан
- *Delayed* - Рейс доступен для регистрации, но задержан 
- *Departed* - Самолет уже вылетел и находится в воздухе
- *Arrived* - Самолет прибыл в пункт назначения
- *Cancelled* - Рейс отменен.

|        Column       |     Type    | Modifiers    |         Description
|--------------------:|------------:|-------------:|----------------------------:|
| flight_id           | serial      | not null     | Flight ID                   |
| flight_no           | char(6)     | not null     | Flight number               |
| scheduled_departure | timestamptz | not null     | Scheduled departure time    |
| scheduled_arrival   | timestamptz | not null     | Scheduled arrival time      |
| departure_airport   | char(3)     | not null     | Airport of departure        |
| arrival_airport     | char(3)     | not null     | Airport of arrival          |
| status              | varchar(20) | not null     | Flight status               |
| aircraft_code       | char(3)     | not null     | Aircraft code, IATA         |
| actual_departure    | timestamptz |              | Actual departure time       |
| actual_arrival      | timestamptz |              | Actual arrival time         |
 
***

### ticket_flights

Перелет соединяет билет с рейсом и идентифицируется их номерами.

|     Column      |     Type      | Modifiers    |    Description      |
|-----------------|---------------|--------------|---------------------|
| ticket_no       | char(13)      | not null     | Ticket number       |
| flight_id       | integer       | not null     | Flight ID           |
| fare_conditions | varchar(10)   | not null     | Travel class        |
| amount          | numeric(10,2) | not null     | Travel cost         |

***

### airports

Аэропорт идентифицируется трехбуквенным кодом (*airport_code*) и имеет свое имя (*airport_name*).

| Column       |  Type   | Modifiers    |                 Description                 |
|--------------|---------|--------------|---------------------------------------------|
| airport_code | char(3) | not null     | Airport code                                |
| airport_name | text    | not null     | Airport name                                |
| city         | text    | not null     | City                                        |
| coordinates  | point   | not null     | Airport coordinates (longitude and latitude)|
| timezone     | text    | not null     | Airport time zone                           |

***

### boarding_passes

Посадочным талонам присваиваются последовательные номера (*boarding_no*), в порядке регистрации пассажиров на рейс (этот номер будет уникальным только в пределах данного рейса). В посадочном талоне указывается номер места (*seat_no*).

| Column      |    Type    | Modifiers    |         Description      |
|-------------|------------|--------------|--------------------------|
| ticket_no   | char(13)   | not null     | Ticket number            |
| flight_id   | integer    | not null     | Flight ID                |
| boarding_no | integer    | not null     | Boarding pass number     |
| seat_no     | varchar(4) | not null     | Seat number              |

***
### tickets

Билет имеет уникальный номер (*ticket_no*), состоящий из 13 цифр.

Билет содержит идентификатор пассажира (*passenger_id*)— номер документа, удостоверяющего личность, — его фамилию и имя (*passenger_name*) и контактную информацию (*contact_data*).

|    Column      |     Type    | Modifiers    |           Description	      |
|----------------|-------------|--------------|-------------------------------|
| ticket_no      | char(13)    | not null     | Ticket number				  |
| book_ref       | char(6)     | not null     | Booking number				  |
| passenger_id   | varchar(20) | not null     | Passenger ID 				  |
| passenger_name | text        | not null     | Passenger name				  |
| contact_data   | jsonb       |              | Passenger contact information |

***

## **Задания**

### **Задание 1**
*Какие самолеты имеют более 50 посадочных мест?*

Вся информация, необходимая для выполнения запроса, содержится в таблице **seats**.

```postgresql
select aircraft_code, count(seat_no) no_seats
from seats
group by aircraft_code
having count(seat_no) > 50
```

7 самолетов соответствуют условиям.

|aircraft_code|no_seats|
|-------------|--------|
|320|140|
|773|402|
|763|222|
|319|116|
|733|130|
|SU9|97|
|321|170|

***

### **Задание 2**
*В каких аэропортах есть рейсы, в рамках которых можно добраться бизнес-классом дешевле, чем эконом-классом?*

Так как в условиях задания нет уточнения, должен ли билет бизнес-класса быть дешевле **всех** билетов эконом-класса на данном рейсе, в запросе ниже подразумевается, что билету бизнес-класса достаточно быть дешевле **хотя бы одного** билета эконом-класса.

Таким образом, в первую очередь необходимо найти минимальную стоимость билета бизнес-класса для каждого рейса из таблицы **ticket_flights**:

```postgresql
select flight_id flight_id_b, 
		min(amount) amount_business
from ticket_flights
where fare_conditions = 'Business'
group by flight_id, fare_conditions
```
а также, максимальную стоимость билета эконом-класса:

```postgresql
select flight_id flight_id_e, 
	    max(amount) amount_economy
from ticket_flights
where fare_conditions = 'Economy'
group by flight_id, fare_conditions
```
Далее, две таблицы, полученные выше были объединены и отфильтрованы так, чтобы остались только строчки, в которых самый дешевый билет бизнес-класса дешевле самого дорогого билета эконом-класса:

```postgresql
select *
from (
    select flight_id flight_id_b, 
            min(amount) amount_business
    from ticket_flights
    where fare_conditions = 'Business'
    group by flight_id, fare_conditions
    ) b 
left join (
    select flight_id flight_id_e, 
            max(amount) amount_economy
    from ticket_flights
    where fare_conditions = 'Economy'
    group by flight_id, fare_conditions
    ) e 
on b.flight_id_b = e.flight_id_e 
where b.amount_business < e.amount_economy
```
В заключение, необходимо добавить из таблицы **flights** информацию о рейсах на которых были найдены данные билеты.

Полный запрос:

```postgresql
with cte as (
	select *
	from (
		select flight_id flight_id_b, 
	    		min(amount) amount_business
		from ticket_flights
		where fare_conditions = 'Business'
		group by flight_id, fare_conditions
		) b 
	left join (
		select flight_id flight_id_e, 
	    		max(amount) amount_economy
		from ticket_flights
		where fare_conditions = 'Economy'
		group by flight_id, fare_conditions
	) e 
    on b.flight_id_b = e.flight_id_e 
	where b.amount_business < e.amount_economy
)
select f.departure_airport, f.arrival_airport, f.flight_id, 
		c.amount_business, c.amount_economy 
from flights f
join cte c on c.flight_id_b = f.flight_id
order by f.departure_airport, f.arrival_airport, f.flight_id
```
Как и следовало предполагать, рейсы, в рамках которых можно добраться бизнес-классом дешевле, чем эконом-классом, не были найдены.

***

### **Задание 3**
*Есть ли самолеты, не имеющие бизнес - класса?*

Вся информация, необходимая для выполнения запроса, содержится в таблице **seats**.

```postgresql
select aircraft_code, array_agg(distinct fare_conditions) fare_conditions_options
from seats
group by aircraft_code
having not 'Business' = any(array_agg(distinct fare_conditions))
```
Были найдены два таких самолета.

|aircraft_code|fare_conditions_options|
|-------------|-----------------------|
|CN1          |{Economy}              |
|CR2          |{Economy}              |

***

### **Задание 4**
*Найдите количество занятых мест для каждого рейса, процентное отношение количества занятых мест к общему количеству мест в самолете, добавьте накопительный итог вывезенных пассажиров по каждому аэропорту на каждый день.*

Для начала, необходимо найти общее количество мест в самолете в таблице **seats**:

```postgresql
select aircraft_code, count(seat_no) total_seats
from seats
group by aircraft_code
```
Далее, найдем количество занятых мест, используя информацию из таблицы **boarding_passes**:

```postgresql
select flight_id, count(seat_no) occupied_seats
from boarding_passes
group by flight_id
```
Теперь стало возможным найти процентное отношение количества занятых мест к общему количеству мест в самолете и дополнить полученные данные информацией о рейсах из таблицы **flights**: 

```postgresql
select f.departure_airport, f.actual_departure::date, f.flight_id, 
        t.occupied_seats, t2.total_seats,
		round(t.occupied_seats::numeric / t2.total_seats * 100, 2) percent_occupied
from flights f
join (
	select flight_id, count(seat_no) occupied_seats
	from boarding_passes
	group by flight_id
) t on f.flight_id = t.flight_id
join (
	select aircraft_code, count(seat_no) total_seats
	from seats
	group by aircraft_code
) t2 on f.aircraft_code = t2.aircraft_code
```
Последний этап - добавление накопительного итога вывезенных пассажиров по каждому аэропорту на каждый день.

Полный запрос:

```postgresql
select f.departure_airport, f.actual_departure::date, f.flight_id, 
        t.occupied_seats, t2.total_seats,
		round(t.occupied_seats::numeric / t2.total_seats * 100, 2) 
            as percent_occupied
        sum(t.occupied_seats) over (partition by f.departure_airport, 
            f.actual_departure::date order by f.actual_departure)
			as sum_passengers_departed
from flights f
join (
	select flight_id, count(seat_no) occupied_seats
	from boarding_passes
	group by flight_id
) t on f.flight_id = t.flight_id
join (
	select aircraft_code, count(seat_no) total_seats
	from seats
	group by aircraft_code
) t2 on f.aircraft_code = t2.aircraft_code
```
Первые 5 строк результата:

|departure_airport|actual_departure|flight_id|occupied_seats|total_seats|percent_occupied|sum_passengers_departed|
|-----------------|----------------|---------|--------------|-----------|----------------|-----------------------|
|AAQ|2016-09-13|21103|3|97|3.09|3|
|AAQ|2016-09-13|20993|51|130|39.23|54|
|AAQ|2016-09-14|21096|3|97|3.09|3|
|AAQ|2016-09-14|20982|50|130|38.46|53|
|AAQ|2016-09-15|21084|5|97|5.15|5|

***

### **Задание 5**
*Найдите процентное соотношение перелетов по маршрутам от общего количества перелетов. Выведите в результат названия аэропортов и процентное отношение.*

Вся информация, необходимая для выполнения запроса, содержится в таблице **flights**.

```postgresql
select distinct f.departure_airport, f.arrival_airport,
	round(
		count(flight_id) over (partition by f.departure_airport, 
            f.arrival_airport)::numeric /
		count(*) over() * 100, 3) percent_of_total_flights
from flights f
order by f.departure_airport, f.arrival_airport
```

Первые 5 строк результата:

|departure_airport|arrival_airport|percent_of_total_flights|
|-----------------|---------------|------------------------|
|AAQ|EGO|0.184|
|AAQ|NOZ|0.027|
|AAQ|SVO|0.184|
|ABA|ARH|0.024|
|ABA|DME|0.054|

***

### **Задание 6**
*Выведите количество пассажиров по каждому коду сотового оператора, если учесть, что код оператора - это три символа после +7*

Сначала необходимо выделить код оператора из телефонных номеров пассажиров, находящихся в таблице **tickets**

```postgresql
select passenger_id, substring(contact_data ->> 'phone' from 3 for 3) code
from tickets)
```
Теперь можно сгруппировать результаты по коду.

Полный запрос:

```postgresql
select code, count(passenger_id) passenger_count
from ( 
	select passenger_id, substring(contact_data ->> 'phone' from 3 for 3) code
	from tickets) t
group by code
order by code
```
Первые 5 строк результата:

|code|passenger_count|
|----|---------------|
|000|3540|
|001|3679|
|002|3733|
|003|3652|
|004|3777|

***

### **Задание 7**
*Между какими городами не существует перелетов?*

В первую очередь, найдем все возможные комбинации городов (без повторов) из таблицы **airports**:

```postgresql
select a1.city city1, a2.city city2
from airports a1, airports a2
where a1.city < a2.city
order by a1.city, a2.city
```
Далее, найдем все уникальные комбинации аэропортов вылета и прибытия из таблицы **flights**, добавив названия городов из таблицы **airports**

```postgresql
select a1.city city1, a2.city city2
from(
    select distinct departure_airport, arrival_airport
    from flights
    order by departure_airport, arrival_airport) t
join airports a1 on t.departure_airport = a1.airport_code 
join airports a2 on t.arrival_airport = a2.airport_code
```
В заключение, вычтем из всех возможных комбинаций городов те, между которыми существуют перелеты.


Полный запрос:

```postgresql
select distinct *
from (
	select a1.city city1, a2.city city2
	from airports a1, airports a2
	where a1.city < a2.city
	order by a1.city, a2.city) t1
except 
select *
from (
	select a1.city city1, a2.city city2
	from(
		select distinct departure_airport, arrival_airport
		from flights
		order by departure_airport, arrival_airport) t
	join airports a1 on t.departure_airport = a1.airport_code 
	join airports a2 on t.arrival_airport = a2.airport_code) t2
```
Первые 5 строк результата:
|city1|city2|
|-----|-----|
|Норильск|Череповец|
|Краснодар|Ханты-Мансийск|
|Новокузнецк|Омск|
|Новокузнецк|Ханты-Мансийск|
|Нефтеюганск|Нижнекамск|

Существует 4792 таких пар городов.

***

### **Задание 8**
*Классифицируйте финансовые обороты (сумма стоимости билетов) по маршрутам:*
- *До 50 млн - low*
- *От 50 млн включительно до 150 млн - middle*
- *От 150 млн включительно - high*

*Выведите в результат количество маршрутов в каждом классе.*

Сначала, необходимо найти сумму стоимости билетов для каждого перелета из таблицы **ticket_flights**:

```postgresql
select flight_id, sum(amount) sum_flight
from ticket_flights
group by flight_id
```
Далее, сгуппируем их и добавим суммы по маршрутам (этот шаг был оформлен как cte):

```postgresql
with rs as (
	select f.departure_airport, f.arrival_airport, sum(tf.sum_flight) sum_route
	from flights f
	join (
		select flight_id, sum(amount) sum_flight
		from ticket_flights
		group by flight_id
	) tf on tf.flight_id = f.flight_id
	group by f.departure_airport, f.arrival_airport
	order by f.departure_airport, f.arrival_airport
)
```
Еще один cte был использован для классификации (high, medium, low) финансовых оборотов, подсчитанных на предыдущем этапе:

```postgresql
with ranked as (
	select rs.*, 
		case
			when rs.sum_route < 50000000 then 'low'
			when rs.sum_route >= 50000000 and rs.sum_route < 150000000 then 'middle'
			else 'high'
		end as "rank"
	from rs
)
```
Заключительный этап - подсчет количества маршрутов в каждом классе.

Полный запрос:

```postgresql
with rs as (
	select f.departure_airport, f.arrival_airport, sum(tf.sum_flight) sum_route
	from flights f
	join (
		select flight_id, sum(amount) sum_flight
		from ticket_flights
		group by flight_id
	) tf on tf.flight_id = f.flight_id
	group by f.departure_airport, f.arrival_airport
	order by f.departure_airport, f.arrival_airport
), 
ranked as (
	select rs.*, 
		case
			when rs.sum_route < 50000000 then 'low'
			when rs.sum_route >= 50000000 and rs.sum_route < 150000000 then 'middle'
			else 'high'
		end as "rank"
	from rs
)
select "rank", 
	count(sum_route) count_rank
from ranked
group by "rank"
order by count_rank desc
```
Финансовых оборотов, классифицированных как low, значительно больше:

|rank|count_rank|
|----|----------|
|low|355|
|middle|77|
|high|25|

Для обнаружения причины необходимо дальнейшее исследование.

***

### **Задание 9**
*Выведите пары городов между которыми расстояние более 5000 км*

С помощью функций ```sind```, ```cosd``` и ```acos``` и информации из таблицы **airports** возможно подсчитать расстояния между всеми парами городов.

*d * 6371* использовано для перевода расстояния из радиан в километры.

```postgresql
select *, d * 6371 as l
from (
    select a1.city city1, a2.city city2,
        acos(sind(a1.latitude)*sind(a2.latitude) + cosd(a1.latitude)*cosd(a2.latitude)*cosd(a1.longitude - a2.longitude)) d
    from airports a1, airports a2
    where a1.city < a2.city
    order by a1.airport_code, a2.airport_code
)
```

После чего остается отфильтровать результаты предыдущего подзапроса:

```postgresql
with dist as (
	select *, d * 6371 as l
	from (
		select a1.city city1, a2.city city2,
			acos(sind(a1.latitude)*sind(a2.latitude) + cosd(a1.latitude)*cosd(a2.latitude)*cosd(a1.longitude - a2.longitude)) d
		from airports a1, airports a2
		where a1.city < a2.city
		order by a1.airport_code, a2.airport_code
	) t
)
select city1, city2, round(l::numeric, 2) distance
from dist
where l > 5000
```

Первые 5 строк результата:
|city1|city2|distance|
|-----|-----|--------|
|Анапа|Благовещенск|6338.85|
|Анапа|Нерюнгри|5836.94|
|Анапа|Магадан|6881.52|
|Анапа|Чита|5389.83|
|Анапа|Хабаровск|6919.42|

Было найдено 511 таких пар городов.
