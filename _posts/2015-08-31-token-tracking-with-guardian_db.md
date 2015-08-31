---
  title: Token Tracking with GuardianDb
  date: 2015-08-31
  categories:
    - elixir
    - guardian
---

I've loved working on Guardian, but one thing has bothered me. When you logout,
vanilla Guardian (and all vanilla JWT implementations) don't actuall invalidate
the token. All the information for the token is stored inside the token itself.

This can lead to short term expiry keys to try and keep a handle on it. Even
with this, you can't actually revoke a token, so when someone logs out, their
token is still valid.

Initially my thought was to try and tie the token to the CSRF token, but this
turned out not to be such a good idea (and practially impossible with masked CSRF tokens). Instead I wanted a way to keep a tight leash on tokens, when you log out they should no longer be valid. I didn't want to load up Guardian with a bunch of database assumptions and baggage for everyones app though so it had to stay out of Guardian core.

The result is [GuardianDb](https://github.com/hassox/guardian_db). This is a
simple plugin that integrates via Guardian.Hooks.

With it:

* Every token is stored in the database (keyed by the jti - uuid)
* When it is encoded, a database entry is created
* Every time you verify (page request) a token it is checked. If the entry is
  missing, it does not verify
* When you logout, or call `Guardian.revoke!` the entry is removed

You can also clear out stale tokens by
using `GuardianDb.Token.purge_expired_tokens!`

Setup is simple (and included in the [README.md](https://github.com/hassox/guardian_db)). Generate the migration and then add to your config:

{% highlight elixir %}

config :guardian, Guardian,
       hooks: GuardianDb,
       # …

config :guardian_db, GuardianDb, repo: MyApp.Repo

{% endhighlight %}
