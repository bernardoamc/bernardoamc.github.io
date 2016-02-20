---
layout:     post
title:      Lateral joins in Postgres
date:       2015-06-23 23:00:00
summary:    In this post we will work in a common problem and explore how to solve
            it using this not so new feature called LATERAL join.
categories: sql
---

Lateral joins are not so new, but still something not used that frequently
and that I never had the chance do explore. It was added in version
`9.3` of postgres and that is probably the reason why it didn't get so much
hype, since the `JSON` type was also introduced in this release, plus `MATERIALIZED VIEWS`
and some other cool features (what a great release!).

Also, it's valid to point out that this feature is not unique to postgres, SQL
Server has the exactly same feature named `CROSS APPLY` for example.

#### So, what is a `LATERAL join`?

As always we can refer to the
[documentation](http://www.postgresql.org/docs/devel/static/sql-select.html) for
help, but to put it simply what it does is:

>"The subquery specified on the RIGHT SIDE of the JOIN is evaluated for each row
on the LEFT side of the JOIN."

This also means that the subquery can access fields from records on the left
side of the join, which normally would be impossible.

The format is something like:

>SELECT * FROM something [INNER/LEFT] JOIN LATERAL (subquery);

Don't worry if this wasn't clear enough, we will see an example with this in
action and things will make sense.

#### The feature request

*Create a report informing the last two sells of each game. Each
row should have the following attributes:*

* *Name of the game*
* *Price of the sell*
* *Date of the sell*

Investigating the database we see an structure like the following:

~~~ sql
\dt

     List of relations
 Schema |    Name    | Type
--------+------------+-------
 public | games      | table
 public | sold_games | table
(2 rows)

-- List of indexes
\di

                List of relations
 Schema |          Name          | Type  |   Table
--------+------------------------+-------+------------
 public | games_pkey             | index | games
 public | sold_games_game_id_idx | index | sold_games
 public | sold_games_pkey        | index | sold_games
(3 rows)
~~~

With the current data:

~~~ sql
SELECT * FROM games;

 game_id |        name
---------+--------------------
       1 | Metal Slug Defense
       2 | Project Druid
       3 | Chroma Squad
       4 | 7 days to die
(4 rows)

SELECT game_id, price, sold_on FROM sold_games;

 game_id | price |  sold_on
---------+-------+------------
       1 |    30 | 2015-05-01
       1 |    45 | 2015-05-13
       1 |    35 | 2015-05-20
       2 |    20 | 2015-04-01
       2 |    12 | 2015-04-09
       3 |    40 | 2015-06-17
       3 |    55 | 2015-06-20
       3 |    40 | 2015-06-30
       4 |    50 | 2015-04-12
(9 rows)
~~~

#### Solution without `LATERAL`

Let's create a solution without `LATERAL` first so we can see the
benefits of using `LATERAL` over this implementation later.

Having decided this, let's fetch the last two sells of each game without
bothering with the game name for now:

~~~ sql
SELECT * FROM (
  SELECT row_number() OVER (
          PARTITION BY game_id ORDER BY sold_on DESC
         ) as sell_pos,
         game_id, price, sold_on
  FROM sold_games
) AS s
WHERE s.sell_pos < 3;

 sell_pos | game_id | price |  sold_on
----------+---------+-------+------------
        1 |       1 |    35 | 2015-05-20
        2 |       1 |    45 | 2015-05-13
        1 |       2 |    12 | 2015-04-09
        2 |       2 |    20 | 2015-04-01
        1 |       3 |    40 | 2015-06-30
        2 |       3 |    55 | 2015-06-20
        1 |       4 |    50 | 2015-04-12
(7 rows)
~~~

It works, now we can fetch the names using an `INNER JOIN`:

~~~ sql
SELECT g.name, ls.price, ls.sold_on
FROM (
  SELECT * FROM
  (
    SELECT row_number() OVER (
            PARTITION BY game_id ORDER BY sold_on DESC
           ) as sell_pos,
           game_id, price, sold_on
    FROM sold_games
  ) AS s
  WHERE s.sell_pos < 3
) AS ls
INNER JOIN games AS g
USING(game_id);

        name        | price |  sold_on
--------------------+-------+------------
 Metal Slug Defense |    35 | 2015-05-20
 Metal Slug Defense |    45 | 2015-05-13
 Project Druid      |    12 | 2015-04-09
 Project Druid      |    20 | 2015-04-01
 Chroma Squad       |    40 | 2015-06-30
 Chroma Squad       |    55 | 2015-06-20
 7 days to die      |    50 | 2015-04-12
(7 rows)
~~~

And we have our answer!

Running the current implementation with `EXPLAIN ANALYZE` we obtain:

~~~ sql
                                QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------
 Hash Join  (cost=150.94..211.38 rows=543 width=44) (actual time=0.036..0.048 rows=7 loops=1)
   Hash Cond: (s.game_id = g.game_id)
   ->  Subquery Scan on s  (cost=113.27..166.24 rows=543 width=16) (actual time=0.019..0.030 rows=7 loops=1)
         Filter: (s.sell_pos < 3)
         Rows Removed by Filter: 2
         ->  WindowAgg  (cost=113.27..145.87 rows=1630 width=16) (actual time=0.017..0.027 rows=9 loops=1)
               ->  Sort  (cost=113.27..117.34 rows=1630 width=16) (actual time=0.013..0.015 rows=9 loops=1)
                     Sort Key: sold_games.game_id, sold_games.sold_on
                     Sort Method: quicksort  Memory: 25kB
                     ->  Seq Scan on sold_games  (cost=0.00..26.30 rows=1630 width=16) (actual time=0.002..0.004 rows=9 loops=1)
   ->  Hash  (cost=22.30..22.30 rows=1230 width=36) (actual time=0.012..0.012 rows=4 loops=1)
         Buckets: 1024  Batches: 1  Memory Usage: 1kB
         ->  Seq Scan on games g  (cost=0.00..22.30 rows=1230 width=36) (actual time=0.006..0.009 rows=4 loops=1)
 Planning time: 0.166 ms
 Execution time: 0.088 ms
(15 rows)
~~~

#### Solution with `LATERAL`

We can count this as a win if we improve our legibility without losing
performance OR if we can improve both of these aspects.

The final solution is something like:

~~~ sql
SELECT g.name, ls.price, ls.sold_on
FROM games AS g
INNER JOIN LATERAL (
  SELECT game_id, price, sold_on
  FROM sold_games AS s
  WHERE g.game_id = s.game_id
  ORDER BY s.sold_on DESC
  LIMIT 2
) AS ls
ON TRUE;
~~~

Definitely easier to read, and running `EXPLAIN ANALYZE` we obtain:

~~~ sql
                                                       QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------
 Nested Loop  (cost=1.12..1433.72 rows=1230 width=44) (actual time=0.024..0.049 rows=7 loops=1)
   ->  Seq Scan on games g  (cost=0.00..22.30 rows=1230 width=36) (actual time=0.004..0.005 rows=4 loops=1)
   ->  Limit  (cost=1.12..1.13 rows=1 width=16) (actual time=0.009..0.010 rows=2 loops=4)
         ->  Sort  (cost=1.12..1.13 rows=1 width=16) (actual time=0.009..0.009 rows=2 loops=4)
               Sort Key: s.sold_on
               Sort Method: quicksort  Memory: 25kB
               ->  Seq Scan on sold_games s  (cost=0.00..1.11 rows=1 width=16) (actual time=0.002..0.004 rows=2 loops=4)
                     Filter: (g.game_id = game_id)
                     Rows Removed by Filter: 7
 Planning time: 0.277 ms
 Execution time: 0.081 ms
(11 rows)
~~~

Which is also an improvement, even with this small subset.

#### What is really going on

1. We fetch rows from the games table.
2. **For each** row we run the subquery.
3. And join the result with the respective **row**.

#### Improved benchmark

Let's expand our example a bit and see how our database performs with something
like 100 games and 100_000 sells.

~~~ sql
INSERT INTO games (name) (select generate_series(1,100));

SELECT count(*) FROM games;

 count
-------
   100
(1 row)

INSERT INTO sold_games (game_id, price, sold_on)
SELECT (1 + random() * 100) AS game_id,
       price,
       (current_date + random() * 100 * interval '1 day') as sold_on
FROM generate_series(1,100000) AS s(price);

SELECT count(*) FROM sold_games;

 count
--------
 100000
(1 row)
~~~

Running `EXPLAIN ANALYZE` for the first query we get the time:

~~~ sql
Planning time: 0.375 ms
Execution time: 165.094 ms
~~~

And now with `LATERAL`:

~~~ sql
Planning time: 0.132 ms
Execution time: 62.724 ms
~~~

Huge win!

That's it for now, `LATERAL` was a huge eye opener for me and
I hope this post had a similar effect for you too!

See you in the next post!
