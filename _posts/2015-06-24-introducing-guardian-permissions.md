---
  title: Introducing Guardian Permissions
  date: 2015-06-24
  categories:
    - elixir
    - guardian
---

I'm really loving using JWT's for authentication. They're a great little self
contained set of information that I can use, with or without hitting the db.

One thing that was missing was permissions. Anyone who's implemented OAuth will
be familiar with why these are needed on a per token basis. Imagine you head
over to a site to OAuth an app. That app will be authenticating as you, but
usually with a customizable list of permissions. One of the tricky parts of this
is that once you add OAuth, you have to make sure all your relevant endpoints
(including downstream s2s) are aware of those permissions and have some way to
get to them. Not a fun X hours/days tracking all that down. Don't worry.
Guardian has your back so you can make your apps permission aware as early as
you like.

JWTs need to be encoded so that they fit into the header of an HTTP request so
keeping them on the small side is pretty important. That's why guardian encodes
all permissions into a bitsting for you.

Enough! Lets see it.

### Config

You'll need to include your list of known permissions in the config. Don't worry
if you have to add some, Guardian will handle unknown permissions by ignoring
them.

{% highlight elixir %}
config :guardian, Guardian,
       permissions: %{
         default: [:read, :write],
         admin: [:dashboard, :make_payments]
       }
```
{% endhighlight %}

Seems pretty straight forward right? The `:default` and `:admin` represent
different sets of permissions. You can have as many sets as you like. I'd lean
towards more permissions per set, than more sets though.

A word of caution. Guardian encodes permissions into bitstrings. The _position_
of the permission in the list is what determines it's position in the bits.
Couple of things to keep in mind:

* Re-ordering is bad
* Renames are fine
* Removals need to keep their place in the list

Ok, now that we've got the caveats out of the way, lets use it.

### Sign in

You can encode these puppies right when you sign in.

{% highlight elixir %}
Guardian.Plug.sign_in(conn, resource, :token, perms: %{ default: [:read], admin: [:dashboard]})
{% endhighlight %}

Simple right. The same thing is true when minting manually.

{% highlight elixir %}
Guardian.mint(resource, :token, perms: %{ default: [:read], admin: [:dashboard]})
{% endhighlight %}

But what if I add a permission and I want the admin to have access to all the
things? (I hear myself asking myself).

{% highlight elixir %}
Guardian.mint(resource, :token, perms: %{ default: [:read], admin: Guardian.Permissions.max})
{% endhighlight %}

By using the `Guardian.Permissions.max/0` function, you get it all.

### Checking permissions

To check them, you'll need the claims.

{% highlight elixir %}
# Using a conn
claims = Guardian.Plug.current_claims(conn)

# In a channel
claims = Guardian.Channel.current_claims(socket)
{% endhighlight %}

Lets have a look then.

{% highlight elixir %}
# Check for the existence of all the permissions you need
Guardian.Permissions.from_claims(claims, :admin)
|> Guardian.Permissions.all?([:dashboard, :reconcile], :admin)

# Check for the existence of any the permissions you need
Guardian.Permissions.from_claims(claims, :admin)
|> Guardian.Permissions.any?([:dashboard, :reconcile], :admin)
{% endhighlight %}

There's also a plug version.

{% highlight elixir %}
defmodule MyApp.MyController do
  use MyApp.Web, :controller
  alias Guardian.Permissions
  alias Guardian.Plug.EnsurePermissions

  plug EnsurePermissions, on_failure: { MyApp.MyController, :forbidden }, admin: [:dashboard]

end
{% endhighlight %}

This will do an `all?` check on the permissions for any that you specify.

One thing to bear in mind with these permission sets. You shouldn't feel locked
in stone on them. If you make a set and then find that you'd like to have a
different set, just deprecate the first one and create a new one. No problem.

Lastly, just in case you want to get right down into it.

{% highlight elixir %}
Guardian.Permissions.to_value([:read, :write], :default) |> Guardian.Permissions.to_list(:default)
{% endhighlight %}


Guardian tries to keep it simple so I think theres not much more to say in a
blog post. Enjoy!
