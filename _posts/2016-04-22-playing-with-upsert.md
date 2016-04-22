---
layout:     post
title:      Playing with upsert
date:       2016-04-22 13:00:00
summary:    In this post we will explore upsert, a new feature provided by
            PostgreSQL 9.5. What is it and what can we do with it?
categories: sql
---

It is not news that every new version of PostgreSQL provides multiple features.
In this post we will explore Upsert, one of the few major features that was missing
from this amazing database.

In simple terms, upsert is nothing more than an insert with new possibilities in case
of a conflict. It is worth to remember that we can always refer to the amazing
[documentation](http://www.postgresql.org/docs/9.5/static/sql-insert.html) that
is provided when in doubt.

To start our exploration, let's create a simple table with a few records:

~~~ sql
CREATE TABLE games (
  game_id serial PRIMARY KEY,
  name varchar(100) NOT NULL UNIQUE,
  price integer NOT NULL
);

INSERT INTO games(name, price)
VALUES ('Final Fantasy IX', 50),
       ('Tales of Abyss', 35),
       ('Breath of Fire IV', 20);

select * from games;

 game_id |         name          | price
---------+-----------------------+-------
       1 | Final Fantasy IX      |    50
       2 | Tales of Abyss        |    35
       3 | Breath of Fire IV     |    20
(3 rows)
~~~

Let's begin with a traditional insert with a conflict.

~~~ sql
INSERT INTO games(name, price)
VALUES ('Final Fantasy IX', 50);

ERROR:  duplicate key value violates unique constraint "games_name_key"
DETAIL:  Key (name)=(Final Fantasy IX) already exists.
~~~

We got an error as expected since we already have this name in our table.
Now suppose a new business rule specified that conflicts like this should be
ignored, we could `rescue` (or `catch`) an exception in our application and
life goes on, but now we have another option:

~~~ sql
INSERT INTO games(name, price)
VALUES ('Final Fantasy IX', 50)
ON CONFLICT DO NOTHING;
INSERT 0 0
~~~

The `INSERT 0 0` hints us that nothing was inserted, which is exactly what we
wanted. Everyone is happy for a while, but the rules change and now in case
of a conflict the `price` need to be updated! Fear not, is what you say to
your manager:

~~~ sql
INSERT INTO games(name, price)
VALUES ('Final Fantasy IX', 75)
ON CONFLICT (name)
DO UPDATE SET price = EXCLUDED.price;
INSERT 0 1
Time: 1.786 ms

SELECT * FROM games WHERE name = 'Final Fantasy IX';
 game_id |       name       | price
---------+------------------+-------
       1 | Final Fantasy IX |    75
~~~

Notice that we use the `EXCLUDED` keyword to specify the updated price.
In this case we also specified the type of conflict, if a different conflict
occurs it will not be "rescued" as we can see below:

~~~ sql
INSERT INTO games(game_id, name, price)
VALUES (1, 'New Game', 10)
ON CONFLICT (name)
DO UPDATE SET price = EXCLUDED.price;

ERROR:  duplicate key value violates unique constraint "games_pkey"
DETAIL:  Key (game_id)=(1) already exists.
~~~

Suddenly the players start complaining that the games are getting expensive
and by consequence the sales in our store are decreasing, the new rule is
that the price should be updated only if the game gets cheaper.

~~~ sql
INSERT INTO games(name, price)
VALUES ('Final Fantasy IX', 80)
ON CONFLICT (name)
DO UPDATE SET price = EXCLUDED.price
WHERE EXCLUDED.price < games.price;
INSERT 0 0

SELECT * FROM games WHERE name = 'Final Fantasy IX';
 game_id |       name       | price
---------+------------------+-------
       1 | Final Fantasy IX |    75

INSERT INTO games(name, price)
VALUES ('Final Fantasy IX', 60)
ON CONFLICT (name)
DO UPDATE SET price = EXCLUDED.price
WHERE EXCLUDED.price < games.price;
INSERT 0 1

SELECT * FROM games WHERE name = 'Final Fantasy IX';
 game_id |       name       | price
---------+------------------+-------
       1 | Final Fantasy IX |    60
~~~

We can specify the name of our constraint in `ON CONFLICT`, but it is recommended
to use column names and let postgres specify the type of the index for you.
Let's see an example just in case:

~~~ sql
\dS games
                                    Table "public.games"
 Column  |          Type          |                        Modifiers
---------+------------------------+---------------------------------------------------------
 game_id | integer                | not null default
nextval('games_game_id_seq'::regclass)
 name    | character varying(100) | not null
 price   | integer                | not null
Indexes:
    "games_pkey" PRIMARY KEY, btree (game_id)
    "games_name_key" UNIQUE CONSTRAINT, btree (name)

INSERT INTO games(name, price)
VALUES ('Final Fantasy IX', 50)
ON CONFLICT ON CONSTRAINT games_name_key DO NOTHING;
INSERT 0 0
~~~

If we had a composite index or a partial index we could also specify
a condition in our `ON CONFLICT` clause. For example, if we had a composite
index on (name, year), we could do:

~~~ sql
INSERT INTO games(name, price)
VALUES ('Final Fantasy XV', 100)
ON CONFLICT (name, year)
WHERE year = 2016
DO NOTHING;
~~~

That's it for now, if you have further questions or suggestions
feel free to leave me a tweet or email so we can discuss things further!

See you in the next post!
