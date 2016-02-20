---
layout:     post
title:      Filter clause in Postgres
date:       2015-04-12 12:00:00
summary:    One of the great things of the Postgres team is that they are always
            trying to improve and simplify our work. This feature is one small
            example of their mentality towards improvement. Let's see how we can
            simplify some queries using this new clause.
categories: postgres
---

This post is a quick tip about using the `FILTER` clause, a feature that was
recently added in version `9.4`, to improve some of our old queries that uses
the `CASE` clause.

### What it does?

The filter clause feeds the aggregation function with rows for which it
evaluates to true. It literally filter rows according to our query.

### Example

Suppose we have two tables in our database like the ones below:

~~~ sql
SELECT * FROM sports;

 id |   name
----+-----------
  1 | football
  2 | jiu jitsu
  3 | parkour
  4 | baseball

SELECT * FROM people;

 id | name  | age | sport_id
----+-------+-----+----------
  1 | john  |  20 |        1
  2 | sam   |  21 |        2
  3 | matos |  22 |        1
  4 | alice |  20 |        1
  5 | jean  |  28 |        1
  6 | bob   |  30 |        4
~~~

Now we want to count the number of people that likes football (`sport_id = 1`) grouped by age.
We could do something like this using the good and old `CASE` clause:

~~~ sql
SELECT age, COUNT(CASE WHEN sport_id = 1 THEN 1 ELSE NULL END) AS f
FROM people
GROUP BY age;

 age | f
-----+---
  28 | 1
  30 | 0
  20 | 2
  22 | 1
  21 | 0
(5 rows)
~~~

Nothing wrong with this approach, but with postgres `9.4` we can use `FILTER`
and simplify things a little.

~~~ sql
SELECT age, COUNT(*) FILTER(WHERE sport_id = 1) AS f
FROM people
GROUP BY age;

 age | f
-----+---
  28 | 1
  30 | 0
  20 | 2
  22 | 1
  21 | 0
(5 rows)
~~~

We are filtering rows that have `sport_id = 1` and passing it to `COUNT` without
worrying about the ones that don't have it. It just reads better in my opinion.

### TL;DR:
Consider using `FILTER` instead of `CASE` to improve the query readability.

See you in the next post!
