---
layout:     post
title:      Unlogged Tables in PostgreSQL
date:       2015-03-26 20:00:00
summary:    In this post we will see a quick review of unlogged tables in PostgreSQL,
            a feature that doesn't have many use cases, but is really useful
            when you know how and where to use it.
categories: postgres
---

PostgreSQL has lots of cool features that are not explored most of the time. In
this post we will see unlogged tables, a feature added in PostgreSQL 9.1 that
doesn't have lots of applications, but works really well in a few cases.

### What are they?

Unlogged tables are tables that are not part of write-ahead logging
([WAL](http://www.postgresql.org/docs/9.4/static/wal-intro.html)), a
standard method to ensure data integrity. This means that in the case of a
server crash, all your data residing in an unlogged table will be lost, so keep
an eye on that.

### When should it be used?

Unlogged tables should be used for data that are not critical, mostly *logs*,
*sessions* or some data that we pre-computed to speed up some future operation
and don't really care if it is lost.

### Example

{% highlight sql %}
CREATE UNLOGGED TABLE http_sessions (
  session_id text PRIMARY KEY,
  created_at timestamptz,
  ...
);
{% endhighlight %}

As we can see the only difference in syntax is the `UNLOGGED` keyword.

### Things to keep in mind

  * GiST indexes are not supported.
  * Indexes in unlogged tables are also unlogged.
  * Data are not replicated to standby servers.
  * In case of a server crash all your data will be lost.

### Advantages

Since there is no `WAL`, writing data to the table will be REALLY fast.

### TL;DR:

Use unlogged tables when you don't mind if your data is lost and want to
write operations to be really fast in your tables.
