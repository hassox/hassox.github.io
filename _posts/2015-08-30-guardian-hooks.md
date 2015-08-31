---
  title: Guardian Hooks
  date: 2015-08-30
  categories:
    - elixir
    - guardian
---

Some time ago I introduced Hooks to Guardian. Hooks are a mechanism for you to
plug-in to lifecycle of authentication. These can be useful to extend or
customize the behaviour of Guardian within your application.

A Guardian.Hook is implemented as a behvaiour with default implementations of
each callback so you only need to implement what you're interested in. The
available hooks are:

{% highlight text %}
defcallback before_encode_and_sign(resource :: term, type :: atom, claims :: Map)
defcallback after_encode_and_sign(resource :: term, type :: atom, claims :: Map, token :: String.t)
defcallback after_sign_in(conn :: Plug.Conn.t, location :: atom | nil)
defcallback before_sign_out(conn :: Plug.Conn.t, location :: atom | nil)
defcallback on_verify(claims :: Map, jwt :: String.t)
defcallback on_revoke(claims :: Map, jwt :: String.t)
{% endhighlight %}

To create your own module just use the Guardian.Hooks module.

{% highlight elixir %}

defmodule MyHooks do
  use Guardian.Hooks

  def before_encode_and_sign(resource, type, claims) do
    # to keep this as an authenticated token, return { :ok, { resource, type, claims }
    # to fail and prevent the mint from happening return { :error, :reason }
  end
end
{% endhighlight %}

### before\_encode\_and\_sign

Runs before the jwt is generated. This can be used to add claims, or halt and
return an error.

### after\_encode\_and\_sign

This runs after the JWT has been encoded. Returning an error will not halt, but
can be used to to extend behaviour. For example store the token in a DB.

### after\_sign\_in

Runs after a token is signed in (via a session). Use this to record information
about logins etc.


### before\_sign\_out

Before logging out of the session, these hooks run. The session will be logged
out regardless but provides an opportunity to record it.

### on\_verify

Runs after claims have successfully been verified from the JWT which occurs
every time a request is verified, which could come from a session, header or
channel.

###  on\_revoke

Called via Guardian.revoke!. This is run when the token should be considered
revoked. By default Guardian takes no specific steps to consider a token
revoked. If you're storing tokens in the DB though, this would be the time to
delete them.
