---
layout: post
title:  "Welcome to Dokku!"
date:   2016-04-10 23:39:01 -0400
categories: first-prost
tags: dokku update
---

> If you're using Dokku - especially for commercial purposes - consider donating to project development via [OpenCollective](https://opencollective.com/dokku) or [Patreon](https://www.patreon.com/dokku). Funds go to general development, support, and infrastructure costs.

Hi all! The dokku maintainers finally decided it was a good idea to have a blog to post thoughts on the development, evolution, and roadmap of Dokku. Our goal with these posts is to help inform you - dokku users and developers - as to where dokku is headed both internally and externally.

### What is dokku?

> Dokku is a docker-powered PaaS that helps you build and manage the lifecycle of applications.

That is the spiel you get from our Site's metadata, the blurb on github, and even the footer on this blog. But what does it actually mean?

Dokku is a somewhat loose set of scripts that are combined together as a sort of "build" pipeline. The input is your code, and the output is (hopefully) a running application. The pipeline looks something like the following:

```
git push dokku master
# magic
curl -kSso /dev/null -w "%{http_code}" "http://your-app.example.com" | grep 200
```

### What is "magic" in the dokku pipeline?

The "magic" comes from several pieces of tech:

- `sshcommand`: kicks off the proper dokku command on git push.
- `plugn`: allows us to coordinate the stdin/stdout/stderr bits of the pipeline.
- `herokuish`: allows us to emulate - to a very high degree - the inner workings of heroku.
- various scripts that implement the bulk of our feature-set.

All three named pieces of software came from the original creator of Dokku, [Jeff Lindsay][progrium]. Both `sshcommand` and `plugn` are now wholly maintained by the [Dokku Team][dokku-team], while the latter is currently under the care of [Glider Labs][gliderlabs], with maintenance being done by the Dokku Team as necessary.

### What are these "various scripts"?

It's important to note that dokku is by and large composed of shell scripts targeting modern bash. Why?

- Shell is relatively easy to pick up.
- Modern bash is available almost everywhere.
- Interacting with docker was initially only available via shell scripting.

Features are built as "plugins" which are triggered by `plugn`. For example, here are a few different official plugins:

- [config][plugin-configuration-management]: a plugin for managing environment variables.
- [checks][plugin-checks]: a plugin for checking that your application starts properly before bringing it into rotation.
- [storage][plugin-storage]: a persistent storage plugin for use with applications that do not (yet) conform to all 12-factor design requirements.

Plugins can be built in any language - in fact, some prototypes have been written in a hybrid of golang/bash or python. For the foreseeable future, however, we do not envision rewriting the core in another language.

### What is the goal of dokku?

Dokku's goal is to provide a simple, hackable build environment for developers to quickly get their code from their laptops into the cloud. Our personal goal is to make the deployment part easy, so all you have to do is worry about writing code.

{: .center}
[![dokku](/img/dokku.png)](http://dokku.viewdocs.io/dokku/)

[dokku-team]: https://github.com/orgs/dokku/people
[gliderlabs]: https://gliderlabs.com/
[plugin-checks]: http://dokku.viewdocs.io/dokku/checks-examples/
[plugin-configuration-management]: http://dokku.viewdocs.io/dokku/configuration-management/
[plugin-storage]: http://dokku.viewdocs.io/dokku/dokku-storage/
[progrium]: http://progrium.com/blog/

---

If you're using Dokku - especially for commercial purposes - consider donating to project development via [OpenCollective](https://opencollective.com/dokku) or [Patreon](https://www.patreon.com/dokku). Funds go to general development, support, and infrastructure costs.
