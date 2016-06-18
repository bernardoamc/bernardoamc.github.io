---
layout:     post
title:      Contributing to Ecto
date:       2016-06-18 13:00:00
summary:    In this post I will present the workflow I utilized to contribute
            to an Elixir project called Ecto.
categories: elixir
---

Recently I sent a [Pull Request](https://github.com/elixir-ecto/ecto/pull/1496)
to allow [lateral joins](http://bernardoamc.github.io/sql/2015/06/23/postgres-lateral-join/)
in [Ecto](https://github.com/elixir-ecto/ecto). Ecto as the documentation
specifies is a domain specific language for writing queries and interacting
with databases in Elixir.

I decided to test my code by writing a simple [Phoenix](phoenixframework.org)
app linked to my local Ecto repository to see if I was doing any progress, but
I had a few issues trying to do it and that is exactly what made me write this
post.

### So, how do we use our custom Ecto with Phoenix?

The first thing I did was edit my `mix.exs` file and add my local `ecto`
repository as a direct depedency of the project by setting the `:path` option,
as below:

~~~elixir
  defp deps do
    [
      # other dependencies ...,
      {:ecto, path: "/my/local/path/ecto"}
    ]
  end
~~~

This didn't work since `ecto` is a dependency of a dependency (`phoenix_ecto`),
so after running `mix help deps` I realized that I should add the `:override`
option, since the documentation specifies the following:

> :override - if set to true the dependency will override any other definitions of itself by other dependencies

Changing `mix.exs` again:

~~~elixir
defp deps do
  [
    # other dependencies ...,
    {:ecto, path: "/my/local/path/ecto", override: true}
  ]
end
~~~

Turns out this still didn't solve the problem, after running `iex -S mix
phoenix.new` I got an error trying to compile `phoenix_ecto`, the reason
probably being due to the stable `phoenix_ecto` expecting another version
of `ecto` instead of `master` branch (which I was currently using).

To solve this problem we need to make our `phoenix_ecto` also point to
`master`, since we don't have a local `phoenix_ecto` repository how do we
do this? Relying again on `mix help deps` we can see that it supports the
`:github` and `:branch` options, just what we need:

~~~elixir
defp deps do
  [
    # other dependencies ...,
    {:phoenix_ecto, github: "phoenixframework/phoenix_ecto", branch: "master"},
    {:ecto, path: "/my/local/path/ecto", override: true}
  ]
end
~~~

After running `mix deps.get` and `iex -S mix phoenix.server` we get the same
problem with `phoenix_html`:

~~~elixir
defp deps do
[
  # other dependencies ...,
  {:phoenix_ecto, github: "phoenixframework/phoenix_ecto", branch: "master"},
  {:phoenix_html, github: "phoenixframework/phoenix_html", branch: "master"},
  {:ecto, path: "/my/local/path/ecto", override: true}
]
end
~~~

And finally everything works fine after `mix deps.get` and `iex -S mix
phoenix.server`.

That is all I wanted to show, hope it helps someone out there! Also, thank you
[Igor Florian](https://twitter.com/igorflorianfs) for the help and for this
awesome
[post](http://blog.plataformatec.com.br/2016/03/inspecting-changing-and-debugging-elixir-project-dependencies/)
about project dependencies.

See you in the next post!
