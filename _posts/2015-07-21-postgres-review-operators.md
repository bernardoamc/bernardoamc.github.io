---
layout:     post
title:      Review of operators in Postgres
date:       2015-07-21 14:00:00
summary:    In this post we will review and learn how to apply some operators in
            Postgres. The datatypes that we will investigate are Array, JSON and JSONB
            since they are newer and not so intuitive.
categories: sql
---

Let's investigate some operators in Postgres and how to use them
in our day-to-day work. The objective is to have a centralized guide.

### Array

+ `@>` and `<@` are operators that checks if an Array is contained by another.
The first one checks if the first Array **contains** the second. The latter
checks if the first Array is **contained by** the second.

{% highlight sql %}
SELECT array[1,3,6] @> array[1,4,7];

 ?column?
----------
 f
(1 row)

SELECT array[1,3,6] @> array[1,3];

 ?column?
----------
 t
(1 row)

SELECT array[1,3] <@ array[4,5,6];

 ?column?
----------
 f
(1 row)

SELECT array[4,6] <@ array[4,5,6];

 ?column?
----------
 t
(1 row)
{% endhighlight %}

+ `&&` tells if an Array has **elements in common** with another Array.

{% highlight sql %}
SELECT array[3,6] && array[1,4,7];

 ?column?
----------
 f
(1 row)

SELECT array[2,3,6] && array[1,3,6];

 ?column?
----------
 t
(1 row)
{% endhighlight %}

+ There is no operator if you need an **intersection**, but we can achieve the
same result using:

{% highlight sql %}
SELECT array(
  SELECT UNNEST(array[3,6,7,8])
  INTERSECT
  SELECT UNNEST(array[4,6,7,9])
);

 array
-------
 {6,7}
(1 row)
{% endhighlight %}

We can use **UNION** instead of the **INTERSECT** to join elements from each
Array without duplicates.

### JSON

There are three kinds of operators and each one works with arrays and
key/values.

+ `->` Returns an array element by index `OR` a value by key as **JSON**.

{% highlight sql %}
SELECT '[{"name": "bernardo"},{"country": "brazil"}]'::json->0;

       ?column?
----------------------
 {"name": "bernardo"}
(1 row)

SELECT pg_typeof(
  '[{"name": "bernardo"},{"country": "brazil"}]'::json->0
);

 pg_typeof
-----------
 json
(1 row)

SELECT '{"name": "bernardo", "country": "brazil"}'::json->'name';

  ?column?
------------
 "bernardo"
(1 row)

SELECT pg_typeof(
  '{"name": "bernardo", "country": "brazil"}'::json->'name'
);

 pg_typeof
-----------
 json
(1 row)
{% endhighlight %}

Since `->` returns a **JSON** we can chain this operator:

{% highlight sql %}
SELECT '{"country": {"name": "brazil"} }'::json->'country'->'name';

?column?
----------
 "brazil"
(1 row)
{% endhighlight %}

+ `->>` Returns an array element by index `OR` a value by key as **TEXT**.

{% highlight sql %}
SELECT '[{"name": "bernardo"},{"country": "brazil"}]'::json->>0;

       ?column?
----------------------
 {"name": "bernardo"}
(1 row)

SELECT pg_typeof(
  '[{"name": "bernardo"},{"country": "brazil"}]'::json->>0
);

 pg_typeof
-----------
 text
(1 row)

SELECT '{"name": "bernardo", "country": "brazil"}'::json->>'name';

  ?column?
------------
 bernardo
(1 row)

SELECT pg_typeof(
  '{"name": "bernardo", "country": "brazil"}'::json->>'name'
);

 pg_typeof
-----------
 text
(1 row)
{% endhighlight %}

*We cannot chain `->>` since it returns a TEXT.*

+ `#>` and `#>>` receives a path as an Array and returns a JSON and
TEXT respectively. Both return `NULL` if the path cannot be found.

{% highlight sql %}
SELECT '{"state": {"name": "abc"} }'::json#>ARRAY['state', 'name'];

?column?
----------
 "abc"
(1 row)

SELECT pg_typeof(
  '{"state": {"name": "abc"} }'::json#>ARRAY['state','name']
);

 pg_typeof
-----------
 json
(1 row)

SELECT '{"state": {"name": "abc"} }'::json#>>ARRAY['state', 'name'];

?column?
----------
 abc
(1 row)

SELECT pg_typeof(
  '{"state": {"name": "abc"} }'::json#>ARRAY['state','name']
);

 pg_typeof
-----------
 text
(1 row)
{% endhighlight %}

### JSONB

`JSONB` adds new operators on top of the existing ones for the `JSON` datatype.

+ `@>` and `<@` operators performs like the ones we saw for the Array datatype.
The first checks if the left JSONB contains the one in the right and the latter
checks the inverse.

{% highlight sql %}
SELECT '{"age": 20, "score": 7}'::jsonb @> '{"score":7}'::jsonb;

 ?column?
----------
 t
(1 row)

SELECT '{"age": 20, "score": 7}'::jsonb @> '{"score":5}'::jsonb;

 ?column?
----------
 f
(1 row)

SELECT '{"score": 7}'::jsonb <@ '{"age": 20, "score": 7}'::jsonb;

 ?column?
----------
 t
(1 row)

SELECT '{"score": 5}'::jsonb <@ '{"age": 20, "score": 7}'::jsonb;

 ?column?
----------
 f
(1 row)
{% endhighlight %}

+ `?`  checks if a key exists.
+ `?|` checks if **ANY** of the keys exists.
+ `?&` checks if **ALL** of the keys exists.

{% highlight sql %}
SELECT '{"age": 15, "score":7}'::jsonb ? 'age';

 ?column?
----------
 t
(1 row)

SELECT '{"age": 15, "score":7}'::jsonb ?| ARRAY['age', 'state'];

 ?column?
----------
 t
(1 row)

SELECT '{"age": 15, "score":7}'::jsonb ?& ARRAY['age', 'state'];

 ?column?
----------
 f
(1 row)

SELECT '{"age": 15, "score":7}'::jsonb ?& ARRAY['age', 'score'];

 ?column?
----------
 t
(1 row)
{% endhighlight %}

That's it for now, if you know other useful operators that were
not mentioned here or have a question feel free to leave me a tweet
so we can discuss things further!

See you in the next post!
