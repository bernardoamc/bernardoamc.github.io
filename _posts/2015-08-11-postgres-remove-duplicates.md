---
layout:     post
title:      Removing duplicates in Postgres
date:       2015-08-11 23:00:00
summary:    This is a study to find ways in which we can remove duplicate rows
            based on a certain criteria from our tables.
categories: sql
---

The best way to avoid duplicates in our database is to use a `UNIQUE` constraint
or index in the first place, but as we know sometimes things get out of control
and some housekeeping is needed. Let's see some ways to do this:

Suppose we have a table called orders like the following:

{% highlight sql %}
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
{% endhighlight %}

We want to remove rows that have the same `item_id` and `code`, what are our
options?

### GROUP BY
Let's start simple and avoid fancy things like `joins`, `window functions` and
so on.

{% highlight sql %}
DELETE
FROM orders
WHERE id NOT in (
  SELECT MIN(id)
  FROM orders
  GROUP BY item_id, code
);
{% endhighlight %}

Here we group our rows using the same criteria we use to find duplicates and
fetch the lowest id in each group. After this we delete every row that is not
in this set.

### EXISTS

{% highlight sql %}
DELETE
FROM orders o1
WHERE EXISTS (
  SELECT 1
  FROM orders o2
  WHERE o1.item_id = o2.item_id AND
        o1.code = o2.code AND
        o1.id > o2.id
);
{% endhighlight %}

This reads as: "Delete every row from o1 that has the same item_id and code and
an id bigger than at least one id from o2". This will ensure that only the
duplicates with the lowest id remains.

### USING

{% highlight sql %}
DELETE
FROM orders o1 USING orders o2
WHERE o1.item_id = o2.item_id AND
      o1.code = o2.code AND
      o1.id > o2.id;
{% endhighlight %}

Which executes the same code in our `GROUP BY` example. We also use the same
technique explained above to keep the lowest id from our duplicated records.
We can see another example
[here](http://www.postgresql.org/docs/9.4/static/sql-delete.html#AEN78458).

### PARTITION (WINDOW FUNCTION)

{% highlight sql %}
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
{% endhighlight %}

Here we partition our records using the specified criteria to find duplicates
and sort this partition by `id`, after this we create a column with the row
number for each row in a partition. The last step is to select only the rows
with row numbers greater than 1 to delete.

### Temporary tables

{% highlight sql %}
CREATE TABLE temporary_orders AS
  SELECT DISTINCT ON (item_id, code) * FROM orders;

DROP TABLE orders;
ALTER TABLE temporary_orders RENAME TO orders;
{% endhighlight %}

This solution is fast, but you have to disable indexes and foreign key constraints
in your table to achieve this, which is not very practical. Still, it's a
possibility that we should take into account.

We can also use a `CREATE TEMPORARY TABLE` if our table should be dropped at the
end of the session.

### What if I don't have a unique identifier?

Just rename the `id` to `ctid` in our queries.

`ctid` is a [system column](http://www.postgresql.org/docs/9.4/static/ddl-system-columns.html)
representing the physical location of the row in our table. This
identifier is guaranteed to be unique, but changes over time if our rows are
updated.

----

That's it for now, if you know other solutions that I have not mentioned here
or have further questions feel free to leave me a tweet or email
so we can discuss things further!

See you in the next post!
