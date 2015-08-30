---
  title: Guardian - Renaming some things
  date: 2015-08-30
  categories:
    - elixir
    - guardian
---

I've been getting some feedback that some of the modules and function aren't
named such that their behaviour is immediately apparent, so I've renamed them.
The rename comes in with 0.6.0 and shouldn't impact everyone.

The changes are:

{% highlight text %}
Guardian.mint -> Guardian.encode_and_sign
Guardian.verify -> Guardian.decode_and_verify

Guardian.Plug.EnsureSession -> Guardian.Plug.EnsureAuthenticated
Guardian.Plug.VerifyAuthorization -> Guardian.Plug.VerifyHeader
{% endhighlight %}

The function signatures and module behaviour remains the same, it's just a
rename.
