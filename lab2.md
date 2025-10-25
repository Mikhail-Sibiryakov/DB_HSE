### Задание
Разработайте систему оптимизации распределения самолетов по маршрутам с учетом. Для каждого маршрута определите оптимальный тип самолета, сравнивая фактически используемый с рекомендуемым. Найдите рейсы, где выгодней всего было бы использовать другой самолет, но чтобы качество услуг не изменилось.


### Что вообще нужно?
Итак первая задача, это вообще понять допустимые самолёты для каждого полета.\
Что нам важно чтобы понять, какой самолет можно было бы использовать, а какой нет?
У маршрута есть 4 важных для выбора самолёта характеристики: дальность в км, кол-во мест в самолете каждого из трёх типов "классов обслуживания" (Economy, Comfort, Business)\
Если какой-то самолет имеет расчетную дальность полета больше (пусть на 10%) и мест каждого типо больше либо равно сколько нужно для данного полета - этот самолёт допустим для данного рейса.\
Так же для каждого самолета добавим информацию о средней цене эксплуатации в целом 1 км полета этого самолета. Эти константы взяты откуда-то из гпт, правда или нет не знаю, в целом не очень важно (тут id самолета - цена 1 км в долларах).
```
773 - 26.5
763 - 13.8
SU9 - 3.85
320 - 10.0
321 - 11.9
319 - 8.8
733 - 7.4
CN1 - 2.9
CR2 - 3.8
```
Как теперь понять какой самолет использовать на конкретном маршруте оптимально?\
Просто выберем из всех допустимых самолетов для данного рейса тот, чей км пути дешевле всего.

### SQL запрос, решающий поставленную задачу
```sql
WITH RouteRequirements AS (
    SELECT
        f.flight_id,
        f.aircraft_code AS current_aircraft,
        
        MAX(6371 * 2 * ASIN(
            SQRT(
                POWER(SIN((RADIANS(a_arr.coordinates[1]) - RADIANS(a_dep.coordinates[1])) / 2), 2) + 
                COS(RADIANS(a_dep.coordinates[1])) * COS(RADIANS(a_arr.coordinates[1])) * POWER(SIN((RADIANS(a_arr.coordinates[0]) - RADIANS(a_dep.coordinates[0])) / 2), 2)
            )
        )) AS required_range,
        
        SUM(CASE WHEN tf.fare_conditions = 'Economy' THEN 1 ELSE 0 END) AS required_economy,
        SUM(CASE WHEN tf.fare_conditions = 'Comfort' THEN 1 ELSE 0 END) AS required_comfort,
        SUM(CASE WHEN tf.fare_conditions = 'Business' THEN 1 ELSE 0 END) AS required_business
    FROM
        flights f
    JOIN airports_data a_dep ON f.departure_airport = a_dep.airport_code
    JOIN airports_data a_arr ON f.arrival_airport = a_arr.airport_code
    JOIN ticket_flights tf ON f.flight_id = tf.flight_id
    GROUP BY
        f.flight_id, f.aircraft_code 
),
AircraftCapabilities AS (
    SELECT
        ad.aircraft_code,
        ad.range AS max_aircraft_range,

        SUM(CASE WHEN s.fare_conditions = 'Economy' THEN 1 ELSE 0 END) AS economy_capacity,
        SUM(CASE WHEN s.fare_conditions = 'Comfort' THEN 1 ELSE 0 END) AS comfort_capacity,
        SUM(CASE WHEN s.fare_conditions = 'Business' THEN 1 ELSE 0 END) AS business_capacity,
        
        (CASE ad.aircraft_code
            WHEN '773' THEN 26.5
            WHEN '763' THEN 13.8
            WHEN 'SU9' THEN 3.85
            WHEN '320' THEN 10.0
            WHEN '321' THEN 11.9
            WHEN '319' THEN 8.8
            WHEN '733' THEN 7.4
            WHEN 'CN1' THEN 2.9
            WHEN 'CR2' THEN 3.8
            ELSE NULL 
        END) AS cost_per_km
    FROM
        aircrafts_data ad
    JOIN seats s ON ad.aircraft_code = s.aircraft_code
    GROUP BY
        ad.aircraft_code, ad.range
),
OptimalChoice AS (
    SELECT
        rr.flight_id,
        rr.current_aircraft,
        ac.aircraft_code AS potential_aircraft,
        ac.cost_per_km,

        (ac.max_aircraft_range >= rr.required_range * 1.10) AND
        
        (ac.economy_capacity >= rr.required_economy) AND
        (ac.comfort_capacity >= rr.required_comfort) AND
        (ac.business_capacity >= rr.required_business)
        AS is_feasible
        
    FROM
        RouteRequirements rr
    CROSS JOIN
        AircraftCapabilities ac
),
FinalRanking AS (
    SELECT
        oc.flight_id,
        oc.current_aircraft,
        oc.potential_aircraft,
        oc.cost_per_km,
        
        ROW_NUMBER() OVER (
            PARTITION BY oc.flight_id
            ORDER BY oc.cost_per_km ASC
        ) as rn_rank
    FROM
        OptimalChoice oc
    WHERE
        oc.is_feasible = TRUE
)
SELECT
    f.flight_id,
    f.flight_no AS "Номер рейса",
    ad_current.model AS "Используемый самолет (Модель)",
    ad_optimal.model AS "Рекомендуемый Самолет (Модель)",
    fr.cost_per_km AS "Оптимальная стоимость 1км"
FROM
    FinalRanking fr
JOIN
    flights f ON fr.flight_id = f.flight_id
JOIN
    aircrafts_data ad_current ON fr.current_aircraft = ad_current.aircraft_code
JOIN
    aircrafts_data ad_optimal ON fr.potential_aircraft = ad_optimal.aircraft_code
WHERE
    fr.rn_rank = 1
	-- Если раскомментировать строку ниже, то останутся только те рейсы где
	-- рекомендуемый самолет не совпадает с используемым
	-- AND ad_current.model != ad_optimal.model
ORDER BY
    f.flight_id;
```

### Пояснение к запросу (что там вообще происходит)
В запросе удобно было создать несколько временных таблиц, чтобы потом получить итоговую нужную выборку.
## RouteRequirements
Это временная табличка в которой содержится информация о характеристиках самолета, который нужен для каждого рейса. (строка это рейс, поля в ней - id рейса, текущий самолет, расстояние которое нужно пролететь и сколько мест каждого класса было куплено пассажирами на это рейс).\
С помощью JOIN присоединяем информацию об аэропортах вылета и прибытия - это нужно чтобы узнать их координаты (по формуле Гаверсинуса считаем расстояние между точками на шарике, 6371 - это константа радиус Земли), и также с помощью JOIN присоединяем для каждого рейса все билеты купленные на этот рейс. И так как для одного рейса билетов может быть много, то это "раздувает" табличку, и в итоге больше не выполняется правило "одна строка - один рейс", и именно по этому делаем GROUP BY чтобы обратно сжать все рейсы в одни, попутна сагрегировав кол-во мест разных классов обслуживания (так как много строк для каждого рейса появилось именно из-за того что было много билетов для этого рейса). Так же пришлось брать расстояние агрегатной функцией, так как у нас для одного рейса целая куча строк. MAX здесь не принципиален, так как у всех одинаковых рейсов расстояние одинаково, поэтому нужно взять одно из них просто (подойдут MAX, MIN, AVG например).

## AircraftCapabilities
Это временная таблица с описанием самолетов. Тут строка - это самолет, в колонках его "характеристики" - код, макс. расстояние полета, кол-во мест каждого класса обслуживания, цена 1 км.\
С помощью JOIN подтянули для каждого самолета все его места - так как нам же нужно понять, сколько мест в нем разных классов обслуживания. Аналогично предыдущему случаю табличка "распухла" из-за кучи мест в одном самолете, аналогично сжали её обратно с помощью GROUP BY, попутно подсчитав кол-во мест каждого типа для самолета, и так же для каждого самолета просто по сути заifали стоимость его 1 км пути (тут кстати никакой констыль в виде агрегатной функции MAX не нужен, так как cost_per_km вычисляется только на основе атрибута ad.aircraft_code в которому и делается GROUP BY, то есть cost_per_km в каждой группе гарантировано одинаков не только из смысла наших данных, но и с точки зрения запроса SQL никак иначе получится и не могло).

## OptimalChoice
В этой временной табличке хранятся все возможные пары "рейс-самолет" (где рейс и самолет это конечно же строки из таблиц, содержащие какие-то атрибуты рейса и самолета соответственно). Именно поэтому тут используется CROSS JOIN (декартово произведение), чтобы получить табличку со всеми парами.\
Итак, каждая строка - это id рейса, текущий самолет, потенциальный, стоимость 1км пути для потенциального самолета, и булевая переменная (можно ли вообще использовать данных самолет на этом рейсе? Должен долететь по расстоянию и вместить всех пассажиров, притом каждого в нужном классе обслуживания).

## FinalRanking
Эта почти такая же табличка, как и предыдущая, только в ней оставили только те пары "рейс-самолет" которые теоретически допустимы (самолет подходит для данного рейса) и притом с помощью оконной функции для каждого рейса проранжировали самолеты от 1 до k, где 1 получил допустимый самолет с самой низкой стоимостью 1 км полета (а k - это количество всего допустимых самолетов для данного рейса).

## SELECT
И наконец финальный SELECT. Дело за малым - нужно из предыдущей временной таблички просто вынуть те строки, где самолет имеет рейтинг 1 - то есть лучший по цене за 1км пути для данного маршрута.\
JOIN'ами подтягиваем информацию о номере рейса, а так же через id самолетов получаем строки с описанием самолетов, и из них берем названия текущего самолета и рекомендуемого.