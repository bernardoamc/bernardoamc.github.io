---
layout:     post
title:      Views in Postgres
date:       2015-03-28 20:00:00
summary:    Exploring views in the latest versions of PostgreSQL. How do we use
            a view and what are they good for? What is a materialized view and
            when should I use it?
categories: postgres
---

Quoting the great book *PostgreSQL Up and Running*, a view is nothing more than
a query permanently stored in the database. Views have been evolving a lot in
the latest versions of PostgreSQL and deserve some study to be well used.

The simplest use of a view can be something like:

{% highlight sql %}
CREATE OR REPLACE VIEW logs_2015 AS
SELECT value, year FROM logs where year = 2015;
{% endhighlight %}

This view uses only a single table, but we can also create views that uses more
than one table:

{% highlight sql %}
CREATE OR REPLACE VIEW author_books AS
SELECT a.name, b.name, b.released_at
FROM authors AS a INNER JOIN books AS b
ON a.id = b.author_id;
{% endhighlight %}

### Materialized Views

Since version `9.3` we can use a feature called `MATERIALIZED VIEW`. It's the
same as a normal view, but the data is cached when the view is created. It's
pretty useful when a commonly used query takes a long time to run and we don't
need real time data. We can also create indexes for materialized views, how
amazing is that?

To update a `MATERIALIZED VIEW` we need to use the command `REFRESH MATERIALIZED
VIEW view`, this will block queries while the refresh is running. In version
`9.4` we have the option `CONCURRENTLY` that allows materialized views to be queried
even during refresh, but there are some restrictions as can be seen
[here](http://www.postgresql.org/docs/9.4/static/sql-refreshmaterializedview.html)
and the refresh takes longer.

Let's see an example:

{% highlight sql %}
CREATE MATERIALIZED VIEW logs_2015 AS
SELECT value, year FROM logs where year = 2015;
{% endhighlight %}

Note that in this case we don't have the `CREATE OR REPLACE` option. This
happens because materialized views must be dropped if they already exist.

The [PostgreSQL
Documentation](http://www.postgresql.org/docs/devel/static/index.html)
has a great section about [materialized views](http://www.postgresql.org/docs/9.4/static/rules-materializedviews.html).

### INSERT, UPDATE and DELETE operations

Pior to version `9.3`, these operations were only supported using `RULES` or
`TRIGGERS`. From version `9.3` onwards these operations are all permitted in
a view that references a single table, but we can only insert new data if our
view has the primary key *or* the primary key is auto incremented.

In a view that references multiple tables we need to use triggers to achieve
these operations. Explaining triggers would be a post on its own, but to start
the
[documentation](http://www.postgresql.org/docs/9.4/static/plpgsql-trigger.html)
is a pretty good place.

One thing to keep an eye on is that your updates can place your records outside
of your view. So, if we have a view that filters our records by the year of
`2015` and we update these records to the year of `2016`, our view will be empty
after the operation. In version `9.4` we have the option `WITH CHECK OPTION`
that blocks these kind of updates.

{% highlight sql %}
CREATE OR REPLACE VIEW logs_2015 AS
SELECT value, year FROM logs where year = 2015
WITH CHECK OPTION;
{% endhighlight %}

If we now try the following:

{% highlight sql %}
UPDATE logs_2015 SET year = 2016 WHERE value = 30;
ERROR:  new row for relation "logs_2015" violates
        check constraint "logs_2015_year_check"
DETAIL:  Failing row contains (15, 30, 2016).
{% endhighlight %}

We can see that we get an error, because our update would put the record
outside the scope of our view.

### Dropping a view

It's really easy to drop a view, all you really need to do is:

{% highlight sql %}
DROP VIEW view;
{% endhighlight %}

There are some extra options that can be found in the
[documentation](http://www.postgresql.org/docs/9.4/static/sql-dropview.html).

### TL;DR

Views are great to simplify working with data, to structure and restrict access
to it. It's also good for caching complex queries and ensuring that the proper queries
are running. Consider using one of them in your next project if you find a
similar use case.
