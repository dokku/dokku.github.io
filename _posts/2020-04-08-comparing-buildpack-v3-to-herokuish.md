---
layout: post
title:  "Comparing Cloud Native Buildpacks to Herokuish"
date:   2020-04-08 14:50:00 -0400
categories: technology
tags: dokku buildpacks herokuish
---

An upcoming piece of technology in the container space is Cloud Native Buildpacks (CNB). This is an initiative led by Pivotal and Heroku and contributed to by a wide range of community members, and one that the Dokku project has been following fairly closely. CNB builds upon the buildpack "standard" initially developed at Heroku, modified at Pivotal for Cloud Foundry, and used/abused by the `gliderlabs/herokuish` project. This post goes over a small amount of history, compares buildpack implementations across vendors, and talks about the future of buildpacks as they relate to Dokku.

## History Channel Vault

When Heroku first launched, they provided support for Rack applications, which worked fairly well for the budding Ruby community. As the Ruby community grew, this initial support started to become limiting for their users, and thus the Heroku community started reworking their internal tech to be more flexible in functionality they supported and how they launched processes, In fact, a lot of the fancy features and patterns you'll see in a modern PaaS - 12 factor apps, Procfile support, etc. - were first prototyped or promoted by Heroku. The process of detecting support, building app slugs, and releasing the built artifacts was one such initiative. Eventually, Heroku rebuilt their platform to support alternative programming runtimes as well as community contributed runtimes in what is now known as a buildpacks.

As time marched on, the Cloud Foundry software from Pivotal picked up the buildpack tech. This would seem like an overal great thing for the community - buildpacks that have better support for enterprise environments and needs would certainly help with adoption in the corporate world - but ended up being not as great. As there was no real specification for Buildpack technology, Pivotal ended up diverging from Heroku's implementation, resulting in:

- Buildpack v2a: Heroku-style buildpacks
- Buildpack v2b: Cloud Foundry-style buildpacks

For buildpack authors, the standards were largely compatible - and it would likely be possible to support both at once - but in practice you really only wrote a buildpack for the platform you were using. This also meant that new features for, say, the Python buildpack would need to be implemented twice in order to be supported on these two platforms.

Additionally, the buildpack spec never specified anything about the underlying platform, so building a buildpack for Heroku's platform that depends on some OS-level dependency might not work at all when used with Cloud Foundry or vice versa.

Suffice to say, this divergence isn't great for the community, nor great for pushing the technology forward. I'll skip all the details - mostly because I don't know them! - but the emergence of containers and related technologies enabled the folks at Heroku and Pivotal to combine efforts to on a unified v3 specification, which is now CNB. Think of it as two mini lion bots coming together to form one super-bot!

## Comparing Buildpack tech

While the new specification - still under development - provides a new, unified way to create and distribute buildpacks, there can still be differences between platforms. At this point in time, there are actually two different main "builders" - a collection of buildpacks - that folks in the community can use to play around with CNBs (both are based on the Bionic stack). They do provide slightly different functionality, so a comparison between them seems like a reasonable thing to do. We'll also compare CNBs to `gliderlabs/herokuish`, which is the main OSS implementation of the buildpack v2a technology.

> At the time of writing, Heroku's builder contains v2a buildpacks with a shim to allow them to run under the v3 specification. Additionally, there is a possibility that the organizations will collaborate on buildpacks in the future - who wants to rebuild the wheel? - but this is sort of all in the air. Please keep this in mind if reading this blog post a few months/years from the time of publication.

### Installing dependencies

For local testing, the CNB project provides the `pack` cli tool to simulate what would be available in a platform. There are related projects for Kubernetes and other tools in the deployment space, but `pack` is great for local testing, so we'll want to install that.

```shell
# on a mac with homebrew
brew install buildpacks/tap/pack

# everywhere else, go to the following page and download a release
# https://github.com/buildpacks/pack/releases
```

CNB splits buildpacks such that there is a "build" base image and a "run" base image. Apps are built within the `build` base image and then the layers are rebased onto the `run` image for distribution. This allows for the distributed image to be smaller in size, as well as avoids the need for distribution of compile-time dependencies. Both of these images need to be available on the machine that is running the build process.

```shell
# cloudfoundry
docker image pull cloudfoundry/cnb:latest     # build
docker image pull cloudfoundry/run:full-cnb   # run

# heroku
docker image pull heroku/buildpacks:18        # build
docker image pull heroku/pack:18              # run
```

On the herokuish side, there is a single image that contains both compile and runtime dependencies. You can export a slug from the built image and run that on a system without the installed runtime dependencies, but in practice very few people do so.

```shell
docker image pull gliderlabs/herokuish:latest # build and run
```

Neither Cloud Foundry nor Heroku currently publish a `Dockerfile` for their images, so they are still somewhat of a black box. It's also not super clear from the documentation as to what the `bionic` stack is. We can only hope - but not assume - that these will be published for public recreation in the future.

### Building apps

The app we'll be playing with is the [node-js-getting-started](https://github.com/heroku/node-js-getting-started) app by Heroku. You can play around with your own app to see the results, but Python is currently supported with both builders as well as the `gliderlabs/herokuish` project.

#### With CNB

Building an app with CNB is the same regardless of your chosen builder, which is nice. The following command will build the app in the current directory with the Cloud Foundry builder:

```shell
pack set-default-builder cloudfoundry/cnb:latest
pack build --no-pull --path . "app/nodejs-cloudfoundry:latest"
```

Whereas utilizing the Heroku builder is not much different:

```shell
pack set-default-builder heroku/buildpacks:18
pack build --no-pull --path . "app/nodejs-heroku:latest"
```

Note that there is a volume cache used for dependencies for both buildpacks. At the time of writing, this can be computed like so:

```shell
IMAGE="app/nodejs-cloudfoundry:latest"
CACHE_VOLUME="pack-cache-$(echo -n "index.docker.io/$IMAGE" | sha256sum | cut -c1-12).build"
```

At the time of writing, there isn't a way to clear out that cache volume, so you can use the above method to compute the volume name. Please note that clearing this cache volume does not necessarily clear out any app image layers created, so this may not do exactly as you'd expect if you are interacting with a remote registry.

If you are rebuilding an app using pack, you'll notice that there doesn't appear to be any caching with the Heroku builder. This might be a bug due to shim usage, and will likely be resolved in the future, but for now should expect this to be the case.

> As an aside, the `buildpacks/lifecycle` project - and therefore pack - creates OCI compatible images, so tooling that only works with the older [Docker Image Specification](https://github.com/moby/moby/tree/master/image/spec) may fail when using pack-built images.

#### With Herokuish

Building an image with `gliderlabs/herokuish` is a bit more complicated. While it is distributed as a binary and a docker image, most folks default to using the docker image. Some platforms - notably Gitlab - utilize it via a `Dockerfile`, but this doesn't allow you to take advantage of build cache. The below simulates the patterns used by the `pack` cli tool:

```shell
# use a cache volume
# this can also be substituted with a directory on disk
CACHE_VOLUME="pack-cache-$(echo -n "index.docker.io/$IMAGE" | sha256sum | cut -c1-12).build"
docker volume rm $CACHE_VOLUME >/dev/null 2>&1|| true
docker volume create $CACHE_VOLUME >/dev/null

# run the build process
IMAGE="app/nodejs-herokuish:latest"
docker container run --cidfile /tmp/cid --env USER=herokuishuser -v "${CACHE_VOLUME}:/tmp/cache" -v $PWD:/tmp/app gliderlabs/herokuish:latest /bin/herokuish buildpack build

# create your final image
docker container commit $(cat /tmp/cid) "$IMAGE"

# cleanup
docker container rm $(cat /tmp/cid)
rm -f /tmp/cid 
```

### Running an app

This part is likely the most similar thing across all platforms. Both the CNB images utilize a special `launcher` process to run your process type for you, so all you really need to do is specify that process type.

```shell
# cloudfoundry
docker container run -d -p 5000 --env PORT=5000 --name app.nodejs.cloudfoundry app/nodejs-cloudfoundry:latest web
```

Docker log output on that container looks something like the following:

```
> node-js-getting-started@0.3.0 start /workspace
> node index.js

Listening on 5000
```

Heroku is pretty similar

```shell
# heroku
docker container run -d -p 5000 --env PORT=5000 --name app.nodejs.heroku app/nodejs-heroku:latest web
```

```
Listening on 5000
```

With herokuish, you need to execute the process type with the `/start` prefix. We also need to specify the `USER`, as this will be used by herokuish to properly set file permissions on the `/app` directory.

```shell
docker container run -d -p 5000 --env PORT=5000 --env USER=herokuishuser --name app.nodejs.herokuish app/nodejs-herokuish:latest /start web
```

```
Listening on 5000
```

Startup time is a bit longer with `herokuish` as it will change ownership on files within the `/app` directory.

Another difference between the two is the environment within which both images run in, which we'll get into next.

### Inspecting the built image

Once we have a running app, lets inspect our artifact.

#### With CNB

Executing a command on the CNB-backed containers is the same as executing a command with any other docker image. Processes are automatically invoked within the correct app environment, which is great for one-off commands.

```shell
# cloudfoundry
docker container exec app.nodejs.cloudfoundry id
```

```
uid=2000(vcap) gid=2000(vcap) groups=2000(vcap)
```

```shell
# heroku
docker container exec app.nodejs.heroku id
```

```
uid=1000(heroku) gid=1000(heroku) groups=1000(heroku)
```

Ditto for process inspection:

```shell
# cloudfoundry
docker container exec app.nodejs.cloudfoundry ps auxf
```

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
vcap        35  0.0  0.1  34404  2856 ?        Rs   02:43   0:00 ps auxf
vcap         1  1.2  2.2 739020 45136 ?        Ssl  02:43   0:00 npm
vcap        22  0.0  0.0   4632   828 ?        S    02:43   0:00 sh -c node index.js
vcap        23  0.6  2.0 573156 41896 ?        Sl   02:43   0:00  \_ node index.js
```

```shell
# heroku
docker container exec app.nodejs.heroku ps auxf
```

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
heroku      22  0.0  0.1  34404  2856 ?        Rs   02:47   0:00 ps auxf
heroku       1  0.9  1.7 566792 36728 ?        Ssl  02:46   0:00 node index.js
```

Please bear in mind that PID 1 in the container will depend upon what the buildpack launches. Aside from that note, there is nothing new with CNB and pack in regards to running commands when compared to normal docker operations, so anyone familiar with Docker will feel right at home.

#### With Herokuish

On the herokuish platform, a process must be executed with the `/exec` command as a prefix. This will give you an environment that simulates running a command in the heroku stack.

```shell
docker container exec app.nodejs.herokuish /exec id
```

The output of the above command is:

```
uid=32767(herokuishuser) gid=32767(herokuishuser) groups=32767(herokuishuser)
```

Inspecting processes is pretty similar:

```shell
docker container exec app.nodejs.herokuish /exec ps auxf
```

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root       287  0.0  0.1   5628  2448 ?        Ssl  02:57   0:00 /exec ps auxf
herokui+   298  0.0  0.1  34404  2844 ?        R    02:57   0:00  \_ ps auxf
root         1  0.2  0.1   5628  2448 ?        Ssl  02:57   0:00 /start web
herokui+    13  2.7  1.9 566160 40020 ?        Sl   02:57   0:00 node index.js
```

You'll notice that, in addition to your app, there is also the `/start web` process. This spawns your app with the correct environment, so just a bit different from CNB containers.

### Cleaning up

Removing the containers and images is relatively straightforward, as we use traditional docker commands.

```shell
# cloudfoundry
IMAGE="app/nodejs-cloudfoundry:latest"
CACHE_VOLUME="pack-cache-$(echo -n "index.docker.io/$IMAGE" | sha256sum | cut -c1-12).build"
docker volume rm $CACHE_VOLUME >/dev/null 2>&1|| true
docker container rm -f app.nodejs.cloudfoundry >/dev/null 2>&1|| true
docker image rm cloudfoundry/run:base-cnb cloudfoundry/run:full-cnb cloudfoundry/cnb:latest >/dev/null 2>&1|| true
docker image rm app/nodejs-cloudfoundry:latest >/dev/null 2>&1|| true

# heroku
IMAGE="app/nodejs-heroku:latest"
CACHE_VOLUME="pack-cache-$(echo -n "index.docker.io/$IMAGE" | sha256sum | cut -c1-12).build"
docker volume rm $CACHE_VOLUME >/dev/null 2>&1|| true
docker container rm -f app.nodejs.heroku >/dev/null 2>&1|| true
docker image rm heroku/pack:18 heroku/buildpacks:18 >/dev/null 2>&1|| true
docker image rm app/nodejs-heroku:latest >/dev/null 2>&1|| true

# herokuish
IMAGE="app/nodejs-herokuish:latest"
CACHE_VOLUME="pack-cache-$(echo -n "index.docker.io/$IMAGE" | sha256sum | cut -c1-12).build"
docker volume rm $CACHE_VOLUME >/dev/null 2>&1|| true
docker container rm -f app.nodejs.herokuish >/dev/null 2>&1|| true
docker image rm gliderlabs/herokuish:latest >/dev/null 2>&1|| true
docker image rm app/nodejs-herokuish:latest >/dev/null 2>&1|| true
```

## Dokku and Cloud Native Buildpacks

In an ideal world, all the functionality currently provided by Heroku's v2a Buildpacks and it's ecosystem would immediately exist with Cloud Native Buildpacks. Unfortunately, the spec is still evolving - though nearing a v1! - and the user-base is comparatively small. If you are a buildpack author or a vendor using buildpack technology, now is likely the best time to get involved and raise concerns with the spec and ecosystem.

That said, Dokku has active development towards the addition of CNB to the platform. In our case, we will likely have it as an experimental feature controlled through the use of environment variables, and folks will be able to switch their installations between CNB and Herokuish. This will also likely require changes to certain plugins - in particular, those that interact with the built images - but these will hopefully be fairly minimal.

Long-term, the plan is to deprecate and eventually remove gliderlabs/herokuish support. The purpose of the Herokuish project was to allow folks to emulate the Heroku build/runtime environment. Heroku - amongst others - will be moving to CNB at some point, and already provide a shim for existing buildpacks in the new system. So in theory, any CNB platform would be able to use that shim + any tooling that supports CNB for buildpack support.

The `pack` cli tool already provides a way for folks to build and run applications, and analogs for most herokuish cli commands. Given that:

- `pack` already exists
- Can support existing v2a buildpacks via shim (Heroku Buildpacks)
- Is actively maintained by the CNB folks and a larger community
- Can run inside of a container with a socket mounted, similar to how herokuish runs

The best course of action is for Herokuish to become unsupported once `pack` hits a stable release. It is unlikely that support for Herokuish will continue at that point, and only general maintenance work (upgrading buildpacks) would be performed at that time. Long-term, the project will likely be archived completely in favor of the upstream CNB projects.

There isn't any timeline for the above other than experimental support for CNB in Dokku will land at some point in the near future, so folks installing and using Dokku won't need to worry too much about the process. Long-term, there will be a migration path outlined for users, and how that impacts users will depend on how much tooling is necessary to shim in the existing ecosystem around buildpacks. We hope to keep this minimal, but sometimes you need to break some eggs to make an omelete.

## Cloud Native Buildpacks are the future

The Dokku project is incredibly excited about Cloud Native Buildpacks and it's implications for speeding up how we build secure applications and services. While the existing community is fairly small, the contributors are very dedicated to getting everything just right, and we expect that this will be a great boon to users of both Dokku and buildpacks in general.

If you'd like to have a say in how the CNB initiative develops, please feel free to join in with development or comment in the Slack community - they are a bunch of very friendly folks - where much of the current development is focused.

- Blog: https://medium.com/buildpacks
- Github: https://github.com/buildpacks
- Mailing List: https://lists.cncf.io/g/cncf-buildpacks
- Site: https://buildpacks.io/
- Slack: https://slack.buildpacks.io/
- Twitter: https://twitter.com/buildpacks_io
