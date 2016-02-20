---
layout:     post
title:      Removing duplicates in Postgres
date:       2015-08-10 13:00:00
summary:    This is a study to find ways in which we can remove duplicate rows
            based on a certain criteria from our tables.
categories: sql
---

The best way to avoid duplicates in our database is to use a `UNIQUE` constraint
or index in the first place, but as we know sometimes things get out of control
and some housekeeping is needed. Let's see some ways to do this:

Suppose we have a table called orders like the following:

~~~ sql
CREATE TABLE orders(
  id serial PRIMARY KEY,
  item_id integer,
  code varchar
);

INSERT INTO orders(item_id, code)
VALUES (1, 'abc'),
       (1, 'abc'),
       (2, 'dba'),
       (2, 'hjk'),
       (2, 'dba'),
       (3, 'fkl');

SELECT * FROM orders;

 id | item_id | code
----+---------+------
  1 |       1 | abc
  2 |       1 | abc
  3 |       2 | dba
  4 |       2 | hjk
  5 |       2 | dba
  6 |       3 | fkl
(6 rows)
~~~

We want to remove rows that have the same `item_id` and `code`, what are our
options? Actually, a lot!

*If your table have an unique identifier like an `id` you can skip the next
section.*

### Tables without an unique identifier

If your table don't have an unique identifier you can replace the `id` column by
the `ctid` column in the examples below.

`ctid` is a [system column](http://www.postgresql.org/docs/9.4/static/ddl-system-columns.html)
representing the physical location of the row in our table. This
identifier is guaranteed to be unique, but changes over time if our rows are
updated, so it is only useful for these kind of operations where you won't need
to use its values later.

With this covered, let's start our study:

### GROUP BY
Let's start simple and avoid "fancy" things like `joins` and `window functions`.

~~~ sql
DELETE
FROM orders
WHERE id NOT in (
  SELECT MIN(id)
  FROM orders
  GROUP BY item_id, code
);
~~~

Here we group our rows using the same criteria we use to find duplicates and
fetch the lowest id in each group. After this we delete every row that is not
in this set.

We could achieve the same result with `DISTINCT ON`, let's see an example.

### DISTINCT ON

~~~ sql
DELETE
FROM orders
WHERE id NOT IN (
  SELECT DISTINCT ON (item_id, code) id
  FROM orders
);
~~~

Here we fetch the first row from each group with the same `item_id` and
`name` and delete the rest. It's important to mention that we are relying in the
order by `id`. If this is not the case we should explicitly order our result as
mentioned [in the
documentation](http://www.postgresql.org/docs/9.0/static/sql-select.html#SQL-DISTINCT).

### EXISTS

~~~ sql
DELETE
FROM orders o1
WHERE EXISTS (
  SELECT 1
  FROM orders o2
  WHERE o1.item_id = o2.item_id AND
        o1.code = o2.code AND
        o1.id > o2.id
);
~~~

This reads as: "Delete every row from o1 that has the same item_id and code and
an id bigger than at least one id from o2". This will ensure that only the
duplicates with the lowest id remains.

### USING

~~~ sql
DELETE
FROM orders o1 USING orders o2
WHERE o1.item_id = o2.item_id AND
      o1.code = o2.code AND
      o1.id > o2.id;
~~~

Which executes the same code in our `GROUP BY` example. We also use the same
technique explained above to keep the lowest id from our duplicated records.
We can see another `USING` example
[here](http://www.postgresql.org/docs/9.4/static/sql-delete.html#AEN78458).

### PARTITION (WINDOW FUNCTION)

~~~ sql
DELETE
FROM orders
WHERE id IN (
  SELECT id
  FROM (
   SELECT row_number() OVER (PARTITION BY item_id, code ORDER BY id),
          id
   FROM orders
  ) AS sorted
  WHERE sorted.row_number > 1
);
~~~

Here the rows were partitioned by the specified criteria to find duplicates
and sorted by `id`, after this we create a column with the row
number for each row in a partition. The last step is to select only the rows
with row numbers greater than 1 to delete.

### Temporary tables

~~~ sql
CREATE TABLE temporary_orders AS
  SELECT DISTINCT ON (item_id, code) * FROM orders;

DROP TABLE orders;
ALTER TABLE temporary_orders RENAME TO orders;
~~~

This solution is fast, but you have to disable indexes and foreign key constraints
in your table to achieve this, which is not very practical. Still, it's a
possibility that we should take into account.

We can also use a `CREATE TEMPORARY TABLE` if our table should be dropped at the
end of the session.

That's it for now, if you know more solutions or have further questions
feel free to leave me a tweet or email so we can discuss things further!

See you in the next post!
