---
layout:     post
title:      Window Functions in Postgres
date:       2015-04-07 20:00:00
summary:    Window functions are supported since version 8.4, but this feature
            is still unknown to some people. In this post we will investigate
            and define what is a window function and how it can be used.
categories: postgres
---

A lot has been said about window functions in this past year, but at least to me
the definition presented in some posts were not clear enough. Let's try to
define it in our own words and examples.

### So, what are they?

Window functions are all about aggregation. As the name implies, they work on a
`window` that contains rows and these rows are **_the output of the query we are
running_**. So, after our records are filtered by `WHERE`, `HAVING` and `GROUP BY`,
the returned records will be used by our window function.

The syntax to use it is: `aggregate_function OVER ()`. We can learn more about
these functions
[here](http://www.postgresql.org/docs/9.4/static/functions-aggregate.html).

### Example

Let's first do the setup for the example:

~~~ sql
CREATE TABLE users (
  id serial PRIMARY KEY,
  name varchar(100),
  age integer,
  salary integer
);

INSERT INTO users (id, name, age, salary) VALUES
  (1, 'a', 23, 1000),
  (2, 'b', 23, 1500),
  (3, 'c', 25, 2500),
  (4, 'd', 25, 3500),
  (5, 'e', 28, 5000);

 id | name | age | salary
----+------+-----+--------
  1 | a    |  23 |   1000
  2 | b    |  23 |   1500
  3 | c    |  25 |   2500
  4 | d    |  25 |   3500
  5 | e    |  28 |   5000
(5 rows)
~~~

Finally, let's use window functions to calculate the average salary per age:

~~~ sql
SELECT name, AVG(salary) OVER (PARTITION BY age) as average_salary
FROM users;

 name |    average_salary
------+-----------------------
 a    | 1250.0000000000000000
 b    | 1250.0000000000000000
 c    | 3000.0000000000000000
 d    | 3000.0000000000000000
 e    | 5000.0000000000000000
(5 rows)
~~~

So, what happened?

We calculated the users average salary by age. Our query does not have a `WHERE`,
so the output will be *all the rows* in our table and that is exactly what will be
used by our window function. The trick here is that we used the `PARTITION BY age`
inside our `OVER()` clause, this tells the `AVG(salary)` function to calculate the
average salary per age.

Without the `PARTITION BY` our *average_salary* would be the same for each row as
we can see below:

~~~ sql
SELECT name, AVG(salary) OVER () as average_salary
FROM users;

 name |    average_salary
------+-----------------------
 a    | 2700.0000000000000000
 b    | 2700.0000000000000000
 c    | 2700.0000000000000000
 d    | 2700.0000000000000000
 e    | 2700.0000000000000000
(5 rows)
~~~

We can also use `ORDER BY` inside our `OVER()` clause, but it is something to
keep an eye on. I will quote the official documentation since it is described
in a great way:

*By default, if ORDER BY is supplied then the frame consists of all rows from
the start of the partition up through the current row, plus any following rows
that are equal to the current row according to the ORDER BY clause.*

So it's not really a surprise that our next example works this way:

~~~ sql
SELECT name, age, salary, SUM(salary) OVER (ORDER BY age) as sum
FROM users;

 name | age | salary |  sum
------+-----+--------+-------
 a    |  23 |   1000 |  2500
 b    |  23 |   1500 |  2500
 c    |  25 |   2500 |  8500
 d    |  25 |   3500 |  8500
 e    |  28 |   5000 | 13500
(5 rows)
~~~

That's it for now, hope the window functions concept is a bit clearer now.

### TL;DR:

Window functions are great to avoid complex joins and consequently speed up our
queries. Thoughtbot has a [nice post](https://robots.thoughtbot.com/postgres-window-functions)
about it and of course there is also the great [official
documentation](http://www.postgresql.org/docs/9.4/static/tutorial-window.html).

See you in the next post!
