---
layout:     post
title:      Hierarchical Data Modeling
date:       2015-04-14 22:30:00
summary:    Recently I was faced with a problem where there was a need to
            model some hierarchical data and I chose to solve it with the Adjacency
            List pattern. In this post we will see some advantages of this
            approach and how it can be even better in databases that implement
            recursive queries.
categories: sql
---

Choosing the `Adjacency List` pattern to solve hierarchical data modeling is one
of [many](http://troels.arvin.dk/db/rdbms/links/#hierarchical) options to solve this
kind of problem, some of them are `Nested Set`, `Closure Table`, `Flat Table`
and `Path Enumeration`. In this post I will investigate the benefits of using
this approach and also some problems.

### How it works?

We simply create a table with a field that references a row in this same table.
An example of a table like this would be:

{% highlight sql %}
CREATE TABLE categories (
  id serial PRIMARY KEY,
  parent_id integer,
  name varchar,
  FOREIGN KEY (parent_id) REFERENCES categories(id)
);
{% endhighlight %}

We can see that `parent_id` references an id in this same table. The rest
in straighfoward, to create a category that does not have a parent we leave
the field `parent_id` blank. Child categories can reference its parent by
id.

Let's see some examples:

{% highlight sql %}
INSERT into categories (name)
  VALUES ('sports');
INSERT INTO categories (parent_id, name)
  VALUES (1, 'football');
INSERT INTO categories (parent_id, name)
  VALUES (2, 'U-14');
INSERT INTO categories (parent_id, name)
  VALUES (2, 'U-17');
INSERT INTO categories (parent_id, name)
  VALUES (1, 'volleyball');
INSERT INTO categories (parent_id, name)
  VALUES (5, 'U-21');

SELECT * FROM categories;

id | parent_id |    name
----+-----------+------------
  1 |    (null) | sports
  2 |         1 | football
  3 |         2 | U-14
  4 |         2 | U-17
  5 |         1 | volleyball
  6 |         5 | U-21
(6 rows)
{% endhighlight %}

So, as we can see we have something like:

{% highlight sql %}
* sports
  * football
    * U-17
    * U-14
  * volleyball
    * U-21
{% endhighlight %}

Let's see the advantages of this approach.

### Advantages

It's pretty easy to **insert** a new row in the hierarchy:

{% highlight sql %}
INSERT INTO categories (parent_id, name)
  VALUES (5, 'U-18');
{% endhighlight %}

It's also easy to **change** the parent of a row (changing its subtree):

{% highlight sql %}
UPDATE categories
SET parent_id = 1
WHERE name = 'U-18';
{% endhighlight %}

Querying the **direct parent or descendant** of a category is straightforward:

{% highlight sql %}
SELECT c1.*
FROM categories c1 INNER JOIN categories c2 ON c2.parent_id = c1.id
WHERE c2.name = 'football';

 id | parent_id |  name
----+-----------+--------
  1 |    (null) | sports
(1 row)

SELECT c1.*
FROM categories c1 INNER JOIN categories c2 ON c1.parent_id = c2.id
WHERE c2.name = 'football';

 id | parent_id | name
----+-----------+------
  3 |         2 | U-14
  4 |         2 | U-17
(2 rows)
{% endhighlight %}

But as nothing is perfect, we also have some issues.

### Issues

We can't **delete a row** without updating first its descendants parent. For example,
suppose we want do delete `football`. To do this and keep the tree consistent,
we would have to update `U-14` and `U-17` parent_id to `sports`, using our
queries to get the parent and direct descendants of a row.

{% highlight sql %}
UPDATE categories SET parent_id = (
  SELECT c1.id
  FROM categories c1
  INNER JOIN categories c2 ON c2.parent_id = c1.id
  WHERE c2.name = 'football'
)
WHERE id IN(
  SELECT c1.id
  FROM categories c1
  INNER JOIN categories c2 ON c1.parent_id = c2.id
  WHERE c2.name = 'football'
);
{% endhighlight %}

And after that we delete our row:

{% highlight sql %}
DELETE from categories where name = 'football';
{% endhighlight %}

It's also not easy to **delete a subtree**. Suppose we want to delete the row that
contains `football` and all its descendants. We would need to find all
descendants of `football` and delete them in reverse order to avoid
constraint errors. So suppose we have the following:

{% highlight sql %}
* sports
  * football
    * U-17
      * male
      * female
    * U-14
      * male
      * female
  * volleyball
    * U-21
{% endhighlight %}

To delete `football` subtree we would have to delete first the `male` and `female` from
both `U-14` and `U-17`. After that we would have to delete `U-14` and `U-17` and finally
we would delete `football`. But what if we have 5 or 10 levels of nesting?

To solve this problem we would need to use `recursive` queries in a database
like PostgreSQL to achieve something like:

{% highlight sql %}
WITH RECURSIVE CategoriesTree (id, parent_id, name) AS (
  SELECT *
  FROM categories
  WHERE name = 'football'
UNION ALL
  SELECT c.*
  FROM CategoriesTree ct INNER JOIN categories c
  ON (ct.id = c.parent_id)
)
DELETE FROM Categories WHERE id IN(
  SELECT id FROM CategoriesTree
);
{% endhighlight %}

The query above gets all descendants ids including the row we want to delete
and pass its ids to DELETE, the database takes care of deleting the rows in
the right order.

To **query all the descendants** we would also need to use recursive queries.

{% highlight sql %}
WITH RECURSIVE CategoriesTree (id, parent_id, name) AS (
  SELECT *, 0 as depth
  FROM categories
  WHERE name = 'sports'
UNION ALL
  SELECT c.*, ct.depth + 1 as depth
  FROM CategoriesTree ct INNER JOIN categories c
  ON (ct.id = c.parent_id)
)
SELECT * FROM CategoriesTree;

 id | parent_id |   name   | depth
----+-----------+----------+-------
  1 |    (null) | sports   |     0
  2 |         1 | football |     1
  3 |         2 | U-14     |     2
  4 |         2 | U-17     |     2
  7 |         3 | male     |     3
  8 |         3 | female   |     3
  9 |         4 | male     |     3
 10 |         4 | female   |     3
{% endhighlight %}

### Wrap Up

As we saw, the `Adjacency List` pattern has good and not so good use cases and
it's important to consider what is your use case before adopting this solution. Keep
in mind that there is no such thing as a perfect solution, just one that fits
your problem really well.

### TL;DR:

Use the `Adjacency List` if your use case uses mostly:

* Inserts
* Updates
* Queries to get just the parent and/or direct descendants

Keep an eye on it if it uses mostly:

* Deletes (row or subtree)
* Queries to get all descendants

See you in the next post!
