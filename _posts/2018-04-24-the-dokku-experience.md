---
layout: post
title:  "The Dokku Experience"
date:   2018-04-24 15:41:01 -0400
categories: general
tags: dokku interfaces
---

Dokku has historically had no way to introspect on the state of an installation. At one point in its history, we included a "backup" feature, which allowed users to export - and _maybe_ import - configuration and data. The challenge is in exposing this information in an easily parseable manner.


### Plumbing vs Porcelain

To understand the challenges better, it is important to note that the internal representation of the state of Dokku is _very_ different from how users interact with it. Data is stored in either `/home/dokku/APP` or `/var/lib/dokku`. Plugins may also store data on another host, and some implementations of the config plugin actually store configuration in distributed datastores. Dokku interacts with this data via a series of plugins - written in various languages - through `plugn`. This hairy mess is Dokku's Plumbing.

Externally, developers use the `dokku` cli to orchestrate their applications. This is a well-known interface with clear documentation and usage examples. The cli rarely changes drastically, only doing so to allow for new functionality in a way that does not break existing use. The output format also rarely changes in backwards incompatible ways. If you know how to use one Dokku installation from 2 years ago, a modern install will be extremely familiar to you. This nice interface is called the Porcelain.

The analogy is thus: most folks using a restroom are well-acquainted with a toilet. Every toilet is similar, allowing for differences in color, size, and features - a bidet could be nice, as could an auto-flush feature. Very few folks know anything about the pipes that move waste and water around, only caring when it's broken or needs updating to support newer toilet features. The important thing is that you know how to use the toilet, that it does what you expect, and that you don't need to re-learn how to use it.

### A Common Interface

When you use Dokku, you'll notice a few things:

- Plugins almost always show help with no specified subcommand.
- Config is down via `:set`, and exposed via `:report`.
- The primary object being manipulated always comes first, so `APP` is commonly the first argument.

If you use Dokku, you only need to learn those patterns once. It is easy to figure out what is available, and straightforward to introspect upon the state of the system. This translates into the following:

- Easy decisions around how to expose new functionality (how did we do it the last time?)
- Decreased support headaches from users getting acquainted with the system (fewer folks hunting for the answer)
- Delightful experiences for our users!

One thing I didn't touch upon is that our output is more or less machine parseable. It is not in json/xml/toml, but you can easily:

- Hide all extraneous headers and output (`--quiet`)
- Use flags to grab specific bits of output
- Split `:report` output on well-known delimiters (`:`)

The consistency here paves the way for automation.

### Interfacing with Dokku

While we have a cli aimed at humans, as developers, we yearn for interfaces computers can automate. Our next blog post will cover how developers may interact with Dokku in a declarative fashion in order to ensure that their servers and applications are configured as expected.
