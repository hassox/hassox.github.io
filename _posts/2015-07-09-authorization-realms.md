---
  title: Adding realm support for Authorization headers
  date: 2015-08-30
  categories:
    - elixir
    - guardian
---

When you're verifying tokens with Guardian, for API access you probably want to
use the Authorization header.

Usually these headers look something like

{% highlight text %}
Authorization: Bearer <token>
{% endhighlight %}

The bearer part is known as the _realm_. You might want to use different realms.
I remember writing and OAuth provider, and as part of renewing your token the
API required that the client and user tokens were set. For example, the client
being the application requesting the renual, and the user for the user who owns
the token.

These authentication headers can both be set on the same response.

{% highlight text %}
Authorization: Client <application token>
Authorization: User <user token>
{% endhighlight %}

Guardian can handle the easy case of token or many if that's what you need.
These can go in your route file to form a plugin, or you can use them directly
in a controller.

Using the above as an example, you can include them in your pipelines

{% highlight elixir %}
pipeline :api do
  plug :accepts, ["json"]
  plug Guardian.Plug.VerifyHeader, realm: "Client", key: :default
  plug Guardian.Plug.VerifyHeader, realm: "User", key: :user
  plug Guardian.Plug.LoadResource, key: :user
end
{% endhighlight %}

Inside your controller action, you can access each set in the usual ways:

{% highlight elixir %}
  plug Guardian.Plug.EnsureAuthenticated, on_failure: { SomeModule, :client_failure }
  plug Guardian.Plug.EnsureAuthenticated, on_failure: { SomeModule, :user_failure }, key: :user

  def index(conn, params) do
    Guardian.Plug.claims(conn) # Fetch the claims for the client/application
    Guardian.Plug.claims(conn, :user) # Fetch the claims for the user
    # snip
  end
{% endhighlight %}

Adding realm support is important for supporting robust API support. Having it
integrated seamlessly with Guardian should make it dead simple.
