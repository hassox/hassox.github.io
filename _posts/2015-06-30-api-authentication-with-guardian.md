---
  title: API Authentication with Guardian
  date: 2015-06-30
  categories:
    - elixir
    - guardian
---

Over my last couple of posts I've talked about [getting
started](http://hassox.github.io/elixir/guardian/2015/06/19/guardian-getting-started.html),
[permissions](http://hassox.github.io/elixir/guardian/2015/06/24/introducing-guardian-permissions.html)
and [simple email/password
auth](http://hassox.github.io/elixir/guardian/2015/06/29/simple-email-password-authentication.html).
It occured to me that I haven't covered how to do an API authentication.

Part of the beauty of using JWT is that they have an expiry time built into
them. In my examples I typically expire them in 30 days, but you can configure
(or overwrite) this to be any value.

So we've added Guardian, created a user and we so far have the ability to create
a 'login' where they're logged into the session, but what about API endpoints.

One thing that Guardian can do easily is authenticate your application for API
endpoints. Lets take a quick look at how that works.

In your router, add:

{% highlight elixir %}
  pipeline :api do
    plug :accepts, ["json"]
    plug Guardian.Plug.VerifyAuthorization
    plug Guardian.Plug.LoadResource
  end

  scope "/api/v1", PhoenixGuardian.Api.V1 do
    pipe_through [:api]

    resources "/users", UserController
  end
{% endhighlight %}

The verify authorization plug will check the Authorization header for a token,
and the LoadResource one will load any resource found there.

Ok, so now protecting your API endpoints is just like your web endpoints:

{% highlight elixir %}
# web/controllers/api/v1/user_controller.ex
defmodule PhoenixGuardian.Api.V1.UserController do
  use PhoenixGuardian.Web, :controller

  alias PhoenixGuardian.User
  alias PhoenixGuardian.SessionController

  plug Guardian.Plug.EnsureSession, on_failure: { SessionController, :unauthenticated_api }
  plug :action

  # …
  def index(conn, _params) do
    users = Repo.all(User)
    json(conn, %{ data: users, current_user: Guardian.Plug.current_resource(conn) })
  end
  # …
end
{% endhighlight %}

The interaction with API clients is just the same as web session clients. I've
used a different failure method in my SessionController, but other than this the
token is fine.

#### How do I get my token?

So far, we've only used the `sign_in` function on Guardian.Plug.
This is great for web pages but not quite enough for APIs. Lets
have a look at how to get a token:

{% highlight elixir %}
user = Repo.get(User, 1)
{ :ok, jwt, full_claims } = Guardian.mint(user, :api)
{% endhighlight %}

This will generate a JWT that you can provide to your client, valid for the
configured ttl. Once you get to the controller it's the same API as the browser
based authentication.

Your client can hand it in via the "Authorization" header.

{% highlight ruby %}
Authorization: <jwt>
{% endhighlight %}

For bonus points, you can also use it for channel authentication!

