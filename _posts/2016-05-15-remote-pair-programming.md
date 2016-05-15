---
layout:     post
title:      Remote Pair Programming
date:       2016-05-15 13:00:00
summary:    In this post I will present a workflow for remote pair programming
            that is currently working for me and a few tips to easier the process.
categories: pair_programming
---

I know that for now this blog has been all about SQL and PostgreSQL, but I had no
plans to exclusively talk about these topics, it just happened. So, for the
first time, let's talk about something else, let's talk about remote pair
programming.

Recently I have been involved in a few projects and some of them requires pair
programming sessions, the only problem being that the contributors don't share
the same physical location. No problem, everyone has access to a good computer
with reliable and fast internet access these days, right? Well, no... not at
all! And we shouldn't expect it, otherwise we are sure to have problems in the
long run (thank you [Victor](https://twitter.com/kotp) and
[Roque](https://twitter.com/repinel) for the help).

So, what can we do to easy pair programming sessions? I think it all boils
down to a workflow that uses the minimum required internet access and processing
power. This currently means to me that we should use the `command line` as much as
possible. So, how can we achieve this?

### The tools

- [tmate](https://tmate.io/) (a fork of [tmux](https://tmux.github.io/))
- [Vim](http://www.vim.org/about.php)
- Other tools like [w3m](http://w3m.sourceforge.net/)

#### tmate

[tmate](https://tmate.io/) is a fork of [tmux](https://tmux.github.io/) that
allows instant terminal sharing. It is pretty easy to install on OSX or similar
systems and can coexist with `tmux` on the same system. After installing we can
simple type `tmate` in our `terminal` and everything will be set up for us.

Let's see an example:

~~~terminal
$ tmate
$ tmate show-messages
[tmate] Connecting to ssh.tmate.io...
[tmate] Note: clear your terminal before sharing readonly access
[tmate] web session read only: https://tmate.io/t/ro-AbCdE
[tmate] ssh session read only: ssh abcde@ny2.tmate.io
[tmate] web session: https://tmate.io/t/aBcDe
[tmate] ssh session: ssh EdCbA@ny2.tmate.io
~~~

What I normally do is share the `ssh session` or `ssh read only session`,
so after typing `ssh EdCbA@ny2.tmate.io` the other person will be able
to see and access our terminal session. Easy, right?

We can see that there is also an `https link`, so it is an easier entry
point for people not familiar with the `command line`.

It is important to mention that we could achieve this with `tmux` also, but
`tmate` is easier for people with no experience since there is practically
no setup involved.

#### Vim

Vim is a text editor that enables efficient text editing. People might argue
that it is a complex editor and that the learning curve is too steep, but I
have found that even people with no experience can learn the basic usage and be
productive in pair programming sessions with the right guidance.

If both people are power users, try to not use a configuration that change Vim's
expected behavior. For example, changing your leader key to comma.

#### Other tools

So, the goal is to use the `command line` as much as possible, that being said
sometimes we need to check something up on our browser and this disrupts the
pair programming session. For this example we could use [w3m](http://w3m.sourceforge.net/)
so both people can see the web page directly on the terminal.

Normally I also use Skype or Google Hangout for voice, but it is not something
required though.

And that's it! It is a really simple workflow and I think this is the beauty of
it. Do you have a different workflow that works for you? I would be glad to hear
about it, just send me an email or leave a message on twitter!

See you in the next post!
