---
  title: Simple email password Authentication
  date: 2015-06-29
  categories:
    - elixir
    - guardian
---

Update: With the arrival or masked CSRF keys in Phoenix, I've re-thought the
CSRF protection baked into Guardian. Ultimately it doesn't provide the kinds of
assurances I originally had in mind so this feature has been removed.

--------------------------------------------------------

I've been head down with thinking about Guardian and how the far out cases it
could be used for will work together. Service2Service, Single sign on, OAuth
provider, stuff like that. It occured to me that there is a bunch of really easy
to handle things that from the outside might look a little daunting. With that
in mind, I wanted to walk through setting up an application for simple,
every-day email/password authentication. _Almost_ the simplest auth setup
around.

A quick top level overview for the uninitiated. Guardian is an authentication
library that uses JSON Web Tokens (JWT) as it's base unit of authentication.
It's a signed collection of _claims_ that can be used in a session or an
Authorization header. They're great, but this post isn't about them.

I don't want to get _too_ bogged down in how to create a User resource. This is
going to vary for you, but if you want to see an example of setting up a simple
User model with email/username using BCrypt you can checkout the [one I use in my example application](https://github.com/hassox/phoenix_guardian/blob/master/web/models/user.ex).

This uses a simple email/encrypted password, where the password is hashed with
BCrypt (using
[Comeonin](https://github.com/elixircnx/comeoni://github.com/elixircnx/comeonin)).

So lets assume that you have [created a
user](https://github.com/hassox/phoenix_guardian/blob/master/web/controllers/user_controller.ex) and you're ready to get going. Lets get setup.

#### Installing Guardian

config.exs

{% highlight elixir %}
config :joken, config_module: Guardian.JWT

config :guardian, Guardian,
      issuer: "PhoenixGuardian",
      ttl: { 10, :days },
      verify_issuer: true,
      secret_key: "lksdjowiurowieurlkjsdlwwer",
      serializer: PhoenixGuardian.GuardianSerializer
{% endhighlight %}

web/routes.ex

{% highlight elixir %}
defmodule PhoenixGuardian.Router do
  use PhoenixGuardian.Web, :router

  pipeline :browser_session do
    plug Guardian.Plug.VerifySession
    plug Guardian.Plug.LoadResource
  end

  scope "/", PhoenixGuardian do
    pipe_through [:browser, :browser_session] # Use the default browser stack

    get "/login", SessionController, :new, as: :login
    post "/login", SessionController, :create, as: :login
    delete "/logout", SessionController, :delete, as: :logout
    get "/logout", SessionController, :delete, as: :logout

    resources "/users", UserController
  end
{% endhighlight %}

We've setup a pipeline for authenticating browser requests (i.e. requests from
the session). Note that this will not actually kick anyone from the app, that
will come later.

You'll need a serializer. I put mine in lib/phoenix\_guardian/guardian\_serializer.ex

{% highlight elixir %}
defmodule PhoenixGuardian.GuardianSerializer do
  @behaviour Guardian.Serializer

  alias PhoenixGuardian.Repo
  alias PhoenixGuardian.User

  def for_token(user = %User{}), do: { :ok, "User:#{user.id}" }
  def for_token(_), do: { :error, "Unknown resource type" }

  def from_token("User:" <> id), do: { :ok, Repo.get(User, String.to_integer(id)) }
  def from_token(thing), do: { :error, "Unknown resource type" }
end
{% endhighlight %}

The serializer just fetches your resource, or given a resource, it serializes it
into the token.

#### Logging in

We added the SessionController into the routes. This just provides the login
form, and handles the result. It is _also_ where the magic happens. Inside this
controller, you're issued a JWT into your session.

{% highlight elixir %}
defmodule PhoenixGuardian.SessionController do
  use PhoenixGuardian.Web, :controller

  alias PhoenixGuardian.User

  def new(conn, params) do
    changeset = User.login_changeset(%User{})
    render(conn, PhoenixGuardian.SessionView, "new.html", changeset: changeset)
  end

  def create(conn, params = %{}) do
    user = Repo.one(UserQuery.by_email(params["user"]["email"] || ""))
    if user do
      changeset = User.login_changeset(user, params["user"])
      if changeset.valid? do
        conn
        |> put_flash(:info, "Logged in.")
        |> Guardian.Plug.sign_in(user, :token)
        |> redirect(to: user_path(conn, :index))
      else
        render(conn, "new.html", changeset: changeset)
      end
    else
      changeset = User.login_changeset(%User{}) |> Ecto.Changeset.add_error(:login, "not found")
      render(conn, "new.html", changeset: changeset)
    end
  end

  def delete(conn, _params) do
    Guardian.Plug.sign_out(conn)
    |> put_flash(:info, "Logged out successfully.")
    |> redirect(to: "/")
  end
end
{% endhighlight %}

There's a couple of methods, but most of what's happening is straight forward.

A user lands on the "new" action and we generate a form. We use the
`User.login_changeset` with the form. It's a simple changeset from the [user.ex model](https://github.com/hassox/phoenix_guardian/blob/master/web/models/user.ex):

{% highlight elixir %}
def login_changeset(model), do: model |> cast(%{}, ~w(), ~w(email password))

def login_changeset(model, params) do
  model
  |> cast(params, ~w(email password), ~w())
  |> validate_password
end
{% endhighlight %}

This will just give us a nice error message and interface for checking the
password.

Once the form is posted to the `create` method, we'll check the
`login_changeset` and make sure it's a good email/pass match.

Once there, we do the 'logging in' part.

{% highlight elixir %}
  |> Guardian.Plug.sign_in(user, :token)
{% endhighlight %}

This line invokes your serializer, generates your token with an expiry of what
you configured (10 days for this example), pushes it into the session and makes
it available on the request for further use. The `:token` option is essentially
a label to identify the type of token (used in the `:aud` field). This can be
anything but should be consistent.

#### Protecting your urls

So, now we have a logged in user, lets protect something.

In the user\_controller.ex

{% highlight elixir %}
defmodule PhoenixGuardian.UserController do
  use PhoenixGuardian.Web, :controller

  alias PhoenixGuardian.User
  alias PhoenixGuardian.SessionController
  alias Guardian.Plug.EnsureSession

  plug EnsureSession, %{ on_failure: { SessionController, :new } } when not action in [:new, :create]

  # …

  def edit(conn, params) do
    user = Guardian.Plug.current_resource(conn)
    changeset = User.update_changeset(user)
    render(conn, "edit.html", user: user, changeset: changeset)
  end

  # …

{% endhighlight %}

I've cut most of it, but you can see in the edit action how we fetch the currently logged in user.

This illustrates a very simple use of Guardian. From here there are a lot of
places to go.

* api vs session credentials
* custom 'claims' in the JWT
* permissions
* hooks
* mashups with existing frameworks

There's a lot of places to go. I want to make sure the road to getting in is a
simple as possible.

