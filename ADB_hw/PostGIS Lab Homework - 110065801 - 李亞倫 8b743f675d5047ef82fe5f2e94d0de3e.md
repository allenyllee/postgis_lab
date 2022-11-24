# PostGIS Lab Homework - 110065801 - 李亞倫

## A. (30%) Answer the following questions with spatial SQL queries (Using multiple queries is allowed). You need to explain your solutions and show the results of the queries. Please make sure you load the data that is the same as data used in the lab.

1. The subway routes are categorized by different colors, and each subway station can be passed through by one or several color routes. For each color route r, calculate the total number of populations living in the census blocks that lie within 200 meters of all subway stations that r pass through.
    
    Originally, the color format looks like this:
    
    ```sql
    SELECT DISTINCT color
    FROM nyc_subway_stations;
    ```
    
    ```sql
    "BLUE-GREY"
    "PURPLE"
    "GREY-PURPLE-YELLOW"
    "PURPLE-YELLOW"
    "GREY-ORANGE"
    "LIME"
    "BROWN"
    "BROWN-ORANGE-YELLOW"
    "ORANGE"
    "RED"
    "BROWN-ORANGE"
    "BROWN-YELLOW"
    "BLUE-ORANGE"
    "GREEN-RED"
    "RED-GREEN"
    "BLUE"
    "GREEN"
    "ORANGE-LIME"
    "ORANGE-YELLOW"
    "MULTI"
    "SI-BLUE"
    "CLOSED"
    "YELLOW"
    "BLUE-BROWN"
    "GREY-YELLOW"
    "BLUE-LIME"
    "GREY"
    "AIR-BLUE"
    "GREEN-ORANGE"
    ```
    
    There are many dash(-) connected color, which means that the different color route across each other at the same station. In order to get all the distinct colors, I first split them by dash(-), select distinct colors and store it into a new table.
    
    Execute:
    
    ```sql
    DROP TABLE IF EXISTS color_table;
    CREATE TABLE color_table AS
    	SELECT DISTINCT unnest(string_to_array(color, '-')) AS colors
    	FROM nyc_subway_stations;
    ```
    
    This will create a table contains all the distinct colors.
    
    Show all the colors we have:
    
    ```sql
    SELECT colors
    FROM color_table;
    ```
    
    ```sql
    "SI"
    "LIME"
    "PURPLE"
    "GREY"
    "BLUE"
    "YELLOW"
    "RED"
    "GREEN"
    "MULTI"
    "CLOSED"
    "BROWN"
    "ORANGE"
    "AIR"
    ```
    
    Then we execute below query to get the finall result:
    
    ```sql
    SELECT 
    sum(popn_total) AS population, ct_colors
    FROM
    (
    	SELECT DISTINCT c.blkid, c.popn_total, ct_colors
    	FROM
    	(
    		SELECT subway.name, subway.geom AS subway_geom, ct.colors AS ct_colors
    		FROM nyc_subway_stations AS subway
    		JOIN color_table AS ct
    		ON strpos(subway.color, ct.colors) > 0
    	) joined_table1
    	JOIN nyc_census_blocks AS c
    	ON ST_DWithin(c.geom, subway_geom, 200)
    ) joined_table2
    GROUP BY ct_colors
    ORDER BY population DESC
    ```
    
    | population | ct_colors |
    | --- | --- |
    | 608833 | RED |
    | 598605 | GREEN |
    | 523019 | ORANGE |
    | 374963 | BLUE |
    | 296476 | YELLOW |
    | 241894 | BROWN |
    | 121511 | GREY |
    | 102210 | PURPLE |
    | 70421 | LIME |
    | 65394 | MULTI |
    | 26626 | SI |
    | 4823 | CLOSED |
    | 2658 | AIR |
    
    That’s break it down:
    
    The `joined_table1` part is used to select subway stations which have a given color acrossed. The given color comes from above color table, and we join this table into subway station table based on the condition that subway’s color contains color_table’s color.
    
    ```sql
    SELECT subway.name, subway.geom AS subway_geom, ct.colors AS ct_colors
    FROM nyc_subway_stations AS subway
    JOIN color_table AS ct
    ON strpos(subway.color, ct.colors) > 0
    ```
    
    After we get the `joined_table1`, we have subway geom and the corresponding color. Then we select census block based on the condition that the distance between census block and subway station are within 200 meters. Also, there may have some census block belongs to different stations within this distance in the same color route. To avoid recounting, we distinct results on census blkid and colors, so that for each color, we never have a same block in the same color route, while the same block can appear in different color routes. After doing this we get `joined_table2`.
    
    ```sql
    (
    	SELECT DISTINCT c.blkid, c.popn_total, ct_colors
    	FROM
    	(
    		......
    	) joined_table1
    	JOIN nyc_census_blocks AS c
    	ON ST_DWithin(c.geom, subway_geom, 200)
    ) joined_table2
    
    ```
    
    Finally, we select population, group it by each color, than sum over the color to obtain the population living near the color route.
    
    ```sql
    SELECT 
    sum(popn_total), ct_colors
    FROM
    (
    ......
    ) joined_table2
    GROUP BY ct_colors
    ```
    
    > Reference:
    > 
    > 1. How to Split a String in PostgreSQL | [LearnSQL.com](http://learnsql.com/)[https://learnsql.com/cookbook/how-to-split-a-string-in-postgresql/](https://learnsql.com/cookbook/how-to-split-a-string-in-postgresql/)
    > 2. PostgreSQL SELECT DISTINCT By Practical Examples
    > [https://www.postgresqltutorial.com/postgresql-tutorial/postgresql-select-distinct/](https://www.postgresqltutorial.com/postgresql-tutorial/postgresql-select-distinct/)
    > 3. sql - Postgres - CREATE TABLE FROM SELECT - Stack Overflow
    > [https://stackoverflow.com/questions/22953450/postgres-create-table-from-select](https://stackoverflow.com/questions/22953450/postgres-create-table-from-select)
    > 4. Introduction to PostGIS — Introduction to PostGIS
    > [https://postgis.net/workshops/postgis-intro/index.html](https://postgis.net/workshops/postgis-intro/index.html)
    
2. Show the name of the top 3 neighborhoods with the most population density. You also need to show the number of total population, the size (in km2), and the population density (in people/km2) of the neighborhood.
    
    
    ```sql
    SELECT n.name, n.boroname, 
    sum(c.popn_total) AS population,
    ST_Area(n.geom) / 1000000 AS area,
    sum(c.popn_total)/(ST_Area(n.geom) / 1000000) AS density
    FROM nyc_census_blocks AS c
    JOIN nyc_neighborhoods AS n
    ON ST_Intersects(n.geom, c.geom)
    GROUP BY n.name, n.boroname, n.geom
    ORDER BY density DESC
    limit 3
    ```
    
    Results:
    
    | name | boroname | population | area | density |
    | --- | --- | --- | --- | --- |
    | North Sutton Area | Manhattan | 22460 | 0.328194 | 68435.13 |
    | East Village | Manhattan | 82266 | 1.632117 | 50404.48 |
    | Chinatown | Manhattan | 16209 | 0.33198 | 48825.18 |
    
3. Following the previous question, you may notice that the system spent much time executing your queries. Find a way to speed up the query executions and explain your solutions. You need to show the difference of the execution time. (Hint: Use command EXPLAIN ANALYZE can show the query plan and the execution time.)
    
    
    1. For question a., the original query is:
        
        ```sql
        EXPLAIN ANALYZE
        SELECT 
        sum(popn_total), ct_colors
        FROM
        (
        	SELECT DISTINCT c.blkid, c.popn_total, ct_colors
        	FROM
        	(
        		SELECT subway.name, subway.geom AS subway_geom, ct.colors AS ct_colors
        		FROM nyc_subway_stations AS subway
        		JOIN color_table AS ct
        		ON strpos(subway.color, ct.colors) > 0
        	) joined_table1
        	JOIN nyc_census_blocks AS c
        	ON ST_DWithin(c.geom, subway_geom, 200)
        ) joined_table2
        GROUP BY ct_colors
        ORDER BY population DESC
        ```
        
        Analyze result:
        
        [https://explain.tensor.ru/archive/explain/418b4284951b77827fc34175ec73a426:0:2022-11-15#explain](https://explain.tensor.ru/archive/explain/418b4284951b77827fc34175ec73a426:0:2022-11-15#explain)
        
        ![Untitled](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Untitled.png)
        
        ```sql
        QUERY PLAN
        GroupAggregate  (cost=312039671.02..312097864.02 rows=200 width=40) (actual time=9831.605..9834.653 rows=13 loops=1)
          Group Key: joined_table2.ct_colors
          ->  Sort  (cost=312039671.02..312059068.02 rows=7758800 width=40) (actual time=9831.574..9832.422 rows=10381 loops=1)
                Sort Key: joined_table2.ct_colors
                Sort Method: quicksort  Memory: 952kB
                ->  Subquery Scan on joined_table2  (cost=310572289.14..310727465.14 rows=7758800 width=40) (actual time=9813.038..9819.324 rows=10381 loops=1)
                      ->  Unique  (cost=310572289.14..310649877.14 rows=7758800 width=56) (actual time=9813.035..9817.461 rows=10381 loops=1)
                            ->  Sort  (cost=310572289.14..310591686.14 rows=7758800 width=56) (actual time=9813.032..9814.281 rows=10381 loops=1)
                                  Sort Key: c.blkid, c.popn_total, ct.colors
                                  Sort Method: quicksort  Memory: 1114kB
                                  ->  Gather  (cost=305865266.41..309154005.27 rows=7758800 width=56) (actual time=9747.608..9750.707 rows=10381 loops=1)
                                        Workers Planned: 1
                                        Workers Launched: 1
                                        ->  HashAggregate  (cost=305864266.41..308377125.27 rows=7758800 width=56) (actual time=9723.752..9724.984 rows=5191 loops=2)
                                              Group Key: c.blkid, c.popn_total, ct.colors
                                              Planned Partitions: 256  Batches: 1  Memory Usage: 1297kB
                                              Worker 0:  Batches: 1  Memory Usage: 1297kB
                                              ->  Nested Loop  (cost=0.00..292694321.60 rows=124685868 width=56) (actual time=9.446..9714.070 rows=5387 loops=2)
                                                    Join Filter: (strpos((subway.color)::text, ct.colors) > 0)
                                                    Rows Removed by Join Filter: 50565
                                                    ->  Nested Loop  (cost=0.00..280592473.60 rows=275042 width=31) (actual time=8.941..9678.114 rows=4304 loops=2)
                                                          Join Filter: st_dwithin(c.geom, subway.geom, '200'::double precision)
                                                          Rows Removed by Join Filter: 9519623
                                                          ->  Parallel Seq Scan on nyc_census_blocks c  (cost=0.00..1861.20 rows=22820 width=267) (actual time=0.005..5.632 rows=19397 loops=2)
                                                          ->  Seq Scan on nyc_subway_stations subway  (cost=0.00..15.91 rows=491 width=39) (actual time=0.001..0.049 rows=491 loops=38794)
                                                    ->  Seq Scan on color_table ct  (cost=0.00..23.60 rows=1360 width=32) (actual time=0.001..0.002 rows=13 loops=8608)
        Planning Time: 0.400 ms
        Execution Time: 9835.496 ms
        ```
        
        We can find the most time comsuming part is `st_dwithin`. So, let’s try to speedup by creating an index of subway stations using `ST_Buffer` with 200 meters radius:
        
        ```sql
        create index on nyc_subway_stations using gist(ST_Buffer(geom, 200));
        ```
        
        Then instead of using original `ST_DWithin(c.geom, subway_geom, 200)`, we use `ST_Intersects`  `ST_Intersects(c.geom, ST_Buffer(subway_geom, 200))`
        
        ```sql
        EXPLAIN ANALYZE
        SELECT 
        sum(popn_total) AS population, ct_colors
        FROM
        (
        	SELECT DISTINCT c.blkid, c.popn_total, ct_colors
        	FROM
        	(
        		SELECT subway.name, subway.geom AS subway_geom, ct.colors AS ct_colors
        		FROM nyc_subway_stations AS subway
        		JOIN color_table AS ct
        		ON strpos(subway.color, ct.colors) > 0
        	) joined_table1
        	JOIN nyc_census_blocks AS c
        	ON ST_Intersects(c.geom, ST_Buffer(subway_geom, 200))
        ) joined_table2
        GROUP BY ct_colors
        ORDER BY population DESC
        ```
        
        Analyze result:
        
        [https://explain.tensor.ru/archive/explain/5dbbf691ea3e7f7b4e91e1ee94061fb3:0:2022-11-15](https://explain.tensor.ru/archive/explain/5dbbf691ea3e7f7b4e91e1ee94061fb3:0:2022-11-15)
        
        ![Untitled](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Untitled%201.png)
        
        ```sql
        QUERY PLAN
        Sort  (cost=2877758.96..2877759.46 rows=200 width=40) (actual time=403.083..404.156 rows=13 loops=1)
          Sort Key: (sum(c.popn_total)) DESC
          Sort Method: quicksort  Memory: 25kB
          ->  HashAggregate  (cost=2877749.32..2877751.32 rows=200 width=40) (actual time=403.062..404.138 rows=13 loops=1)
                Group Key: ct.colors
                Batches: 1  Memory Usage: 40kB
                ->  HashAggregate  (cost=2515126.19..2761367.32 rows=7758800 width=56) (actual time=399.407..402.394 rows=10359 loops=1)
                      Group Key: c.blkid, c.popn_total, ct.colors
                      Planned Partitions: 256  Batches: 1  Memory Usage: 1809kB
                      ->  Nested Loop  (cost=1000.14..1603050.09 rows=8635040 width=56) (actual time=0.540..391.992 rows=10749 loops=1)
                            Join Filter: (strpos((subway.color)::text, ct.colors) > 0)
                            Rows Removed by Join Filter: 100921
                            ->  Gather  (cost=1000.14..1149680.69 rows=19048 width=31) (actual time=0.515..367.363 rows=8590 loops=1)
                                  Workers Planned: 1
                                  Workers Launched: 1
                                  ->  Nested Loop  (cost=0.14..1146775.89 rows=11205 width=31) (actual time=0.454..356.931 rows=4295 loops=2)
                                        ->  Parallel Seq Scan on nyc_census_blocks c  (cost=0.00..1861.20 rows=22820 width=267) (actual time=0.006..2.921 rows=19397 loops=2)
                                        ->  Index Scan using nyc_subway_stations_st_buffer_idx on nyc_subway_stations subway  (cost=0.14..50.16 rows=1 width=39) (actual time=0.016..0.018 rows=0 loops=38794)
                                              Index Cond: (st_buffer(geom, '200'::double precision, ''::text) && c.geom)
                                              Filter: st_intersects(c.geom, st_buffer(geom, '200'::double precision, ''::text))
                                              Rows Removed by Filter: 0
                            ->  Materialize  (cost=0.00..30.40 rows=1360 width=32) (actual time=0.000..0.001 rows=13 loops=8590)
                                  ->  Seq Scan on color_table ct  (cost=0.00..23.60 rows=1360 width=32) (actual time=0.018..0.019 rows=13 loops=1)
        Planning Time: 0.298 ms
        Execution Time: 404.917 ms
        ```
        
        As we can see, the speedup is very significant, from `9835.496 ms` to `404.917 ms`, reducing about 95% of time, very good!
        
        And the query result do not change a lot:
        
        | population | ct_colors |
        | --- | --- |
        | 607784 | RED |
        | 596761 | GREEN |
        | 521222 | ORANGE |
        | 374088 | BLUE |
        | 294614 | YELLOW |
        | 241355 | BROWN |
        | 121511 | GREY |
        | 102210 | PURPLE |
        | 70421 | LIME |
        | 65394 | MULTI |
        | 26626 | SI |
        | 4823 | CLOSED |
        | 2658 | AIR |
        
        > Reference:
        > 
        > 1. postgresql - ST_DWithin does not use index with non-literal argument - Stack Overflow
        > [https://stackoverflow.com/questions/40919364/st-dwithin-does-not-use-index-with-non-literal-argument](https://stackoverflow.com/questions/40919364/st-dwithin-does-not-use-index-with-non-literal-argument)
        > 2. ST_Buffer
        > [http://postgis.net/docs/ST_Buffer.html](http://postgis.net/docs/ST_Buffer.html)
        > 3. Extent Expand Buffer Distance: PostGIS - ST_Extent, Expand, ST_Buffer, ST_Distance
        > [https://www.bostongis.com/postgis_extent_expand_buffer_distance.snippet](https://www.bostongis.com/postgis_extent_expand_buffer_distance.snippet)
        > 4. PostgreSQL List Indexes
        > [https://www.postgresqltutorial.com/postgresql-indexes/postgresql-list-indexes/](https://www.postgresqltutorial.com/postgresql-indexes/postgresql-list-indexes/)
        
    2. For question b., the original query is:
        
        ```sql
        EXPLAIN ANALYZE
        SELECT n.name, n.boroname, 
        sum(c.popn_total) AS population,
        ST_Area(n.geom) / 1000000 AS area,
        sum(c.popn_total)/(ST_Area(n.geom) / 1000000) AS density
        FROM nyc_census_blocks AS c
        JOIN nyc_neighborhoods AS n
        ON ST_Intersects(n.geom, c.geom)
        GROUP BY n.name, n.boroname, n.geom
        ORDER BY density DESC
        limit 3
        ```
        
        Anaylyze result:
        
        [https://explain.tensor.ru/archive/explain/2bab341c75598b4964e3518f709e3003:0:2022-11-16](https://explain.tensor.ru/archive/explain/2bab341c75598b4964e3518f709e3003:0:2022-11-16)
        
        ![Untitled](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Untitled%202.png)
        
        ```sql
        QUERY PLAN
        Sort  (cost=74333389.20..74333389.52 rows=129 width=892) (actual time=1697.372..1700.462 rows=129 loops=1)
          Sort Key: ((sum(c.popn_total) / (st_area(n.geom) / '1000000'::double precision))) DESC
          Sort Method: quicksort  Memory: 135kB
          ->  Finalize GroupAggregate  (cost=74328564.89..74333384.68 rows=129 width=892) (actual time=1654.584..1700.202 rows=129 loops=1)
                Group Key: n.name, n.boroname, n.geom
                ->  Gather Merge  (cost=74328564.89..74333348.88 rows=129 width=876) (actual time=1654.460..1699.713 rows=256 loops=1)
                      Workers Planned: 1
                      Workers Launched: 1
                      ->  Partial GroupAggregate  (cost=74327564.88..74332334.36 rows=129 width=876) (actual time=1619.041..1656.655 rows=128 loops=2)
                            Group Key: n.name, n.boroname, n.geom
                            ->  Sort  (cost=74327564.88..74328518.52 rows=381455 width=876) (actual time=1618.949..1646.691 rows=18382 loops=2)
                                  Sort Key: n.name, n.boroname, n.geom
                                  Sort Method: external merge  Disk: 11632kB
                                  Worker 0:  Sort Method: external merge  Disk: 18744kB
                                  ->  Nested Loop  (cost=0.00..73997536.80 rows=381455 width=876) (actual time=0.357..1530.938 rows=18382 loops=2)
                                        Join Filter: st_intersects(n.geom, c.geom)
                                        Rows Removed by Join Filter: 2483831
                                        ->  Parallel Seq Scan on nyc_census_blocks c  (cost=0.00..1861.20 rows=22820 width=251) (actual time=0.006..3.008 rows=19397 loops=2)
                                        ->  Seq Scan on nyc_neighborhoods n  (cost=0.00..16.29 rows=129 width=868) (actual time=0.000..0.013 rows=129 loops=38794)
        Planning Time: 0.565 ms
        Execution Time: 1703.292 ms
        ```
        
        As we can see, the most time consumming process is `st_intersects(n.geom, c.geom)`. The reason that it is slow may due to the underlaying geommetry is too compilcated. So, this time, we try to use `ST_Subdivide` to divide census block into parts using rectilinear lines only contains 10 vertices. 
        
        First, we create a new table for the subdivided geoms:
        
        ```sql
        CREATE TABLE subdivided_nyc_census_blocks AS
        	SELECT blkid, ST_Subdivide(geom, 10) AS subdiv_geom
        	FROM nyc_census_blocks;
        ```
        
        Then create index for these subdivided geoms:
        
        ```sql
        create index on subdivided_nyc_census_blocks using gist(subdiv_geom);
        ```
        
        Insted of using original `nyc_census_blocks`, we use `subdivided_nyc_census_blocks` to join with `nyc_neighborhoods`, this makes `ST_Intersects(n.geom, sc.subdiv_geom)` faster:
        
        ```sql
        EXPLAIN ANALYZE
        SELECT table1.name, table1.boroname, 
        sum(c.popn_total) AS population,
        ST_Area(table1.geom) / 1000000 AS area,
        sum(c.popn_total)/(ST_Area(table1.geom) / 1000000) AS density
        FROM
        (
        	SELECT DISTINCT n.name, n.boroname, sc.blkid, n.geom
        	FROM subdivided_nyc_census_blocks AS sc
        	JOIN nyc_neighborhoods AS n
        	ON ST_Intersects(n.geom, sc.subdiv_geom)
        ) table1
        JOIN nyc_census_blocks AS c
        ON table1.blkid = c.blkid
        GROUP BY table1.name, table1.boroname, table1.geom
        ORDER BY density DESC
        limit 3
        ```
        
        Analyze result:
        
        [https://explain.tensor.ru/archive/explain/c00d9999425702b6daab331246b31036:0:2022-11-16](https://explain.tensor.ru/archive/explain/c00d9999425702b6daab331246b31036:0:2022-11-16)
        
        ![Untitled](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Untitled%203.png)
        
        ```sql
        QUERY PLAN
        Limit  (cost=3569194.14..3569194.95 rows=3 width=892) (actual time=1111.754..1111.758 rows=3 loops=1)
          ->  Result  (cost=3569194.14..3610170.42 rows=151764 width=892) (actual time=1111.752..1111.756 rows=3 loops=1)
                ->  Sort  (cost=3569194.14..3569573.55 rows=151764 width=884) (actual time=1111.750..1111.753 rows=3 loops=1)
                      Sort Key: ((sum(c.popn_total) / (st_area(n.geom) / '1000000'::double precision))) DESC
                      Sort Method: top-N heapsort  Memory: 31kB
                      ->  HashAggregate  (cost=3211036.67..3567232.62 rows=151764 width=884) (actual time=1111.579..1111.707 rows=129 loops=1)
                            Group Key: n.name, n.boroname, n.geom
                            Planned Partitions: 32  Batches: 1  Memory Usage: 465kB
                            ->  Hash Join  (cost=1436577.68..1825711.34 rows=1517645 width=876) (actual time=904.817..1052.579 rows=36764 loops=1)
                                  Hash Cond: ((sc.blkid)::text = (c.blkid)::text)
                                  ->  HashAggregate  (cost=1434071.81..1787161.41 rows=1517645 width=884) (actual time=895.081..1019.786 rows=36764 loops=1)
                                        Group Key: n.name, n.boroname, sc.blkid, n.geom
                                        Planned Partitions: 256  Batches: 257  Memory Usage: 8401kB  Disk Usage: 111624kB
                                        ->  Nested Loop  (cost=0.28..36889.89 rows=1517645 width=884) (actual time=0.122..542.855 rows=84138 loops=1)
                                              ->  Seq Scan on nyc_neighborhoods n  (cost=0.00..16.29 rows=129 width=868) (actual time=0.015..0.174 rows=129 loops=1)
                                              ->  Index Scan using subdivided_nyc_census_blocks_subdiv_geom_idx on subdivided_nyc_census_blocks sc  (cost=0.28..285.74 rows=10 width=170) (actual time=0.213..4.116 rows=652 loops=129)
                                                    Index Cond: (subdiv_geom && n.geom)
                                                    Filter: st_intersects(n.geom, subdiv_geom)
                                                    Rows Removed by Filter: 550
                                  ->  Hash  (cost=2020.94..2020.94 rows=38794 width=24) (actual time=9.534..9.534 rows=38794 loops=1)
                                        Buckets: 65536  Batches: 1  Memory Usage: 2634kB
                                        ->  Seq Scan on nyc_census_blocks c  (cost=0.00..2020.94 rows=38794 width=24) (actual time=0.011..5.137 rows=38794 loops=1)
        Planning Time: 0.643 ms
        Execution Time: 1125.600 ms
        ```
        
        As we can see, the execution time from `1703.292 ms` to `1125.600 ms`, reducing about 34% of time, not bad!
        
        Futhermore, we can also apply this subdivide technique to geoms of the `nyc_neighborhoods` table:
        
        ```sql
        CREATE TABLE subdivided_nyc_neighborhoods AS
        	SELECT n.gid, n.name, n.boroname, ST_Subdivide(ST_buffer(geom, 0), 10) AS subdiv_geom
        	FROM nyc_neighborhoods AS n;
        ```
        
        notice that, it well producing error:
        
        > ERROR:  lwgeom_intersection_prec: GEOS Error: TopologyException: Input geom 0 is invalid: Self-intersection at 596394.58472969243 4500899.0716296434
        SQL state: XX000
        > 
        
        if we don’t use `ST_buffer(geom, 0)`. see refs.
        
        Create index:
        
        ```sql
        create index on subdivided_nyc_neighborhoods using gist(subdiv_geom);
        ```
        
        Instead of using `nyc_neighborhoods`, use `subdiv_geom` from  `subdivided_nyc_neighborhoods` to do the `ST_Intersects(sn.subdiv_geom, sc.subdiv_geom)` :
        
        ```sql
        EXPLAIN ANALYZE
        SELECT table1.name, table1.boroname, 
        sum(c.popn_total) AS population,
        ST_Area(n.geom) / 1000000 AS area,
        sum(c.popn_total)/(ST_Area(n.geom) / 1000000) AS density
        FROM
        (
        	SELECT DISTINCT sn.name, sn.boroname, sc.blkid
        	FROM subdivided_nyc_census_blocks AS sc
        	JOIN subdivided_nyc_neighborhoods AS sn
        	ON ST_Intersects(sn.subdiv_geom, sc.subdiv_geom)
        ) table1
        JOIN nyc_census_blocks AS c
        ON table1.blkid = c.blkid
        JOIN nyc_neighborhoods AS n
        ON table1.name = n.name
        GROUP BY table1.name, table1.boroname, n.geom
        ORDER BY density DESC
        limit 3
        ```
        
        Analyze result:
        
        [https://explain.tensor.ru/archive/explain/536e1d3a72c77b4501112a8edbc28cc5:0:2022-11-16](https://explain.tensor.ru/archive/explain/536e1d3a72c77b4501112a8edbc28cc5:0:2022-11-16)
        
        ![Untitled](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Untitled%204.png)
        
        ```sql
        QUERY PLAN
        Limit  (cost=1389719.70..1389720.51 rows=3 width=893) (actual time=773.057..773.071 rows=3 loops=1)
          ->  Result  (cost=1389719.70..1586584.80 rows=729130 width=893) (actual time=773.056..773.067 rows=3 loops=1)
                ->  Sort  (cost=1389719.70..1391542.53 rows=729130 width=885) (actual time=773.051..773.060 rows=3 loops=1)
                      Sort Key: ((sum(c.popn_total) / (st_area(n.geom) / '1000000'::double precision))) DESC
                      Sort Method: top-N heapsort  Memory: 27kB
                      ->  GroupAggregate  (cost=1269103.51..1380295.83 rows=729130 width=885) (actual time=704.657..772.954 rows=129 loops=1)
                            Group Key: sn.name, sn.boroname, n.geom
                            ->  Sort  (cost=1269103.51..1270926.33 rows=729130 width=877) (actual time=704.224..757.540 rows=36764 loops=1)
                                  Sort Key: sn.name, sn.boroname, n.geom
                                  Sort Method: external merge  Disk: 30368kB
                                  ->  Hash Join  (cost=573046.82..634874.50 rows=729130 width=877) (actual time=533.743..577.040 rows=36764 loops=1)
                                        Hash Cond: ((sc.blkid)::text = (c.blkid)::text)
                                        ->  Hash Join  (cost=570540.96..622343.10 rows=729130 width=885) (actual time=524.431..547.988 rows=36764 loops=1)
                                              Hash Cond: ((sn.name)::text = (n.name)::text)
                                              ->  HashAggregate  (cost=570523.06..599490.43 rows=1130434 width=38) (actual time=524.294..534.402 rows=36764 loops=1)
                                                    Group Key: sn.name, sn.boroname, sc.blkid
                                                    Planned Partitions: 32  Batches: 1  Memory Usage: 4625kB
                                                    ->  Nested Loop  (cost=0.28..468784.00 rows=1130434 width=38) (actual time=0.092..467.809 rows=96185 loops=1)
                                                          ->  Seq Scan on subdivided_nyc_neighborhoods sn  (cost=0.00..66.23 rows=1823 width=176) (actual time=0.005..0.383 rows=1823 loops=1)
                                                          ->  Index Scan using subdivided_nyc_census_blocks_subdiv_geom_idx on subdivided_nyc_census_blocks sc  (cost=0.28..257.01 rows=10 width=170) (actual time=0.041..0.250 rows=53 loops=1823)
                                                                Index Cond: (subdiv_geom && sn.subdiv_geom)
                                                                Filter: st_intersects(sn.subdiv_geom, subdiv_geom)
                                                                Rows Removed by Filter: 18
                                              ->  Hash  (cost=16.29..16.29 rows=129 width=859) (actual time=0.129..0.130 rows=129 loops=1)
                                                    Buckets: 1024  Batches: 1  Memory Usage: 113kB
                                                    ->  Seq Scan on nyc_neighborhoods n  (cost=0.00..16.29 rows=129 width=859) (actual time=0.010..0.029 rows=129 loops=1)
                                        ->  Hash  (cost=2020.94..2020.94 rows=38794 width=24) (actual time=9.110..9.111 rows=38794 loops=1)
                                              Buckets: 65536  Batches: 1  Memory Usage: 2634kB
                                              ->  Seq Scan on nyc_census_blocks c  (cost=0.00..2020.94 rows=38794 width=24) (actual time=0.017..4.551 rows=38794 loops=1)
        Planning Time: 0.519 ms
        Execution Time: 778.694 ms
        ```
        
        Even more faster! From original `1703.292 ms` down to `778.694 ms`, reducing about 54% of time, very good!
        
        And the query result almost the same:
        
        | name | boroname | population | area | density |
        | --- | --- | --- | --- | --- |
        | North Sutton Area | Manhattan | 22460 | 0.328194 | 68435.13 |
        | East Village | Manhattan | 82266 | 1.632117 | 50404.48 |
        | Chinatown | Manhattan | 16209 | 0.33198 | 48825.18 |
        
        > Reference:
        > 
        > 1. ST_Subdivide
        > [https://postgis.net/docs/ST_Subdivide.html](https://postgis.net/docs/ST_Subdivide.html)
        > 2. postgresql - Optimizing an intersection between a single massive multipolygon (WKT) and many features from PostGIS - Geographic Information Systems Stack Exchange
        > [https://gis.stackexchange.com/questions/412469/optimizing-an-intersection-between-a-single-massive-multipolygon-wkt-and-many](https://gis.stackexchange.com/questions/412469/optimizing-an-intersection-between-a-single-massive-multipolygon-wkt-and-many)
        > 3. ST_MakeValid
        > [https://postgis.net/docs/ST_MakeValid.html](https://postgis.net/docs/ST_MakeValid.html)
        > 4. Shapely.Polygon.intersection报错:TopologyException: Input geom 1 is invalid: Ring Self-intersection..._江前云后的博客-CSDN博客
        > [https://blog.csdn.net/songyu0120/article/details/104489282](https://blog.csdn.net/songyu0120/article/details/104489282)
        > 5. ERROR: lwgeom_difference: GEOS Error: TopologyException: Input geom 0 is invalid: Too few points in_大白菜炒鸡蛋的博客-CSDN博客
        > [https://blog.csdn.net/qq_27808209/article/details/124127226](https://blog.csdn.net/qq_27808209/article/details/124127226)
        > 6. postgresql - Postgis ST_Intersects performance - Geographic Information Systems Stack Exchange
        > [https://gis.stackexchange.com/questions/166010/postgis-st-intersects-performance](https://gis.stackexchange.com/questions/166010/postgis-st-intersects-performance)
        > 7. Contains/Covers/Intersects/Within? | by Michael Entin | Medium
        > [https://mentin.medium.com/which-predicate-cb608b470471](https://mentin.medium.com/which-predicate-cb608b470471)
        > 8. How we optimized PostgreSQL queries 100x | by Vadim Markovtsev | Towards Data Science
        > [https://towardsdatascience.com/how-we-optimized-postgresql-queries-100x-ff52555eabe](https://towardsdatascience.com/how-we-optimized-postgresql-queries-100x-ff52555eabe)
    

## B. (70%) Find at least two spatial data sets online and show at least two different spatial relationships between the data sets. (For example, show the subway routes with the most unbalanced racial ratio.) You need to explain
how you prepare the data, and the queries you use to find the relationships. Finally, show your query results on google map using .kml format. (You need to share the link of the map in your homework.)

At the beginning of 2020, when the COVID-19 pandamic just started, everyone chasing to buy a mask. In order to solve the shortage of mask, Taiwan’s government announce that everyone should “booking” to buy their own mask. But the mask is limited and only avaliable on the National Health Insurance (NHI) contracted pharmacies and district public health center, so many people asked: “where are the contracted pharmacy?” “Which pharmacy still have mask selling?”. 

Therefore, many enthusiastic people try to collect the coordinates of all the pharmacies and health centers, put it on the map. So now, we have a very correct spatial dataset of pharmacies and health centers.

Now(2022), the pandemic almost gone, instead of knowing which pharmacy still have masks, I want to know: 

1. Does low/hight death rate area have more/less pharmacies there?
2. Do pharmacies over-served in the Indigenous area?

To answer this question, first I need to download the maskmap dataset from:

[https://github.com/kiang/pharmacies/blob/9acba244cbfd4aee365aef31ff68da69edf85a17/data.csv](https://github.com/kiang/pharmacies/blob/9acba244cbfd4aee365aef31ff68da69edf85a17/data.csv)

Open `QGIS > Layer > Add layer > Add Delimited Text Layer` to load this `data.csv` into layer:

![Snipaste_2022-11-21_16-25-43.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-21_16-25-43.png)

Choosing `TGOS X`, `TGOS Y` as X, Y columns, and CRS `EPSG4326`:

![Snipaste_2022-11-21_16-33-34.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-21_16-33-34.png)

Saving it as shapefile:

![Snipaste_2022-11-21_16-38-50.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-21_16-38-50.png)

Because shape file can not save column name exceed 10 charachters, we use other names: `eid` and `name` to represent `醫事機構代碼` and `醫事機構名稱` :

see: [geotools - Bypassing 10 character limit of field name in shapefiles? - Geographic Information Systems Stack Exchange](https://gis.stackexchange.com/questions/15784/bypassing-10-character-limit-of-field-name-in-shapefiles)

![Snipaste_2022-11-21_18-04-33.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-21_18-04-33.png)

To obtain the population spatial data, I find this site:

[https://tesas.nat.gov.tw/lflt/references.html](https://tesas.nat.gov.tw/lflt/references.html)

![Snipaste_2022-11-21_19-56-50.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-21_19-56-50.png)

It collect a lot of Taiwan grovenment opendata, I select population, then it bing me to this site:

[https://segis.moi.gov.tw/STAT/Web/Platform/QueryInterface/STAT_QueryInterface.aspx?Type=0](https://segis.moi.gov.tw/STAT/Web/Platform/QueryInterface/STAT_QueryInterface.aspx?Type=0)

By typing some query condition, you can get many type of statistical data. I just input query conditions: `109yr, smallest census block, population, population statistics`, then it gave me a lot of download link, these link are population statistics of year 109 for each city, then I choose statistic on June version, because at that time, 2020/06, the covid-19 just beginning.

![Snipaste_2022-11-21_19-42-06.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-21_19-42-06.png)

![Snipaste_2022-11-21_19-41-04.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-21_19-41-04.png)

Download all files and extract them, we will see a lot of `TW-04-XXXX` folders, each one represent a county:

![Snipaste_2022-11-22_19-54-28.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_19-54-28.png)

Enter `TW-04-301000000A-010002_U0200-2020M06-09007` folder, we will see `109年6月連江縣統計區人口統計_最小統計區_SHP.zip`, extract it:

![Snipaste_2022-11-22_19-55-08.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_19-55-08.png)

then we will see extracted file contains `109年6月連江縣統計區人口統計_最小統計區.DBF`, `109年6月連江縣統計區人口統計_最小統計區.SHP`, `109年6月連江縣統計區人口統計_最小統計區.SHX` :

![Snipaste_2022-11-22_19-54-56.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_19-54-56.png)

Because we have 22 countys, we extract all and copy paste them into `all` folder:

![Snipaste_2022-11-22_20-02-46.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-02-46.png)

Open QGIS>layer>add vector layer

![Snipaste_2022-11-22_20-04-16.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-04-16.png)

![Snipaste_2022-11-22_20-05-06.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-05-06.png)

![Snipaste_2022-11-22_20-06-29.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-06-29.png)

![Snipaste_2022-11-22_20-06-59.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-06-59.png)

![Snipaste_2022-11-22_20-07-29.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-07-29.png)

![Snipaste_2022-11-22_20-16-36.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-16-36.png)

![Snipaste_2022-11-22_20-17-09.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-17-09.png)

![Snipaste_2022-11-22_20-18-20.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-18-20.png)

![Snipaste_2022-11-22_20-18-32.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-18-32.png)

![Snipaste_2022-11-22_20-26-02.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-26-02.png)

use: `EPSG:3826 - TWD97 / TM2 zone 121`, see: [https://data.gov.tw/dataset/25128](https://data.gov.tw/dataset/25128)

![Snipaste_2022-11-21_19-04-19.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-21_19-04-19.png)

![Snipaste_2022-11-22_20-20-56.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-20-56.png)

![Snipaste_2022-11-22_20-21-31.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-21-31.png)

![Snipaste_2022-11-22_20-22-11.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-22-11.png)

Export as `109_06_Taiwan_population.shp`:

![Snipaste_2022-11-22_20-23-20.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-23-20.png)

![Snipaste_2022-11-22_20-24-41.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-24-41.png)

After remove unessasary layers, we obtain:

![Snipaste_2022-11-22_20-29-32.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-29-32.png)

Looks good!

Then do the same thing to get the indigenous population:

![Snipaste_2022-11-22_23-00-54.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_23-00-54.png)

![Snipaste_2022-11-22_23-02-10.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_23-02-10.png)

In order to know the death rate, I find the statistics of death reason dataset from:

[https://data.gov.tw/dataset/5965](https://data.gov.tw/dataset/5965)

![Snipaste_2022-11-22_20-41-06.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-41-06.png)

get the `全死因1110822.zip` file, extract it, and see:

![Snipaste_2022-11-22_20-43-36.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-43-36.png)

It contains all the death reson from year 82-110, and there is a `欄位說明.xls` file, open it:

![Snipaste_2022-11-22_20-48-04.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-48-04.png)

![Snipaste_2022-11-22_20-48-14.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-48-14.png)

![Snipaste_2022-11-22_20-48-47.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-48-47.png)

![Snipaste_2022-11-22_20-48-39.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-48-39.png)

![Snipaste_2022-11-22_20-48-53.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-48-53.png)

there are many column code mapping, we need to use them to get a meanful value from `dead110.txt` :

![Snipaste_2022-11-22_20-56-51.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_20-56-51.png)

The most important column is `county`, because the statistics are based on county, but the county use in here are not the same as Taiwan population’s county. In order to be able to join them, We need to do some transformation.

The `county` column in the `dead110.txt` is formed like `COUNTY` + `TOWN` , for example: `台北市松山區` . So, We need to to create a new column `COUNTY2`, which combine two columns `COUNTY` and `TOWN` from the `109_06_Taiwan_population` table. By using `COUNTY2` as pivot key, we can join these two table.

![Snipaste_2022-11-22_21-22-39.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_21-22-39.png)

![Snipaste_2022-11-22_21-23-00.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_21-23-00.png)

Enter expression `to_string("COUNTY") + to_string("TOWN")` to combine `COUNTY` and `TOWN`, then create a new field `COUNTY2` with type `string` :

![Snipaste_2022-11-22_21-23-55.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_21-23-55.png)

We can see the new column `COUNTY2` has added:

![Snipaste_2022-11-22_21-24-28.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_21-24-28.png)

Remember to save:

![Snipaste_2022-11-22_21-24-43.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_21-24-43.png)

Later, I use pandas to join all the related columns into `dead110.txt`, then export it as `dead110_joined.csv`:

```bash
death_data.join(county_code.set_index('county'), on='county')\
          .join(sex_code.set_index('sex'), on='sex')\
          .join(age_code.set_index('age_code'), on='age_code')\
          .join(cause_code.set_index('109年以後cause'), on='cause')
```

![Snipaste_2022-11-22_21-41-09.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-22_21-41-09.png)

Note:

When I use this table to join with `county2_table`, I find that there are missing all the regions in `桃園市`, because the latest dead table still name it `桃園縣`.

![Snipaste_2022-11-24_11-26-15.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-24_11-26-15.png)

![Snipaste_2022-11-24_11-33-39.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-24_11-33-39.png)

Therefore. I need to do some string replacing:

```sql
def replace_words(x):
    if '桃園縣' in x:
        y = list(x)
        y[2] = '市'
        y[5] = '區'
    else:
        y = x
    return ''.join(y)

death_data_joined['100年~鄉鎮市區'] = death_data_joined['100年~鄉鎮市區'].apply(lambda x: replace_words(x))
```

Due to chinese Encoding issue, I can not use `shp2pgsql` command to proporly import shape file, so instead, I use QGIS’s DB manager to import shape file into postgre DB.

First I use `add layer` to add all the csv files I collect: 

maskmap_data:

![Snipaste_2022-11-23_16-32-38.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-23_16-32-38.png)

dead110_joined:

![Snipaste_2022-11-23_16-31-17.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-23_16-31-17.png)

results:

![Snipaste_2022-11-23_16-42-32.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-23_16-42-32.png)

open `psql` , create database `adb_hw` and add postgis extension:

![Snipaste_2022-11-23_17-21-20.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-23_17-21-20.png)

go to QGIS, create new connection `adb_hw`:

![Snipaste_2022-11-23_16-51-38.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-23_16-51-38.png)

![Snipaste_2022-11-23_16-52-28.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-23_16-52-28.png)

go to `DB Manager`:

![Snipaste_2022-11-23_16-50-33.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-23_16-50-33.png)

select `import`:

![Snipaste_2022-11-23_16-56-10.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-23_16-56-10.png)

import all the layers we have:

![Snipaste_2022-11-23_16-58-06.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-23_16-58-06.png)

![Snipaste_2022-11-23_16-59-01.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-23_16-59-01.png)

![Snipaste_2022-11-23_17-30-43.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-23_17-30-43.png)

![Snipaste_2022-11-23_17-31-29.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-23_17-31-29.png)

![Snipaste_2022-11-23_17-34-00.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-23_17-34-00.png)

go to `pgAdmin` and choose `dead110_joined` table, you can see chinese characters are display correctly:

![Snipaste_2022-11-23_17-47-30.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-23_17-47-30.png)

Create a table `county2_table` , which joined total population and indigenous population table, and group by `county2` column to obtain each district’s population. Also, I aggregate `geom` by using `ST_Union()`, and in order to do the `ST_Contains()` between county2 and maskmap’s geom, we need to transform it into the same projection system, by applying `ST_Transform()` with SRID `EPSG:4326`, because maskmap’s geom using `EPSG:4326`.

```sql
DROP TABLE IF EXISTS county2_table;
CREATE TABLE county2_table AS
SELECT county2, sum(p_cnt) AS p_tot, sum(o_cnt) AS o_tot,
sum(o_cnt)/sum(p_cnt) AS indigenous_ratio,
ST_Transform(ST_Union(geom), 4326) AS union_geom
FROM
(
	SELECT tp.codebase, tp.village, tp.town, tp.county, 
	tp.county2, tp.p_cnt, tip.o_cnt, tp.geom
	FROM "109_06_Taiwan_population" AS tp
	JOIN "109_06_Taiwan_indigenous_population" AS tip
	ON tp.codebase = tip.codebase
) joined_table1
GROUP BY joined_table1.county2
ORDER BY indigenous_ratio DESC
```

Create index of union geom:

```sql
create index on county2_table using gist(union_geom);
```

Create a table `maskmap2_table`, which contains pharmacy’s name and geom:

```sql
DROP TABLE IF EXISTS maskmap2_table;
CREATE TABLE maskmap2_table AS
SELECT md_e.eid, md_e.name, md.地址 AS Addr, md.電話 AS Tel, md_e.geom 
FROM maskmap_data AS md
JOIN "maskmap_data(醫事機構代碼_eid)" AS md_e
ON md.醫事機構代碼 = md_e.eid
```

Create index of geom:

```sql
create index on maskmap2_table using gist(geom);
```

Now, we can ask: find all districts which have pharmeracies with top 20 most average served people. Because I think the more average people served per pharmeracy, the less medical quality it provides.

```sql
SELECT county2, p_tot, o_tot, indigenous_ratio, 
count(*) AS pharmacy_cnt,
p_tot/count(*) AS p_serv,
ST_Collect(mt.geom) AS loc_geom
FROM county2_table AS ct
JOIN maskmap2_table AS mt
ON ST_Contains(ct.union_geom, mt.geom)
GROUP BY county2, p_tot, o_tot, indigenous_ratio, union_geom
ORDER BY p_serv DESC
limit 20
```

Before I see the result, I guess that the less medical quality region probably the high indigenous density region. But to my surprise, most of regions in this table are non-indigenous region, only `花蓮縣秀林鄉` has high indigenous ratio. It means that, we should not only care about indigenous region, but also need to care about the remote area. 

Note:

`p_tot`: population, `o_tot`: indigenous population, `p_serv`: number person served per pharmacy

| county2 | p_tot | o_tot | indigenous_ratio | pharmacy_cnt | p_serv |
| --- | --- | --- | --- | --- | --- |
| 金門縣金寧鄉 | 31798 | 286 | 0.008994276 | 1 | 31798 |
| 南投縣鹿谷鄉 | 17346 | 56 | 0.00322841 | 1 | 17346 |
| 花蓮縣秀林鄉 | 15884 | 14052 | 0.884663813 | 1 | 15884 |
| 高雄市永安區 | 13728 | 76 | 0.005536131 | 1 | 13728 |
| 金門縣烈嶼鄉 | 12687 | 48 | 0.0037834 | 1 | 12687 |
| 新竹縣橫山鄉 | 12673 | 300 | 0.023672374 | 1 | 12673 |
| 苗栗縣造橋鄉 | 12326 | 110 | 0.008924225 | 1 | 12326 |
| 新北市貢寮區 | 12264 | 97 | 0.007909328 | 1 | 12264 |
| 宜蘭縣壯圍鄉 | 24300 | 215 | 0.008847737 | 2 | 12150 |
| 嘉義縣東石鄉 | 23984 | 65 | 0.00271014 | 2 | 11992 |
| 嘉義縣六腳鄉 | 21964 | 49 | 0.002230923 | 2 | 10982 |
| 宜蘭縣三星鄉 | 21026 | 564 | 0.026823932 | 2 | 10513 |
| 金門縣金沙鎮 | 20692 | 171 | 0.008264063 | 2 | 10346 |
| 臺南市安定區 | 30478 | 132 | 0.004330993 | 3 | 10159.33333 |
| 金門縣金湖鎮 | 30208 | 395 | 0.013076006 | 3 | 10069.33333 |
| 花蓮縣富里鄉 | 10020 | 1712 | 0.170858283 | 1 | 10020 |
| 新竹市香山區 | 78760 | 1189 | 0.015096496 | 8 | 9845 |
| 苗栗縣南庄鄉 | 9708 | 2080 | 0.214256283 | 1 | 9708 |
| 桃園市新屋區 | 47267 | 805 | 0.01703091 | 5 | 9453.4 |
| 高雄市鳥松區 | 46449 | 294 | 0.006329523 | 5 | 9289.8 |

Create a table `dead2_table`, which contains the joined information of `county2_table` and `dead110_joined`. Because I only want to know the death rate which related to the disease, I select `cause < 39`, which means exclude accident, murder, suicide and others cases, and select `age_code < 24`, which means exclude older than 90 years old cases. 

```sql
DROP TABLE IF EXISTS dead2_table;
CREATE TABLE dead2_table AS
SELECT ct.county2, ct.p_tot, ct.o_tot, ct.indigenous_ratio, 
selected_table.dead_cnt, selected_table.dead_cnt/ct.p_tot AS death_rate,
ct.union_geom
FROM 
(
	SELECT "100年~鄉鎮市區" AS county2, sum("N") AS dead_cnt
	FROM dead110_joined
	WHERE cause < 39 and age_code < 22
	GROUP BY "100年~鄉鎮市區"
)selected_table
JOIN county2_table AS ct
ON selected_table.county2 = ct.county2
ORDER BY death_rate DESC
```

Create index:

```sql
create index on dead2_table using gist(union_geom);
```

Then we can ask: Does the number of pharmacy affects the health?  By conbining `dead2_table` and `maskmap2_table`, we can list top 20 highest and lowest death rate region, and see if it correlated to the number of pharmacy?

```sql
SELECT county2, p_tot, indigenous_ratio, death_rate,
count(*) AS pharmercy_cnt, ST_Collect(mt.geom) AS loc_geom,
p_tot/count(*) AS p_serv
FROM dead2_table AS dt
JOIN maskmap2_table AS mt
ON ST_Contains(dt.union_geom, mt.geom)
GROUP BY county2, p_tot, indigenous_ratio, death_rate
ORDER BY death_rate DESC
limit 20
```

Top 20 hightest death rate region:

| county2 | p_tot | indigenous_ratio | death_rate | pharmacy_cnt | p_serv |
| --- | --- | --- | --- | --- | --- |
| 屏東縣霧臺鄉 | 2273 | 0.83457985 | 0.01099868 | 1 | 2273 |
| 屏東縣瑪家鄉 | 4840 | 0.965909091 | 0.010330579 | 1 | 4840 |
| 新竹縣五峰鄉 | 4470 | 0.908277405 | 0.010067114 | 1 | 4470 |
| 屏東縣獅子鄉 | 4896 | 0.947099673 | 0.009803922 | 1 | 4896 |
| 花蓮縣卓溪鄉 | 6046 | 0.955011578 | 0.009427721 | 1 | 6046 |
| 臺東縣海端鄉 | 3822 | 0.916274202 | 0.009419152 | 1 | 3822 |
| 新北市平溪區 | 4524 | 0.00464191 | 0.009062776 | 1 | 4524 |
| 屏東縣春日鄉 | 3797 | 0.959441664 | 0.008954438 | 2 | 1898.5 |
| 臺東縣長濱鄉 | 6703 | 0.595554229 | 0.008802029 | 2 | 3351.5 |
| 高雄市甲仙區 | 5914 | 0.019783564 | 0.008792695 | 5 | 1182.8 |
| 高雄市桃源區 | 4134 | 0.918964683 | 0.008466376 | 1 | 4134 |
| 新竹縣尖石鄉 | 9604 | 0.87817576 | 0.008329863 | 2 | 4802 |
| 臺東縣延平鄉 | 3503 | 0.923208678 | 0.008278618 | 1 | 3503 |
| 高雄市茂林區 | 1935 | 0.941602067 | 0.008268734 | 1 | 1935 |
| 花蓮縣玉里鎮 | 23532 | 0.323559408 | 0.008244093 | 8 | 2941.5 |
| 屏東縣枋山鄉 | 5260 | 0.032889734 | 0.008174905 | 2 | 2630 |
| 屏東縣泰武鄉 | 4533 | 0.971542025 | 0.008162365 | 1 | 4533 |
| 臺南市左鎮區 | 4712 | 0.004032258 | 0.007852292 | 2 | 2356 |
| 嘉義縣阿里山鄉 | 5489 | 0.638732009 | 0.00783385 | 2 | 2744.5 |
| 屏東縣滿州鄉 | 7410 | 0.274089069 | 0.007692308 | 1 | 7410 |

```sql
SELECT county2, p_tot, indigenous_ratio, death_rate,
count(*) AS pharmercy_cnt, ST_Collect(mt.geom) AS loc_geom,
p_tot/count(*) AS p_serv
FROM dead2_table AS dt
JOIN maskmap2_table AS mt
ON ST_Contains(dt.union_geom, mt.geom)
GROUP BY county2, p_tot, indigenous_ratio, death_rate
ORDER BY death_rate
limit 20
```

Top 20 lowest death rate region:

| county2 | p_tot | indigenous_ratio | death_rate | pharmacy_cnt | p_serv |
| --- | --- | --- | --- | --- | --- |
| 新竹縣竹北市 | 197408 | 0.006271276 | 0.001651402 | 56 | 3525.142857 |
| 金門縣金寧鄉 | 31798 | 0.008994276 | 0.002107051 | 1 | 31798 |
| 連江縣東引鄉 | 1353 | 0.022172949 | 0.002217295 | 1 | 1353 |
| 新北市林口區 | 118640 | 0.019310519 | 0.002233648 | 32 | 3707.5 |
| 臺北市內湖區 | 284135 | 0.008671934 | 0.00225597 | 58 | 4898.87931 |
| 臺中市南屯區 | 175216 | 0.007727605 | 0.002282897 | 62 | 2826.064516 |
| 連江縣北竿鄉 | 2602 | 0.019984627 | 0.002305919 | 1 | 2602 |
| 新竹市東區 | 218731 | 0.007278346 | 0.002313344 | 62 | 3527.919355 |
| 金門縣金湖鎮 | 30208 | 0.013076006 | 0.002317267 | 3 | 10069.33333 |
| 臺北市大安區 | 305328 | 0.003759891 | 0.002345019 | 89 | 3430.651685 |
| 金門縣金城鎮 | 43045 | 0.005296782 | 0.002369613 | 8 | 5380.625 |
| 臺北市松山區 | 202264 | 0.004711664 | 0.0024028 | 64 | 3160.375 |
| 臺中市西屯區 | 230808 | 0.008457246 | 0.002465253 | 85 | 2715.388235 |
| 桃園市蘆竹區 | 167260 | 0.029564749 | 0.002499103 | 59 | 2834.915254 |
| 臺北市中正區 | 155607 | 0.0046656 | 0.00251274 | 56 | 2778.696429 |
| 臺中市南區 | 126727 | 0.008096144 | 0.002540895 | 36 | 3520.194444 |
| 金門縣金沙鎮 | 20692 | 0.008264063 | 0.002561376 | 2 | 10346 |
| 連江縣南竿鄉 | 7589 | 0.015548821 | 0.002635393 | 1 | 7589 |
| 臺北市文山區 | 270126 | 0.008248003 | 0.002687635 | 58 | 4657.344828 |
| 桃園市桃園區 | 455064 | 0.018085368 | 0.002716101 | 144 | 3160.166667 |

The most interesting part is, most of highest death rate region is the indigenous region, and doesn’t have many pharmacy, but the average person served per pharmacy doesn’t seem to be high; in the contrary, most of lowest death rate region have many pharmacies, and the average person served per pharmacy is higher. 

Export tables and convert it to the KML file:

Top_20_highest_people_served_pharmacy_table:

```sql
DROP TABLE IF EXISTS Top_20_highest_people_served_pharmacy_table;
CREATE TABLE Top_20_highest_people_served_pharmacy_table AS
SELECT county2, p_tot, o_tot, indigenous_ratio, p_serv, 
mt.name, mt.Addr, mt.Tel, mt.geom
FROM
(
	SELECT county2, p_tot, o_tot, indigenous_ratio, 
	count(*) AS pharmacy_cnt,
	p_tot/count(*) AS p_serv,
	ST_Collect(mt.geom) AS loc_geom
	FROM county2_table AS ct
	JOIN maskmap2_table AS mt
	ON ST_Contains(ct.union_geom, mt.geom)
	GROUP BY county2, p_tot, o_tot, indigenous_ratio
	ORDER BY p_serv DESC
	limit 20
)table1
JOIN maskmap2_table AS mt
ON ST_Contains(table1.loc_geom, mt.geom)
```

Top_20_highest_death_rate_pharmarcy:

```sql
DROP TABLE IF EXISTS Top_20_highest_death_rate_pharmarcy;
CREATE TABLE Top_20_highest_death_rate_pharmarcy AS
SELECT county2, p_tot, indigenous_ratio, death_rate, p_serv, 
mt.name, mt.Addr, mt.Tel, mt.geom
FROM
(
	SELECT county2, p_tot, indigenous_ratio, death_rate,
	ST_Collect(mt.geom) AS loc_geom,
	p_tot/count(*) AS p_serv
	FROM dead2_table AS dt
	JOIN maskmap2_table AS mt
	ON ST_Contains(dt.union_geom, mt.geom)
	GROUP BY county2, p_tot, indigenous_ratio, death_rate
	ORDER BY death_rate DESC
	limit 20
)table1
JOIN maskmap2_table AS mt
ON ST_Contains(table1.loc_geom, mt.geom)
```

Top_20_lowest_death_rate_pharmarcy:

```sql
DROP TABLE IF EXISTS Top_20_lowest_death_rate_pharmarcy;
CREATE TABLE Top_20_lowest_death_rate_pharmarcy AS
SELECT county2, p_tot, indigenous_ratio, death_rate, p_serv, 
mt.name, mt.Addr, mt.Tel, mt.geom
FROM
(
	SELECT county2, p_tot, indigenous_ratio, death_rate,
	ST_Collect(mt.geom) AS loc_geom,
	p_tot/count(*) AS p_serv
	FROM dead2_table AS dt
	JOIN maskmap2_table AS mt
	ON ST_Contains(dt.union_geom, mt.geom)
	GROUP BY county2, p_tot, indigenous_ratio, death_rate
	ORDER BY death_rate
	limit 20
)table1
JOIN maskmap2_table AS mt
ON ST_Contains(table1.loc_geom, mt.geom)
```

QGIS view:

![Snipaste_2022-11-24_12-54-51.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-24_12-54-51.png)

Export to KML:

![Snipaste_2022-11-24_13-14-31.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-24_13-14-31.png)

Upload to the Google Map:

[https://www.google.com/maps/d/edit?mid=1QYh6eQSLoTS5rif0jzLa--qeIPN5trE&usp=sharing](https://www.google.com/maps/d/edit?mid=1QYh6eQSLoTS5rif0jzLa--qeIPN5trE&usp=sharing)

![Snipaste_2022-11-24_13-12-40.png](PostGIS%20Lab%20Homework%20-%20110065801%20-%20%E6%9D%8E%E4%BA%9E%E5%80%AB%208b743f675d5047ef82fe5f2e94d0de3e/Snipaste_2022-11-24_13-12-40.png)

> Reference:
> 
> 1. Projecting Data — Introduction to PostGIS
> [https://postgis.net/workshops/postgis-intro/projection.html#projecting-data](https://postgis.net/workshops/postgis-intro/projection.html#projecting-data)
> 2. WGS 84 - WGS84 - World Geodetic System 1984, used in GPS - EPSG:4326
> [https://epsg.io/4326](https://epsg.io/4326)
> 3. TWD97 / TM2 zone 121 - EPSG:3826
> [https://epsg.io/3826](https://epsg.io/3826)
> 4. 全國門牌地址定位服務
> [https://www.tgos.tw/TGOS/Web/Address/TGOS_Address.aspx](https://www.tgos.tw/TGOS/Web/Address/TGOS_Address.aspx)
> 5. ST_Union
> [https://postgis.net/docs/ST_Union.html](https://postgis.net/docs/ST_Union.html)
> 6. ST_Collect
> [https://postgis.net/docs/ST_Collect.html](https://postgis.net/docs/ST_Collect.html)
> 7. postgresql - Import CSV with many columns to pgAdmin v4.1 - Stack Overflow
> [https://stackoverflow.com/questions/54321733/import-csv-with-many-columns-to-pgadmin-v4-1](https://stackoverflow.com/questions/54321733/import-csv-with-many-columns-to-pgadmin-v4-1)
> 8. TESAS | 地方創生資料庫
> [https://tesas.nat.gov.tw/lflt/references.html](https://tesas.nat.gov.tw/lflt/references.html)