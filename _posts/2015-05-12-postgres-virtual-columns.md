---
layout:     post
title:      Virtual columns in Postgres
date:       2015-05-11 15:00:00
summary:    This post is about virtual columns in Postgres (or the lack of them)
            and a small trick I recently learned that can make our queries
            easier to read, which is always something to strive for.
categories: sql
---

It's probably a good idea to start saying that Postgres doesn't have the
concept of virtual columns (or generated columns) as other databases does,
for example, [MySQL](http://mysqlserverteam.com/generated-columns-in-mysql-5-7-5/).
With this in mind, let's see what we can do to at least simulate this
functionality.

To start the discussion, let's first setup our example and explain the
situation:

{% highlight sql %}
CREATE TABLE users (
  user_id serial PRIMARY KEY,
  name varchar,
  registration varchar
);

INSERT INTO users(name, registration) VALUES
  ('john', '1234-5'),
  ('alice', '12-345'),
  ('bob', '36872'),
  ('jean', '551-34');

SELECT * FROM users;

 user_id | name  | registration
---------+-------+--------------
       1 | john  | 1234-5
       2 | alice | 12-345
       3 | bob   | 36872
       4 | jean  | 551-34
(4 rows)
{% endhighlight %}

As we can see we have *users* in our database and each user have a registration
number. Suppose this is a legacy system and some users were created with
duplicated registration numbers, the only difference is the *dash* between
numbers, that in this case shouldn't make a difference.

So, *john* and *alice* have the same registration number `12345` and we should
inform this in our query. We can start with a JOIN to solve our problem:

{% highlight sql %}
SELECT u1.name, u1.registration, u2.name, u2.registration
FROM users AS u1 INNER JOIN users AS u2
ON replace(u1.registration, '-', '') = replace(u2.registration,'-', '')
WHERE u1.user_id <> u2.user_id;

name  | registration | name  | registration
-------+--------------+-------+--------------
 john  | 1234-5       | alice | 12-345
 alice | 12-345       | john  | 1234-5
(2 rows)
{% endhighlight %}

Or we could use a `GROUP BY` and the `string_agg` function:

{% highlight sql %}
SELECT
  replace(registration, '-', '') as registration,
  string_agg(name, ',') as names
FROM users
GROUP BY replace(registration, '-', '')
HAVING count(*) > 1;

registration |   names
--------------+------------
 12345        | john,alice
(1 row)
{% endhighlight %}

Nothing wrong with these queries, but the need to call `replace` repeatedly is
someting that bothers me, can we avoid this? The first thing that I can think of
is to [create a view](http://www.postgresql.org/docs/9.4/static/sql-createview.html)
with our `replace` function as a virtual column:

{% highlight sql %}
CREATE OR REPLACE VIEW normalized_users AS
SELECT user_id, name, replace(registration, '-', '') as registration
FROM users;

SELECT
  registration,
  string_agg(name, ',') as names
FROM normalized_users
GROUP BY registration
HAVING count(*) > 1;

 registration |   names
--------------+------------
 12345        | john,alice
(1 row)
{% endhighlight %}

Great, no more functions to remember! This is a great way to simplify our data
model, but, do we have alternatives to solve this?

### Function as a column

This alternative require us to create a function that will receive our row
as a paramater and normalize its registration number, after that we can call
this function like it was a column.

It's easier to explain this by showing the result:

{% highlight sql %}
CREATE OR REPLACE FUNCTION normalized_registration(users)
RETURNS text AS $$
  SELECT replace($1.registration, '-', '')
$$ STABLE LANGUAGE SQL;

SELECT
  users.normalized_registration,
  string_agg(name, ',') as names
FROM users
GROUP BY users.normalized_registration
HAVING count(*) > 1;

 registration |   names
--------------+------------
 12345        | john,alice
(1 row)
{% endhighlight %}

We created the `normalized_registration` function and called it using the syntax
`users.normalized_registration`, how is this possible?

The trick here is that **attribute notation** (*users.name*) and **function
notation** (*name(users)*) are equivalent in Postgres! Let's see this in use:

{% highlight sql %}
SELECT name(users), users.name
FROM users;

 name  | name
-------+-------
 john  | john
 alice | alice
 bob   | bob
 jean  | jean
(4 rows)
{% endhighlight %}

With this we can call our functions like virtual columns! Another cool thing is
that since our function is `STABLE` we can create an index using it!

{% highlight sql %}
CREATE INDEX normalized_registration_idx
ON users (normalized_registration(users));
{% endhighlight %}

And if we analyze our query we can see that it's using the index:

{% highlight sql %}
EXPLAIN SELECT
  users.normalized_registration,
  string_agg(name, ',') as names
FROM users
GROUP BY users.normalized_registration
HAVING count(*) > 1;

                          QUERY PLAN
------------------------------------------------------------------
 GroupAggregate  (cost=0.13..12.30 rows=4 width=64)
   Group Key: replace((registration)::text, '-'::text, ''::text)
   Filter: (count(*) > 1)
   ->  Index Scan using normalized_registration_idx on users
       (cost=0.13..12.20 rows=4 width=64)
(4 rows)
{% endhighlight %}

If the index is not being used in your example it's probably because the
sample is small and the database performs better with a sequential scan. In this
case if we want to be sure that the index is going to be used we can disable the
sequencial scan and run our query again:

{% highlight sql %}
SET enable_seqscan = OFF;
-- Run our query with EXPLAIN
SET enable_seqscan = ON;
{% endhighlight %}

The last thing that is useful to mention is that if the virtual column requires
some heavy computation it is probably better to create a real column and
populate it using a
[trigger](http://www.postgresql.org/docs/9.4/static/plpgsql-trigger.html) since
it's something that's only going to be invoked during `INSERT` and `UPDATE`.

### TL;DR:

We can achieve something similar as a virtual column using:

- Views
- Functions

But its better to use
[triggers](http://www.postgresql.org/docs/9.4/static/plpgsql-trigger.html)
with a real column in case of heavy computations.

If there is another way of solving this or something bothers you in this post
please leave me a tweet or email, I will appreciate it!

See you in the next post!
