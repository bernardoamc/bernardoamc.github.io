---
layout:     post
title:      Adding new validations
date:       2017-02-27 23:00:00
summary:    Adding existing validations in our current project can be
            something dangerous. What should we be aware of and how to
            deal with it?
categories: general
---

**TL;DR - Mind your data.**

Web frameworks like Ruby on Rails or Django do a great job abstracting a lot
of complex problems from us developers. From validations to migrations, we just
need to add a few more lines to our code and things will work just fine most of
the time. This usually cause a huge disconnect from the database layer
(specially for new developers) and is the main reason for data inconsistency and
lack of performance in the long run.

We will investigate the process of adding new validations (not tied to the
database) to an existing project and how to deal with our current data.
These scenarios can happen in any language/framework, so I will not focus in
specifics. Let's begin.

#### I want to add a new validation, what should I do?

Nothing wrong with that, business rules tend to evolve all the time and we can't
possibly anticipate and model everything right in just one go. This means that
we will already have data stored somewhere though, and this usually complicate
things. Let's start with the **happy path**:

1. Query your database and check if all your current data conforms with the new
validation.
2. It does?!
3. Add the new validation.

#### But what if my current data does not conform to the new validation?

This can happen in a lot of situations, for example,  when you want to enforce
uniqueness, but your column has duplicate values. Or maybe you don't want to
allow NULLs, but you already have a few. The list goes on, the takeaway here is
that we are enforcing something in our application, but not in our database.
Here comes our **second best scenario:**

1. Can you adjust your current data?
2. Yes?!
3. Make sure active code paths can’t create any more invalid data.
4. Create a maintenance script and run it.
5. Query your database again and check if all your data conforms with the new
validation.
  * Yes? Add the new validation.
  * No? Back to step 1.

Step 5 is **really** important, always query your database after trying to adjust
your data, it's surprisingly easy to miss some edge cases.

#### My data does not conform to the new validation and I can't backfill! ;(

This is tricky, sorry to hear that! In these cases we will be forced to live
with inconsistent data for a while. This situation is very particular for each
project, but what usually happens is that the project will rely on their users to
update the data by enforcing the new rules in the web form or API. This means we
can have invalid that in our database for a long period, which is always bad.

If this data is critical we can ask our users to adjust it for us, but we should
avoid annoying our users as much as possible.

See you in the next post!
