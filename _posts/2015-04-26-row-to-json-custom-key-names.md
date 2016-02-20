---
layout:     post
title:      row_to_json and custom key names
date:       2015-04-26 22:00:00
summary:    Recently I was in need to generate JSON from a query and set
            my key names at will. At first I thought the process would
            be straightforward, but there is a trick behind this. Let's find
            more about it!
categories: postgres
---

Prior to PostgreSQL `9.4` there is no "easy" way to generate `JSON` from our
queries and define custom key names. In this post we will see some solutions
for this problem and how everything changed in version `9.4`.

Suppose we have the following tables and data:

~~~ sql
CREATE TABLE categories (
  category_id serial PRIMARY KEY,
  name varchar
);

CREATE TABLE products (
  product_id serial PRIMARY KEY,
  category_id BIGINT,
  name varchar,
  FOREIGN KEY (category_id) REFERENCES categories(category_id)
);

CREATE TABLE orders (
  order_id serial PRIMARY KEY,
  product_id BIGINT,
  name varchar,
  FOREIGN KEY (product_id) REFERENCES products(product_id)
);

INSERT INTO categories (name)
  VALUES ('books'), ('electronics');

INSERT INTO products (category_id, name)
  VALUES (1, 'Musashi'), (2, 'Microwave');

INSERT INTO orders (product_id, name)
  VALUES (1, 'Musashi Order'), (2,'Microwave Order');
~~~

Now we want to fetch the name and category of each product associated
with an `order` and return it as `JSON`. The keys will be named `product_name`
and `category_name`. How do we do something like this?

At first I thought a simple query like this would suffice to get the data I
wanted:

~~~ sql
SELECT row_to_json(p.name AS product_name, c.name AS category_name)
FROM orders
JOIN products As p USING (product_id)
JOIN categories AS c USING(category_id);
~~~

Turns out `row_to_json` expects a `record` type, not a bunch of `character varying`
elements. This made me think that I could get my result combining `row_to_json`
and `row` functions:

~~~ sql
SELECT row_to_json(
  ROW(p.name AS product_name, c.name AS category_name)
)
FROM orders
JOIN products As p USING (product_id)
JOIN categories AS c USING(category_id);

ERROR:  syntax error at or near "AS"
LINE 1: SELECT row_to_json(row(p.name AS product_name, c.name AS cat...
~~~

This only works if I do not specify a custom name for my fields:

~~~ sql
SELECT row_to_json(
  ROW(p.name, c.name)
)
FROM orders
JOIN products As p USING (product_id)
JOIN categories AS c USING(category_id);

              row_to_json
---------------------------------------
 {"f1":"Musashi","f2":"books"}
 {"f1":"Microwave","f2":"electronics"}
(2 rows)
~~~

Still not what I want, where's my custom key names?! After some deliberation I
came up with a subquery, this would let me pass to `row_to_json` what it
expects:

~~~ sql
SELECT row_to_json(r)
FROM (
  SELECT p.name as product_name, c.name as category_name
  FROM orders
  JOIN products As p USING (product_id)
  JOIN categories AS c USING(category_id)
) as r;

                        row_to_json
------------------------------------------------------------
 {"product_name":"Musashi","category_name":"books"}
 {"product_name":"Microwave","category_name":"electronics"}
(2 rows)
~~~

And it works, but I'm not a fan of subqueries, could we reach our goal
without using it? What about custom types?

~~~ sql
CREATE TYPE product_json AS
  (product_name varchar, category_name varchar);

SELECT row_to_json((p.name, c.name)::product_json)
FROM orders
JOIN products As p USING (product_id)
JOIN categories AS c USING(category_id);

                        row_to_json
------------------------------------------------------------
 {"product_name":"Musashi","category_name":"books"}
 {"product_name":"Microwave","category_name":"electronics"}
(2 rows)
~~~

Nice! This is way easier to read in my opinion and the result is the same, I
would stop here, but is there another way? `CTEs` would probably also work:

~~~ sql
WITH products_json AS (
  SELECT p.name AS product_name, c.name AS category_name
  FROM orders
  JOIN products As p USING (product_id)
  JOIN categories AS c USING(category_id)
)
SELECT row_to_json(products_json) FROM products_json;

                        row_to_json
------------------------------------------------------------
 {"product_name":"Musashi","category_name":"books"}
 {"product_name":"Microwave","category_name":"electronics"}
(2 rows)
~~~

And it does! I still prefer using `custom types`, but `CTEs` are also not bad.

### Postgres 9.4

As always newer versions of postgres brings us some improvements and after
version `9.4` we could just use the `json_build_object` function.

~~~ sql
SELECT json_build_object(
  'product_name',  p.name,
  'category_name', c.name
)
FROM orders
JOIN products As p USING (product_id)
JOIN categories AS c USING(category_id);

                        row_to_json
------------------------------------------------------------
 {"product_name":"Musashi","category_name":"books"}
 {"product_name":"Microwave","category_name":"electronics"}
(2 rows)
~~~

Beautiful!

Maybe there are more ways to solve this problem, please leave me a
tweet or email if you know, I would appreciate!

See you in the next post!
