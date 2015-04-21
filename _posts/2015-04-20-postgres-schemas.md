---
layout:     post
title:      Schemas in Postgres
date:       2015-04-20 20:00:00
summary:    Even after using Postgres for some time I was not aware of schemas
            and how it could be used to organize objects into logical groups.
            In this post we will see what is a schema and how we can use it to
            improve our database management.
categories: postgres
---

If you never heard of schemas and have used postgres for a long time you should be
asking yourself if schemas are really important. The short answer is that it
really depends on your use case, but it is always good to learn about available
options, right? So, what is a schema?

A schema is like a namespace where we can define our objects, in the case
of a database we are talking about `tables`, `functions`, `types` and so on. A
database can have multiple schemas and a schema belongs to a single database.
This is nice because we can separate our tables into logical groups or allow
users to interact only with the data that they should in a database.

The truth is: everybody that uses postgres is actually using schemas.

### How it works?

If we don't specify a schema, postgres will use the default one named `public`.
This is the default namespace where our tables are being created this entire
time.

Let's see this in action by creating a table named `categories` and querying it
without a namespace first.

{% highlight sql %}
CREATE TABLE categories (
  id serial PRIMARY KEY,
  name varchar
);

INSERT INTO categories (name) VALUES ('books'), ('electronics');

SELECT * FROM categories;

id |    name
----+-------------
  1 | books
  2 | electronics
(2 rows)
{% endhighlight %}

We got the result that we expected. Let's run the same query specifying
the default schema this time:

{% highlight sql %}
SELECT * FROM public.categories;

 id |    name
----+-------------
  1 | books
  2 | electronics
(2 rows)
{% endhighlight %}

Exactly the same result! This tell us that our table is in the `public` schema
(namespace) and also that we can query it using the notation `namespace.table`.

We can check which tables are in a schema by using the following query:

{% highlight sql %}
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'public';

  table_name
--------------
 categories
(1 row)
{% endhighlight %}

Or if you use the awesome `psql` interface:

{% highlight sql %}
\dt public.*

            List of relations
 Schema |     Name     | Type  |  Owner
--------+--------------+-------+----------
 public | categories   | table | bernardo
(1 row)
{% endhighlight %}

### Creating a schema

So far we saw that if we don't specify a schema our tables will end inside the
`public` schema. How do we create a schema so we can avoid this? Let's see an
example:

{% highlight sql %}
CREATE SCHEMA example;
CREATE SCHEMA
Time: 39.698 ms
{% endhighlight %}

That's it, we created a schema named `example`. To make sure this is true let's
list our available schemas in this database using the `psql` interface:

{% highlight sql %}
\dn

   List of schemas
   Name   |  Owner
----------+----------
 example  | bernardo
 public   | bernardo
(2 rows)
{% endhighlight %}

We have two schemas as expected, the default one and the one we created. We can
add objects to our schema right away.

### Adding objects to a schema

We will start by creating a table with the same name as the one we already have:

{% highlight sql %}
CREATE TABLE categories (
  id serial PRIMARY KEY,
  name varchar
);

ERROR:  relation "categories" already exists
Time: 3.112 ms
{% endhighlight %}

What? Oh, right, we forgot to specify our schema, so postgres tried to create a
table named `categories` in the `public` schema, but we already have a table
with this same name. Let's try again:

{% highlight sql %}
CREATE TABLE example.categories (
  id serial PRIMARY KEY,
  name varchar
);

CREATE TABLE
Time: 235.251 ms

INSERT INTO example.categories (name) VALUES ('toys');
{% endhighlight %}

Nice, we created our new table inside the `example` schema and already inserted
a row in it. Let's see our tables:

{% highlight sql %}
SELECT * FROM example.categories;

 id | name
----+------
  1 | toys
(1 row)

SELECT * FROM public.categories;

 id |    name
----+-------------
  1 | books
  2 | electronics
(2 rows)
{% endhighlight %}

As we can see they are independent tables with its own data in the same database,
this is only possible because they are in different schemas. What if we don't
want our tables in the `public` schema by default?

### The `search_path` variable

This variable acts like the environment variable `PATH` in Linux. We define
schema names in a certain order and when we do some operation it will search
for each schema in the specified order. Let's see the default value of
this variable.

{% highlight sql %}
SHOW search_path;

   search_path
-----------------
 "$user", public
(1 row)
{% endhighlight %}

Shouldn't the `search_path` contain only the `public` schema? And what about this
`"$user"`? This is a postgres variable that contains the current user name, this
means that when we execute an operation, the first schema that is going to be
searched is the one with the same name as the user we are logged in, if it is
not found we go to the `public` schema that is created by default.

That's exactly what was happening since we don't have a schema with the current
user name.

{% highlight sql %}
SELECT user;
 current_user
--------------
 bernardo
(1 row)

\dn

   List of schemas
   Name   |  Owner
----------+----------
 example  | bernardo
 public   | bernardo
(2 rows)
{% endhighlight %}

This is a trick that postgres employs so you don't need to change your
`search_path` every time you want to namespace your tables. You just have to create
a new user and a schema with the same name, after this every action done by this
user will end in the correct schema.

To change the `search_path` variable we just do:

{% highlight sql %}
SET search_path TO example,"$user",public;
{% endhighlight %}

And if we query our categories table without passing a schema we hope to list
the entries of `example.categories`, since it is the first schema that is going
to be searched.

{% highlight sql %}
SELECT * from categories;

 id | name
----+------
  1 | toys
(1 row)
{% endhighlight %}

We get the content of `example.categories` as expected. Every command will be
executed in this schema from now on, so things like queries,
creating new tables, updates, all of these will be executed in the `example`
schema.

Suppose we now want to remove our schema, not a problem:

{% highlight sql %}
-- If our schema is empty
DROP SCHEMA example;

-- If our schema has objects
DROP SCHEMA example CASCADE;
{% endhighlight %}

That's basically it about schemas. Jamie Winsor has a [cool
talk](https://www.youtube.com/watch?v=_i6n-eWiVn4&list=PLWbHc_FXPo2h0sJW6X2RZDtT1ndw6KKpQ&index=34)
about building a MMO game using Elixir and Postgres with schemas.

### TL;DR:

Schemas are in essence just namespaces and are necessary if you want to separate
your database objects into logical groups. They are also used to restrict which
objects an user should see and interact with.

See you in the next post!
