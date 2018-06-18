---
layout: post
title:  "Running Sentry with Dokku and turning plumbing to porcelain"
date:   2018-05-01 8:17:00 -0400
author: noah
categories: general
tags: dokku interfaces
---

In a previous post, the amazing Jose Diaz-Gonzalez referred to Dokku as the "porcelain" to Docker's "plumbing". The analogy has been used in software a long time, and refers to this: household sinks (made of porcelain) are easy to use, but the vast network of water infrastructure (plumbing) is too complicated to use otherwise. The sink (or nice hot bath) is a simpler interface to the engineering beneath.

PaaS systems are like this too. You get to deploy "containers" with "add-ons" or "plugins" that enable things like databases, but without the heavy construction effort container systems require (or the government of [Michigan](https://en.wikipedia.org/wiki/Flint_water_crisis) just doesn't care about).

# PaaS sans-SaaS

_OK, Dokku is like other SaaS, but you have to run a shell script_. No, there's more. Heroku's list of [add-ons](https://elements.heroku.com/addons) is long and impressive. But these services _don't actually deploy to a container you control_, so costs can increase and features can be limited by the provider.

Worse yet, your favourite service may not be included yet, you can end up paying for free and open source software. In most cases, that software is easily deployable on Docker; and guess what, Docker is the plumbing that runs Dokku, so nearly all Docker images are automatically equivalent to Heroku's add-ons, but at no cost.

# Lets set up Sentry, but for free

[Sentry](https://sentry.io/welcome/) is a great error-tracking tool. Think Google Analytics, but it onlyl send data to your (very pretty) dashboard when an uncaptured error is thrown. From there, you can send the details of the error to Trello, Jira, Github Issues or wherever. It's very powerful and yet so easy to understand and use. And it's open source with official Docker images, so it's ready to go on Dokku. Here's how.

I provide direct commands and links to the official, latest Sentry Docker below, but if you want to know how to repeat this process and turn any Docker into and add-on, navigate to either https://store.docker.com or https://hub.docker.com (Docker is migrating over to the `store.` sub-domain currently). Search for whatever service you're interested in; but right now we're after the [official Sentry Docker](https://hub.docker.com/_/sentry/) page.

Sentry needs both Postgres and Redis, we can tell that from the page. Good news, Dokku's amazing maintainers have made Postgres and Redis plugins that work with Sentry, if you follow my directions.

We find these official plugins at [http://dokku.viewdocs.io/dokku/community/plugins/](http://dokku.viewdocs.io/dokku/community/plugins/). From there, click the top [Postgres plugin](https://github.com/dokku/dokku-postgres-plugin) and [Redis plugin](https://github.com/dokku/dokku-redis-plugin) links. In short, do these on your Dokku-running box:

```
sudo dokku plugin:install https://github.com/dokku/dokku-redis.git redis
sudo dokku plugin:install https://github.com/dokku/dokku-postgres.git postgres
```

Yepp, we can use the official Dokku database plugins for this Docker version of Sentry that probably wan't made with Dokku in mind. It just takes a few extra commands. And the same goes for many other images.

First, lets create from databases for Sentry to use. Commands like `dokku postgres:help` or `dokku redis:help` will explain how to do this, or this blog article will just tell you! Do:

```
dokku postgres:create sentry
dokku redis:create sentry
```

Great, we have a Postgres and Redis database both named sentry, but we still have no Sentry. Time to do some apprenticeship-level "plumbing" and install the Sentry Docker image. [This Dokku documentation section](http://dokku.viewdocs.io/dokku/deployment/methods/images/#image-workflows) explains the steps:

```
dokku apps:create sentry
sudo docker pull sentry:latest
sudo docker tag sentry:latest dokku/sentry:latest
```

There's a last step you can try now, or wait till I explain why it probably won't work: `dokku tags:deploy sentry`. First, we need to do a few porcelain _and_ plumbing things. First, the porcelain:

```
dokku postgres:link sentry sentry
dokku redis:link sentry sentry
```

This tells Dokku's plugin scripts to link the databases we named `sentry` with the app we named `sentry`. The names might seem confusingly similar right now. Name them `sentryredis` or `sentrydb` if you like, but you'll see once we look at the plumbing they are easy to tell apart.

Dokku does something smart, it makes an environment variable called `DATABASE_URL` and `REDIS_URL` after we run both these commands. Take a look by running `dokku config sentry`:

```
=====> sentry env vars
DATABASE_URL:  postgres://postgres:b4e230f75d28c687fa6a152ba0907d1e@dokku-postgres-sentry:5432/sentry
REDIS_URL:     redis://sentry:b4d0351c5dc873a28f9f3ef5fd8dbfe6729ae7bd19baeafe7d52f1d70d10658f@dokku-redis-sentry:6379
```

These variables can be used in non-Docker apps, like `node` apps, like this `connectToPostgres(process.env.DATABASE_URL)`. But unfortunately not all Docker images look for a `DATABASE_URL` or `REDIS_URL`, so how can we connect them?

# How to connect Dokku plugins to Dokku-tagged Docker images

Using this Sentry image as an example, let's look at the top of the Docker Hub/Store page and see the link titled [`(8.22/Dockerfile)`](https://github.com/getsentry/docker-sentry/blob/b7d55c572e0b23c5c537432211823a8677727e1c/8.22/onbuild/Dockerfile)

![](../img/posts/sentry-tutorial/docker-github-link.png)

From here, navigate up one folder to [this folder](https://github.com/getsentry/docker-sentry/tree/master/8.22). The file that helped me figure out what `ENV` variables the app really wanted where in [`sentry.conf.py`](https://github.com/getsentry/docker-sentry/blob/master/8.22/sentry.conf.py). The important ones (as I'll explain why) are:

```
SENTRY_DB_NAME
SENTRY_REDIS_PASSWORD
```

Not only does `postgres:link` add `DATABASE_URL` (which includes username and password), but it triggers Docker's standard `--link` function to add a few more variables that we don't actually see when we run `dokku config sentry`. Want to know how to see what these secret variables are? Run:

```
sudo docker run --rm --name ilovedokku --link dokku.postgres.sentry:postgres --link dokku.redis.sentry:redis busybox env
```

You'll see an output like this (whic I've shortened to show the relevant keys):

```
REDIS_PORT_6379_TCP_ADDR=172.17.0.8
POSTGRES_PORT_5432_TCP_ADDR=172.17.0.6
POSTGRES_ENV_POSTGRES_PASSWORD=b4e230f75d28c687fa6a152ba0907d1e
```

The default [`sentry.conf.py`](https://github.com/getsentry/docker-sentry/blob/master/8.22/sentry.conf.py) I showed above looks for these variables! Great! And the rest of the information it fills in with defaults that work with Dokku/Postgres/Docker's defaults as well. It's just `SENTRY_DB_NAME`
and `SENTRY_REDIS_PASSWORD` that seem to be neglected by whoever made this Docker image, Docker itself, Dokku, or [whoever brought computers into this world](https://en.wikipedia.org/wiki/Ada_Lovelace).

But don't worry. Now with this in mind we head back (but don't follow yet) [the official instructions](https://github.com/docker-library/docs/tree/master/sentry#how-to-use-this-image)

```
1. Start a Redis container // already did that, skip
...
2. Start a Postgres container // already did that too
...
```

Step 3: This is to make a secret key for Sentry to use internally. We just add one more step, so do below:

```
dokku config:set --no-restart sentry SENTRY_SECRET_KEY=$(sudo docker run --rm sentry config generate-secret-key)
--OR--
sudo docker run --rm sentry config generate-secret-key
# then copy the output and do
dokku config:set --no-restart sentry SENTRY_SECRET_KEY=555&uh&uq+326nml+1f7i&t+rzkp427i15)0isuii&urqbv4lf
````

4. Now an important part: the docs will have us run commands with `--link` and assume names link `sentry-postgres` and `sentry-redis`. If you run `sudo docker ps` you'll see our `postgres` and `redis` containers are named like `dokku.postgres.sentry` and `dokku.redis.sentry`. So we have to substitute those names.

But remeber I mentioned `SENTRY_DB_NAME` and `SENTRY_REDIS_PASSWORD`? They aren't included either by Dokku or by Docker's `--link` command, _and_ the defaults in the config file aren't theones we have. So, we need to do `dokku config sentry`, extract that info, and making matching environment variables like so:

```
=====> sentry env vars
DATABASE_URL:       postgres://postgres:b4e230f75d28c687fa6a152ba0907d1e@dokku-postgres-sentry:5432/sentry
# SENTRY_DB_NAME is the last part, 'sentry'
REDIS_URL:          redis://sentry:b4d0351c5dc873a28f9f3ef5fd8dbfe6729ae7bd19baeafe7d52f1d70d10658f@dokku-redis-sentry:6379
# SENTRY_REDIS_PASSWORD is the long bit 'b4d0351c5dc873a28f9f3ef5fd8dbfe6729ae7bd19baeafe7d52f1d70d10658f'
SENTRY_SECRET_KEY:  t2i7=!+qy(gfp4fbotjk24mob_h+k6ysykgypxr7o3kbo*w4!^
```

Now simply set these missing variables the Dokku way:

```
dokku config:set --no-restart sentry SENTRY_REDIS_PASSWORD=b4d0351c5dc873a28f9f3ef5fd8dbfe6729ae7bd19baeafe7d52f1d70d10658f SENTRY_DB_NAME=sentry
```

4. The Sentry set-up requires a special database "upgrade" step. Before we do this, we should not that Dokku sets up things slightly incorrectly for this use case when we did `dokku postgres:link sentry sentry`. We can fix this with `dokku docker-options:remove ...` and `dokku docker-options:add ...`. But here's another way; do `sudo nano /home/dokku/sentry/DOCKER_OPTIONS_RUN` edit the file as below, then do the same in `sudo nano /home/dokku/sentry/DOCKER_OPTIONS_DEPLOY`, ignoring any other lines.

```
--link dokku.postgres.sentry:postgres
--link dokku.redis.sentry:redis
```

These probably said `--link dokku.postgres.sentry:dokku-postgres-sentry` or something, just make them say `...:postres` and `...:redis` as above. Docker will name those variables based on what comes after the `:`. That part is called the `alias`. If we did `--link dokku.postgres.sentry:ilovedokku` will make environment variables like `ILOVEDOKKU_PORT_6379_TCP_ADDR=172.17.0.8`, which I'm sad to say, does not work with this Sentry Docker. Now lets do

```
dokku --rm run sentry upgrade
```

OR

```
sudo docker run -it --rm -e SENTRY_SECRET_KEY=$(dokku config:get sentry SENTRY_SECRET_KEY) -e SENTRY_DB_NAME=$(dokku config:get sentry SENTRY_DB_NAME) -e SENTRY_REDIS_PASSWORD=$(dokku config:get sentry SENTRY_REDIS_DB_PASSWORD) --link dokku.postgres.sentry:postgres --link dokku.redis.sentry:redis dokku/sentry upgrade
```

You'll be asked to make an admin account name and details, which you'll use to loging to Sentry shortly.

# Final steps - setting up cron and worker(s)

Most Docker images I've deployed on my Dokku rigs have not required this extra step. So hopefully it's a worthwhile guide for these special exceptions. We need to run two special commands, which we can do two ways:

```
dokku --detach run sentry run cron
dokku --detach run sentry run worker
```

# Take-aways

There are many great Dokku plugins on the [Dokku plugins](http://dokku.viewdocs.io/dokku/community/plugins/) page, and they can be made to work with other Docker images; not just `Herokuish` deploys. It just takes an understanding of what environment variables `--link` creates, and a peak at the config files of the particular `Dockerfile` and config files of that repo. Some commands to keep in mind:

```
# show the running containers, use the Image name to --link
sudo docker ps
# show what variables are made by a particular --link
sudo docker run --rm --name dokkudaisuki --link dokkku.postgres.myappname:postgres busybox env
```