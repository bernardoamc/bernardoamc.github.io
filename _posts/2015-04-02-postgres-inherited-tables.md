---
layout:     post
title:      Inherited Tables in Postgres
date:       2015-04-02 20:00:00
summary:    Investigating inherited tables feature from PostgreSQL. Is
            inheritance useful in a relational database? If so, when should
            I use it? How they work and what should I keep an eye on during
            the implementation? Let's find out!
categories: postgres
---

There is a significant amount of features supported only by PostgreSQL that are
really interesting, the one that we are going to explore in this post is called
inherited tables.

### What are they?

As the name suggests, inherited tables are tables that inherit from another one,
commonly named the *parent* table. This means that the new table will have its own
columns plus the columns from the parent table.

### When should it be used?

Normally for data partitioning, so things like *logs* and *statistics* are valid
cases for creating inherited tables. As we will see ahead, there are a few
things to keep an eye on when using this feature.

### Example

{% highlight sql %}
CREATE TABLE statistics(
  id serial PRIMARY KEY,
  name varchar,
  created_at timestamp without time zone
);

CREATE TABLE statistics_2015 (
  CHECK (
    created_at >= '2015-1-1'::timestamp
    AND
    created_at < '2016-1-1'::timestamp
  )
) INHERITS (statistics);

CREATE INDEX idx_statistics_2015_created_at
  ON statistics USING btree(created_at);
{% endhighlight %}

As we can see we create a table named *statistics* and a child table named
*statistics_2015* with a *CHECK CONSTRAINT*. This constraint is necessary so the
*query planner* knows which tables it should look when querying the parent table.

We also create a simple index, so the query will be really fast in the child table.

### Things to keep in mind

*CHECK* constraints are inherited, but the following are not:

  * Indexes
  * Primary Key Constraints
  * Foreign Key Constraints
  * Uniqueness Constraints
  * Triggers or Rules

When one of these need to be used it's time to reevaluate your use case for
inherited tables.

### Inserting data

Normally we would need to insert data directly into the child tables, but if we
want to simply `INSERT INTO statistics ...` we would need to use a trigger to do it.

First we need to create a function that will be called by the trigger. This
function will dinamically check in which year we are and will call the proper
child table:

{% highlight sql %}
CREATE OR REPLACE FUNCTION statistics_insert_trigger()
RETURNS TRIGGER AS $$
BEGIN
  EXECUTE format(
    'INSERT INTO %I VALUES (($1).*)',
    concat_ws('_', TG_TABLE_NAME, date_part('year', current_date))
  )
  USING NEW;

  RETURN NULL;
END;
$$
LANGUAGE plpgsql;
{% endhighlight %}

Now the trigger:

{% highlight sql %}
CREATE TRIGGER insert_statistics_trigger
  BEFORE INSERT ON statistics
  FOR EACH ROW EXECUTE PROCEDURE statistics_insert_trigger();
{% endhighlight %}

### Advantages

  * We can create new partitions easily
  * To remove old records we just need to `DROP` old tables
  * We can remove partitions and deal with child tables separately (`ALTER TABLE
    statistics_2015 NO INHERIT statistics;`)

### TL;DR:

Inherited Tables are great for data partitioning but they require some careful
attention because lots of things are not inherited automatically. Another thing
to keep in mind is that your `CHECK CONSTRAINTS` should be *mutually exclusive*.

Have fun with inherited tables!
