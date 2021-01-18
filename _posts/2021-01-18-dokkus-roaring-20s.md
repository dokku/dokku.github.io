---
layout: post
title:  "Dokku's Roaring 0.20s"
date:   2021-01-18 00:13:19 -0400
categories: release
tags: dokku release
---

It's been a few months since the last release post, so we'll summarize whats been going on in Dokku Land in 2020.

> If you're using Dokku - especially for commercial purposes - consider donating to project development via [OpenCollective](https://opencollective.com/dokku) or [Patreon](https://www.patreon.com/dokku). Funds go to general development, support, and infrastructure costs.

## Discussions

Where should you get help? We've added [Github Discussions](https://github.com/dokku/dokku/discussions) support to the main project. Folks are encouraged to post questions and seek help from the community there. This will be a monitored channel, and is subject to the [Code of Conduct](https://github.com/dokku/.github/blob/master/CODE_OF_CONDUCT.md).

If you are seeking more "live" support, join the `#dokku` channel on the [Gliderlabs Slack](https://glider-slackin.herokuapp.com/). A member of the core team - in addition to community members! - will attempt to help you given enough time and information.

Users can continue posting to Stackoverflow or other locations, though those will not be actively monitored for issues.

## Releases

### 0.21.x

We released quite a few major things in the 0.21.0 release:

- Ubuntu 20.04 support: Our packages generally support all releases of Debian and Ubuntu, though having an official package repository for your OS is always nice.
- Go Module usage: Glide was great for the time it was created, but as the Golang community has moved on, so should we.
- Upgrades of herokuish, plugn, procfile-util, sigil, and sshcommand: Dokku occasionally reaches out to single-purpose tooling for certain functionality, and our tight integration means we occasionally need to do things internally to support toolchain upgrades.

Other than the above, 0.21.0 was mostly a bugfix release. Hopefully it was to your liking.

### 0.22.x

The 0.22.x was a bit more expansive in the number of changes it introduced. There were a number of [changes and deprecations](http://dokku.viewdocs.io/dokku/appendices/0.22.0-migration-guide/), the most major of which were:

- Process type names specified in `Procfile` files and app names may no longer use characters not valid in DNS Label Names (RFC 1123). This allows us to properly support networking in alternative schedulers - such as Kubernetes and Nomad - as well as internal app networking with the default Docker Local scheduler.
- The `ps` plugin had it's `*all` commands removed in favor of a `--all` flag. This was actually fairly major, as it changes how parallelism works within Dokku, and makes it easier to support parallel commands against multiple applications. In the future, you'll be seeing more `--all` flag support in Dokku (as well as `--global` as necessary).

What else showed up in 0.22.x?

#### Cloud Native Buildpacks support

[Cloud Native Buildpacks](https://buildpacks.io/) are the future of Dokku's Buildpack support. We previously [blogged about it](/technology/comparing-buildpack-v3-to-herokuish), comparing it to our current Herokuish support. This new initiative is supported by a wide range of folks affiliated with the [Cloud Native Computing Foundation](https://www.cncf.io/), and we're hoping to see tighter integration with Dokku in the future.

While it's still in development, this functionality is currently behind an app-specific environment variable and depends on the `pack` binary. To use it:

```shell
# where $APP is your app name
dokku config:set $APP DOKKU_CNB_EXPERIMENTAL=1

# install pack: https://buildpacks.io/docs/tools/pack/
# apt based installation in use here, but use what you are comfortable with
sudo add-apt-repository ppa:cncf-buildpacks/pack-cli
sudo apt-get update
sudo apt-get install pack-cli
```

Some future improvements:

- `buildpacks` command support
- allowing users to specify CNB usage globally
- builder specification (currently uses `heroku/buildpacks`)
- default to CNB

Definitely a space to watch.

#### Aggressive Container Cleanup

Previous releases of Dokku would leave behind intermediate containers for debugging purposes. As Dokku has largely stabilized, these containers are no longer as necessary. The intermediate containers are commonly confused for containers that utilize server resources. Even worse, Docker will occasionally start these intermediate containers on reboot (Docker does not keep track of whether a container was manually stopped or not), causing actual performance issues for servers.

In 0.22.x, we start deleting these intermediate containers at most 5 minutes after a deploy, and sometimes even sooner. We also cleanup images that may have been used during this process. While the cleanup time is not currently configurable, we hope that this change allows folks to feel better about resource utilization on more constrained servers.

#### Nginx Configuration Knobs

While we attempt to ship with the best nginx configuration by default, folks may want to tune specific knobs to match their setup without needing to ship an `nginx.conf.sigil` that may become outdated. We've added an `nginx:set` command that can be used to set various nginx settings, and will continue to add more settings over time.

Additionally, for installations that have a single `nginx.conf.sigil` that should be used amongst all apps by default, we've added the ability to set the location of the global defaults via a plugin trigger. This is exceptionally useful for platform creators that must inject some special configuration to integrate with their platform.

Finally, if you wish to disable app-side `nginx.conf.sigil` extraction, we've added a tunable property for this via `nginx:set` (`disable-custom-config`). While this cannot currently be set globally or as a default, enterprising platform developers may create plugins that inject properties on app creation...

#### Vector-based Log Shipping

This came late in the 0.22.x lifecycle (0.22.6/0.22.7), but was added to address the logging options issue ([#2268](https://github.com/dokku/dokku/issues/2268)). Vector is an open-source, lightweight and ultra-fast tool for building observability pipelines. Dokku integrates with it for shipping container logs for the `docker-local` scheduler. Users may configure log-shipping on a per-app or global basis, neither of which interfere with the `dokku logs` commands.

Vector is based on the idea of `inputs`, `transforms` and `sinks`. Dokku will automatically inject the correct `inputs` for all Docker containers, and it is up to you to define a `sink` to which you can send container logs (globally or per-app). Dokku does not currently support customizing transforms, though this may come in a future release.

Checkout the [Vector Log Shipping docs](http://dokku.viewdocs.io/dokku/deployment/logs/#vector-logging-shipping) for mor usage information.

#### Golang Plugin Rewrites

A few plugins have been rewritten in Golang:

- app-json
- logs
- ps

Generally speaking, plugins are rewritten when the complexity of writing future features outweighs keeping the plugin implemented in shell code. In particular, any plugins that perform one of the following tasks are good candidates for higher-level languages:

- Argument parsing
- JSON handling (reading and writing)
- Network requests
- Interacting with APIs
- Managing structured data

While all of these have been performed one time or another in bash, the fact is that there are a ton of edge cases that have popped up over the years that make it super difficult to even reason about some of the code, whereas it is fairly easy for me to handle these within Golang (either with the standard library or something off-the-shelf).

All that said, not all plugins will be rewritten in Golang - the `git` plugin is likely to stay as is - and some plugins may see hybrid rewrites as necessary, but this should hopefully give folks a bit of an idea as to why the Dokku codebase is not predominantly written in shell.

## Future Development

0.22.x was a feature-packed release, and we're not slowing down. 0.23.x is around the corner, and will include quite a few interesting features for folks. We'll save those notes for the next blog post :)

As always, please post issues with bugs or functionality you think Dokku might benefit from. As well, feel free to hop onto our [Slack channel](https://glider-slackin.herokuapp.com/) if you have questions, comments, or concerns.

---

If you're using Dokku - especially for commercial purposes - consider donating to project development via [OpenCollective](https://opencollective.com/dokku) or [Patreon](https://www.patreon.com/dokku). Funds go to general development, support, and infrastructure costs.
