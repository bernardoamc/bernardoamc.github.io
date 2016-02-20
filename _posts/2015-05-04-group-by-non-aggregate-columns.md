---
layout:     post
title:      Non aggregate columns and GROUP BY
date:       2015-05-04 22:00:00
summary:    This is a question answered from time to time but that still confuses
            some people. Let's understand why it's not so easy to select non
            aggregate columns after a GROUP BY and what are our alternatives.
categories: sql
---

The following question is not new, but keeps being repeated over time.

>"How do we select non-aggregate columns in a query with a `GROUP BY` clause?"

In this post we will investigate this question and try to answer it in a didatic
way, so we can refer to this post in the future. The code examples are from
postgres, but should work in any other relational database.

Let's create and populate our table:

~~~ sql
CREATE TABLE games (
  game_id serial PRIMARY KEY,
  name VARCHAR,
  price BIGINT,
  released_at DATE,
  publisher TEXT
);

INSERT INTO games (name, price, released_at, publisher) VALUES
  ('Metal Slug Defense', 30, '2015-05-01', 'SNK Playmore'),
  ('Project Druid', 20, '2015-05-01', 'shortcircuit'),
  ('Chroma Squad', 40, '2015-04-30', 'Behold Studios'),
  ('Soul Locus', 30, '2015-04-30', 'Fat Loot Games'),
  ('Subterrain', 40, '2015-04-30', 'Pixellore');

SELECT * FROM games;

 game_id |        name        | price | released_at |   publisher
---------+--------------------+-------+-------------+----------------
       1 | Metal Slug Defense |    30 | 2015-05-01  | SNK Playmore
       2 | Project Druid      |    20 | 2015-05-01  | shortcircuit
       3 | Chroma Squad       |    40 | 2015-04-30  | Behold Studios
       4 | Soul Locus         |    30 | 2015-04-30  | Fat Loot Games
       5 | Subterrain         |    40 | 2015-04-30  | Pixellore
(5 rows)
~~~

We want something like:

~~~ sql
SELECT released_at, name, publisher, MAX(price) as most_expensive
FROM games
GROUP BY released_at;
~~~

This means that we want to group games by release date and after this get
the most expensive one and its name  and publisher for each date.
What we get instead is:

~~~ sql
ERROR:  column "games.name" must appear in the GROUP BY
clause or be used in an aggregate function
LINE 1: SELECT released_at, name, publisher, MAX(price) ...
                            ^
~~~

### Why is that?

To understand why this happens, first, we must know that after we group
our games by release date our database will have a pool of rows (by date)
and can't infer which row to choose from if we want some column
other than the grouped one. So, in the case of the date `2015-05-01` it has to
choose from the pool:

~~~ sql
 game_id |        name        | price | released_at |   publisher
--------+--------------------+-------+-------------+----------------
      1 | Metal Slug Defense |    30 | 2015-05-01  | SNK Playmore
      2 | Project Druid      |    20 | 2015-05-01  | shortcircuit
~~~

It can't know if we mean `Metal Slug Defense` or `Project Druid` for the
game name and `SNK Playmore` or `shortcircuit` for publisher.

But what about `MAX(price)`? It's obvious that we are talking about
`Metal Slug Defense` and `SNK Playmore` since we are asking for the most
expensive game of this date, no?

No! Databases don't work this way, when you ask for the higher price it will
inspect each price from the pool and return it, but it will not select the entire
row after returning the price. This assumption is what brings the question in the
first place. Let's make this clear:

>Selecting the MAX(price) does not select the entire row.

But why not? It's obvious! In this case it is, but what about the date
`2015-04-30`? Our pool for this date is:

~~~ sql
 game_id |        name        | price | released_at |   publisher
--------+--------------------+-------+-------------+----------------
      3 | Chroma Squad       |    40 | 2015-04-30  | Behold Studios
      4 | Soul Locus         |    30 | 2015-04-30  | Fat Loot Games
      5 | Subterrain         |    40 | 2015-04-30  | Pixellore
~~~

Here the higher price is `40`, but we have two games with this price, `Chroma
Squad` and `Subterrain`, which one should we choose from to set the name and
publisher? The database can't know and when it can't give the right answer every
time for a given query it should give us an error, and that's what it does!

Ok... Ok... It's not so simple, what can we do?

### Approach to solve this problem

**1)** Create a query that contains the fields that we want in our answer:

~~~ sql
SELECT g1.name, g1.publisher, g1.price, g1.released_at
FROM games AS g1;

        name        |   publisher    | price | released_at
--------------------+----------------+-------+-------------
 Metal Slug Defense | SNK Playmore   |    30 | 2015-05-01
 Project Druid      | shortcircuit   |    20 | 2015-05-01
 Chroma Squad       | Behold Studios |    40 | 2015-04-30
 Soul Locus         | Fat Loot Games |    30 | 2015-04-30
 Subterrain         | Pixellore      |    40 | 2015-04-30
(5 rows)
~~~

**2)** Create a query that calculates the higher price per release date:

~~~ sql
SELECT released_at, MAX(price) as price
FROM games
GROUP BY released_at;

 released_at | price
-------------+-------
 2015-04-30  |    40
 2015-05-01  |    30
(2 rows)
~~~

**3)** Joins both queries making **2** a temporary table:

~~~ sql
SELECT g1.name, g1.publisher, g1.price, g1.released_at
FROM games AS g1
INNER JOIN (
  SELECT released_at, MAX(price) as price
  FROM games
  GROUP BY released_at
) AS g2
ON g2.released_at = g1.released_at AND g2.price = g1.price;

       name        |   publisher    | price | released_at
--------------------+----------------+-------+-------------
 Metal Slug Defense | SNK Playmore   |    30 | 2015-05-01
 Chroma Squad       | Behold Studios |    40 | 2015-04-30
 Subterrain         | Pixellore      |    40 | 2015-04-30
(3 rows)
~~~

We just found rows in table **g1** that have the criteria specified by the rows
in table **g2**. With this we are guaranteed to find the right answer, even if
there is more than one game with the same price in the same date.

### Alternative

We can use a `LEFT OUTER JOIN` to solve this problem, but it is a bit tricky.
Let's see part of the query and try to understand it:

~~~ sql
SELECT g1.name, g1.publisher, g1.price, g2.price, g1.released_at
FROM games AS g1
LEFT OUTER JOIN games AS g2
ON g1.released_at = g2.released_at AND g1.price < g2.price;

        name        |   publisher    | price | price  | released_at
--------------------+----------------+-------+--------+-------------
 Metal Slug Defense | SNK Playmore   |    30 | (null) | 2015-05-01
 Project Druid      | shortcircuit   |    20 |     30 | 2015-05-01
 Chroma Squad       | Behold Studios |    40 | (null) | 2015-04-30
 Soul Locus         | Fat Loot Games |    30 |     40 | 2015-04-30
 Soul Locus         | Fat Loot Games |    30 |     40 | 2015-04-30
 Subterrain         | Pixellore      |    40 | (null) | 2015-04-30
(6 rows)
~~~

We are fetching rows from table `g1` that have the same date as rows
in table `g2` and that have a lower price. Since the rows in `g1` with
the higher price doesn't have a corresponding row in `g2`, the field `g2.price`
will be `NULL`.

We can see that the rows with a `NULL` value for `g2.price` is the rows that
we want, so we just need to use this filter to obtain the expected rows:

~~~ sql
SELECT g1.name, g1.publisher, g1.price, g2.price, g1.released_at
FROM games AS g1
LEFT OUTER JOIN games AS g2
ON g1.released_at = g2.released_at AND g1.price < g2.price
WHERE g2.price IS NULL;

        name        |   publisher    | price | price  | released_at
--------------------+----------------+-------+--------+-------------
 Metal Slug Defense | SNK Playmore   |    30 | (null) | 2015-05-01
 Chroma Squad       | Behold Studios |    40 | (null) | 2015-04-30
 Subterrain         | Pixellore      |    40 | (null) | 2015-04-30
(3 rows)
~~~

And that's it!

Do you know another way to solve this problem? Leave me a
tweet or email if you know, I would appreciate!

See you in the next post!
