---
  title: Removing CSRF token type
  date: 2015-07-09
  categories:
    - elixir
    - guardian
---

Recently masked CSRF tokens have shown up in Phoenix. Masked CSRF tokens are an
important feature to prevent against the [BREACH attack](http://breachattack.com/).

Unfortunately masked tokens are incompatible with the way Guardian was using
them. It's not all bad though, working through it, it turns out that the way
Guardian was putting the CSRF into the JWT wasn't actually buying much in terms
of security. Since you need both pieces of information to connect to a channel,
you essentially have the lock and key and can reply the token.

After reviewing it I've decided to remove the functionality from Guardian. From
here it's clear that there needs to be a more robust solution for revoking
tokens. That'll be the focus of the next round of work on Guardian.
