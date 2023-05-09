# Airline flights SQL analysis project
## **General information**
This analysis was done as the final test of an SQL course.

PostgreSQL and DBeaver were used to complete the tasks.

The database analysed in the project is a PostgreSQL demonstration database, the subject field of which is airline flights across various airports.
Full description of the database can be found [here](https://www.postgrespro.ru/education/demodb).

## **Description of tables used**

To get the ganaral idea about the structure of the tables neccessary to complete the task were used the query below and the database documentation.
```postgresql
select column_name, data_type 
from information_schema.columns 
where table_name = table_name
```

### **Schema Diagram**
'#' mark primary keys.

![Schema Diagram](https://repo.postgrespro.ru/doc//std/10.23.1/en/html/demodb-bookings-schema.svg)

***

### seats
Seats define the cabin configuration of each aircraft model. Each seat is defined by its number (*seat_no*) and has an assigned travel class (*fare_conditions*): *Economy*, *Comfort* or *Business*.

| Column          |     Type    | Modifiers    |      Description    |
|-----------------|-------------|--------------|---------------------|
| aircraft_code   | char(3)     | not null     | Aircraft code, IATA |
| seat_no         | varchar(4)  | not null     | Seat number         |
| fare_conditions | varchar(10) | not null     | Travel class        |

***

### flights
A flight always connects two points — the airport of departure (*departure_airport*) and arrival (*arrival_airport*). 

Each flight has a scheduled date and time of departure (*scheduled_departure*) and arrival (*scheduled_arrival*). The actual departure time (*actual_departure*) and arrival time (*actual_arrival*) can differ.

Flight status (*status*) can take one of the following values: Scheduled, On Time, Delayed, Departed, Arrived, Cancelled.

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

A flight segment connects a ticket with a flight and is identified by their numbers.

|     Column      |     Type      | Modifiers    |    Description      |
|-----------------|---------------|--------------|---------------------|
| ticket_no       | char(13)      | not null     | Ticket number       |
| flight_id       | integer       | not null     | Flight ID           |
| fare_conditions | varchar(10)   | not null     | Travel class        |
| amount          | numeric(10,2) | not null     | Travel cost         |

***

### airports

An airport is identified by a three-letter code (*airport_code*) and has a name (*airport_name*).

| Column       |  Type   | Modifiers    |                 Description                 |
|--------------|---------|--------------|---------------------------------------------|
| airport_code | char(3) | not null     | Airport code                                |
| airport_name | text    | not null     | Airport name                                |
| city         | text    | not null     | City                                        |
| coordinates  | point   | not null     | Airport coordinates (longitude and latitude)|
| timezone     | text    | not null     | Airport time zone                           |

***

### boarding_passes

Boarding passes are assigned sequential numbers (*boarding_no*), in the order of check-ins for the flight (this number is unique only within the context of a particular flight). The boarding pass specifies the seat number (*seat_no*).

| Column      |    Type    | Modifiers    |         Description      |
|-------------|------------|--------------|--------------------------|
| ticket_no   | char(13)   | not null     | Ticket number            |
| flight_id   | integer    | not null     | Flight ID                |
| boarding_no | integer    | not null     | Boarding pass number     |
| seat_no     | varchar(4) | not null     | Seat number              |

***
### tickets

A ticket has a unique number (*ticket_no*) that consists of 13 digits.

The ticket includes a passenger ID (*passenger_id*) — the identity document number, — their first and last names (*passenger_name*), and contact information (*contact_data*).

|    Column      |     Type    | Modifiers    |           Description	      |
|----------------|-------------|--------------|-------------------------------|
| ticket_no      | char(13)    | not null     | Ticket number				  |
| book_ref       | char(6)     | not null     | Booking number				  |
| passenger_id   | varchar(20) | not null     | Passenger ID 				  |
| passenger_name | text        | not null     | Passenger name				  |
| contact_data   | jsonb       |              | Passenger contact information |

***

## **Tasks**

### **Query 1**
*Which aircrafts have more than 50 seats?*

All the necessary infirmation can be found in the table **seats**.

```postgresql
select aircraft_code, count(seat_no) no_seats
from seats
group by aircraft_code
having count(seat_no) > 50
```

There are 7 aircrafts that fit the description.

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

### **Query 2**
*Which airports have flights within which you can get a business class cheaper than an economy class?*

As there were no additional constraints on whether a buiseness class ticket sholud be cheaper than **every** economy class ticket of the same flight, it was assumed here that it is enough for it to be cheaper than **at least one** economy class ticket.

Thus, it is firts necessary to find the minimum cost of a buiseness class ticket for each flight in the table **ticket_flights**:

```postgresql
select flight_id flight_id_b, 
		min(amount) amount_business
from ticket_flights
where fare_conditions = 'Business'
group by flight_id, fare_conditions
```
as well as the maximum costs of economy class tickets:

```postgresql
select flight_id flight_id_e, 
	    max(amount) amount_economy
from ticket_flights
where fare_conditions = 'Economy'
group by flight_id, fare_conditions
```
Further on, we need to join the two tabels above and leave only those lines, where the cheapest buiseness class tickets are cheaper than the most expensive economy class tickets:

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
As a final step we need to add from the table **flights** the information about the flights on which these tickets were found.

The final query:

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
Logically, there no such flights where you can get a business class cheaper than an economy class.

***

### **Query 3**
*Are there aircrafts that do not have a business class?*

All the necessary infirmation can be found in the table **seats**.

```postgresql
select aircraft_code, array_agg(distinct fare_conditions) fare_conditions_options
from seats
group by aircraft_code
having not 'Business' = any(array_agg(distinct fare_conditions))
```
There are two such aircrafts.

|aircraft_code|fare_conditions_options|
|-------------|-----------------------|
|CN1          |{Economy}              |
|CR2          |{Economy}              |

***

### **Query 4**
*Find the number of occupied seats for each flight, the percentage of the number of occupied seats to the total number of seats on the plane; add a cumulative total of passengers taken out for each airport for each day.*

First, we need to find the total number of seats for each flight from the table **seats**:

```postgresql
select aircraft_code, count(seat_no) total_seats
from seats
group by aircraft_code
```
Second, it is necessary to find the number of occupied seats, which can be counted using the information in the table **boarding_passes**:

```postgresql
select flight_id, count(seat_no) occupied_seats
from boarding_passes
group by flight_id
```
Now, it is possible to find the percentage of the number of occupied seats to the total number of seats on the plane and complete it with the information about flights from the table **flights**: 

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
The last step is to add a cumulative total of passengers taken out for each airport for each day.

Final query:

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
Top 5 lines from the result:
|departure_airport|actual_departure|flight_id|occupied_seats|total_seats|percent_occupied|sum_passengers_departed|
|-----------------|----------------|---------|--------------|-----------|----------------|-----------------------|
|AAQ|2016-09-13|21103|3|97|3.09|3|
|AAQ|2016-09-13|20993|51|130|39.23|54|
|AAQ|2016-09-14|21096|3|97|3.09|3|
|AAQ|2016-09-14|20982|50|130|38.46|53|
|AAQ|2016-09-15|21084|5|97|5.15|5|

***

### **Query 5**
*Find the percentage of flights on routes from the total number of flights. Output the names of the airports and the percentage as a result.*

All the necessary information can be calculated using the table **flights**.

```postgresql
select distinct f.departure_airport, f.arrival_airport,
	round(
		count(flight_id) over (partition by f.departure_airport, 
            f.arrival_airport)::numeric /
		count(*) over() * 100, 3) percent_of_total_flights
from flights f
order by f.departure_airport, f.arrival_airport
```

Top 5 lines from the result:
|departure_airport|arrival_airport|percent_of_total_flights|
|-----------------|---------------|------------------------|
|AAQ|EGO|0.184|
|AAQ|NOZ|0.027|
|AAQ|SVO|0.184|
|ABA|ARH|0.024|
|ABA|DME|0.054|

***

### **Query 6**
*Print the number of passengers for each mobile operator code, given that the operator code is three characters after +7*

First, it is necessary to extract mobile operator codes from each passenger phone number in the table **tickets**

```postgresql
select passenger_id, substring(contact_data ->> 'phone' from 3 for 3) code
from tickets)
```
Now, we can group the results by code.

Final query:

```postgresql
select code, count(passenger_id) passenger_count
from ( 
	select passenger_id, substring(contact_data ->> 'phone' from 3 for 3) code
	from tickets) t
group by code
order by code
```
Top 5 lines from the result:

|code|passenger_count|
|----|---------------|
|000|3540|
|001|3679|
|002|3733|
|003|3652|
|004|3777|

***

### **Query 7**
*Between which cities there are no flights?*

First step is to get all the combinations of cities from table **airports**:

```postgresql
select a1.city city1, a2.city city2
from airports a1, airports a2
where a1.city < a2.city
order by a1.city, a2.city
```
Next step is to find all the distict combinations of departure and arrival cities from the tables **flights** adding city names from the table **airports**

```postgresql
select a1.city city1, a2.city city2
from(
    select distinct departure_airport, arrival_airport
    from flights
    order by departure_airport, arrival_airport) t
join airports a1 on t.departure_airport = a1.airport_code 
join airports a2 on t.arrival_airport = a2.airport_code
```
Final step is to substract from all the possible combinations of cities the ones between which there are existing flights.


Final query:

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
Top 5 lines from the result:
|city1|city2|
|-----|-----|
|Норильск|Череповец|
|Краснодар|Ханты-Мансийск|
|Новокузнецк|Омск|
|Новокузнецк|Ханты-Мансийск|
|Нефтеюганск|Нижнекамск|

There are 4792 such cities.

***

### **Query 8**
*Classify financial turnover (sum of ticket prices) by routes:*
- *Up to 50 million - low*
- *From 50 million inclusive to 150 million - middle*
- *From 150 million inclusive - high*

*Output the number of routes in each class as a result.*

First, we need to find the sum of ticket prices for each flight from the table **ticket_flights**:

```postgresql
select flight_id, sum(amount) sum_flight
from ticket_flights
group by flight_id
```
Then, group and add the sums by routes (this step was be formated as a cte):

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
One more cte was used to add ranks (high, medium, low) to the financial turnovers calculated in the previous step:

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
Finally, the number of routes in each class is counted.

Final query:

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
There are significantly more low turnover routes:
|rank|count_rank|
|----|----------|
|low|355|
|middle|77|
|high|25|

As to why it is so, further investigation is necessary.

***

### **Query 9**
*Find pairs of cities between which the distance is more than 5000 km*

With the help of functions ```sind```, ```cosd``` and ```acos``` and information from the table **airports** it is possible to calculate distances between all pairs of sities.

*d * 6371* is used to convert radian distance into kilometers.

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
Final step is to filter the results of the previous query:
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

Top 5 lines from the result:
|city1|city2|distance|
|-----|-----|--------|
|Анапа|Благовещенск|6338.85|
|Анапа|Нерюнгри|5836.94|
|Анапа|Магадан|6881.52|
|Анапа|Чита|5389.83|
|Анапа|Хабаровск|6919.42|

There are 511 such pairs of cities.
