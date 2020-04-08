---
layout: post
title:  "Resource Management in Dokku"
date:   2016-04-19 02:13:01 -0400
categories: features
tags: dokku resource-management
---

Every so often, user's ask if it's possible to use Dokku as the basis of a system where each user in Dokku would have access to *only* their applications. Because of various reasons, this isn't possible out of the box, though it's certainly within the realm of possibility.

There are two requirements for such a system, one of which we'll cover here.

> If you're using Dokku - especially for commercial purposes - consider donating to project development via [OpenCollective](https://opencollective.com/dokku) or [Patreon](https://www.patreon.com/dokku). Funds go to general development, support, and infrastructure costs.

### Resource Management

A common issue you may come up against is how to limit resource usage for different applications. One user's use of memory should not cause OOM issues in another application. Similarly, you would not want a particular application to hog the network bandwidth unfairly, or saturate disk I/O.

These are not new problems to containers, multiple users on a system, or Dokku. These same issues _also_ occur when deploying services directly on a server, or even running applications on your computer. Remember the last time your torrenting application used up all your network and you couldn't load a web page?

It's also a solved problem. Applications that tend to use more bandwidth end up implementing quality of service algorithms to ensure your system runs smooth, allow users access to settings which they can modify to crank up/down resource usage, or some combination of the two. Dokku is no different.

In Dokku's case, we normally provide users with high-level porcelain to handle the low-level plumbing. This usually comes in the form of plugins (porcelain) which orchestrates docker calls (plumbing). However, we move slowly from the plumbing towards porcelain as we judge just _what_ the requirements are and how to best expose a ui around the given problem.

#### Persistent Storage

Take for instance the persistent storage. Dokku has long had plugin hooks, and very specifically implemented the `docker-args` plugin trigger way back in the `0.2.0` era. Gradually that evolved into the current `docker-args-PHASE` triggers, with the following phases:

- `build`: the container that executes the appropriate buildpack
- `deploy`: the container that executes your running/deployed application
- `run`: the container that executes any arbitrary command via `dokku run myapp`

We discovered that while providing plugin triggers is great for tinkerers, it wasn't exactly the nicest way to configure docker options. Many users ended up using a plugin by [Dyson Simmons](https://github.com/dyson), the [unofficial docker-options](https://github.com/dyson/dokku-docker-options) plugin. I even pointed users at it for a while. At some point, we decided to integrate it into the core, and it was implemented in [#1080](https://github.com/dokku/dokku/pull/1080) by [Michael Hobbs](https://github.com/michaelshobbs) and released in [0.3.17](https://github.com/dokku/dokku/blob/master/HISTORY.md#0317).

Even back then, the first comment was ["How do I use this to have persistent storage?"](https://github.com/dokku/dokku/commit/df8f4fb8824550518b07c87ac56aba568bd81295#commitcomment-10907582). In retrospect, yes, this is a *great* feature to have in the core, and the new [docker-options](http://dokku.viewdocs.io/dokku/docker-options/) plugin was a bit too much like shiny plumbing. While the maintainers were distracted with other issues, the hack-fix was to update the documentation to have persistent storage as the example usage.

Dokku implemented this feature in 0.5.0 as the [storage](http://dokku.viewdocs.io/dokku/dokku-storage/) plugin thanks to [Justin Clark](https://github.com/u2mejc/). The interface is a nice piece of porcelain that utilizes the same plugin triggers that the docker-options plugin exposes, except handles the very specific case of attaching persistent storage. It has resulted in many fewer support requests, and I believe has provided developers with a much nicer Dokku experience.

### Where is the resource porcelain?

The Dokku team has yet to see a *nice* interface to limiting resources of the following kind:

- Disk I/O
- RAM usage
- CPU usage
- Network I/O

These are the common resources which dokku users may wish to limit for specific applications, and having a good ui is more important to us than implementing a feature off the cuff.

We also need to consider how such a tool integrates with other Dokku features. The storage plugin works great as a standalone plugin, but it may not be ideal to have a plugin for each type of resource. As well, resource limitation in docker has a [few different implementations](https://gist.github.com/afolarin/15d12a476e40c173bf5f), depending upon what your exact requirements are. Ideally the Dokku solution is a generic one which 80% of our users are happy with, and the 20% that are not can drop down to plugin triggers or the `docker-options` plugin.

At the end of the day, this porcelain is defined by you, our users. Want this feature sooner rather than later? [Submit a pull request](https://github.com/dokku/dokku/pulls) with an implementation, and we'll help shepherd it along to a state where everyone will be happy to use it.

### Alerting on Resource Usage

While dokku manages the lifecycle of application containers, it *does not* and almost certainly *will never* manage monitoring and alerting on that usage. If your application does not have resource limitations in place, or hasn't run a background task in a while, or maybe just isn't running, that is **your** responsibility as a server operator to monitor/correct. Our recommendation here is to send logs/metrics to whatever upstream provider of metrics you prefer. Here are some awesome options:

- [DataDog](https://www.datadoghq.com/): Server and Application Performance Monitoring
- [Dead Man's Snitch](https://deadmanssnitch.com/): Make sure your stuff is still running
- [Logentries](https://logentries.com/): Centralized logging and alerting
- [NewRelic](https://newrelic.com/): Server and Application Performance Monitoring
- [Papertrail](https://papertrailapp.com/): Centralized logging and alerting
- [Pingdom](https://www.pingdom.com/): Make sure your site is responding to requests

As many of our users have never actually maintained a server, we can certainly do more to help push our them in the right direction. In the next few weeks, we will be putting together a document that will gently push our users towards providers that may be able to take care of their needs, as well as clearly delineate where Dokku draws the line in the sand in terms of server management.

{: .center}
[![dokku](/img/dokku.png)](http://dokku.viewdocs.io/dokku/)

---

If you're using Dokku - especially for commercial purposes - consider donating to project development via [OpenCollective](https://opencollective.com/dokku) or [Patreon](https://www.patreon.com/dokku). Funds go to general development, support, and infrastructure costs.
