---
layout: post
title:  "Cloning a postgres database with docker"
date:   2016-03-31 19:45:53 +0000
categories: development
author: Alex Couper
tags: kubernetes docker postgres
---

#### Introducing [postgres-backup][pg-backup-github]: A docker image that allows us to clone postgres databases for easy testing.

We've recently been working on reorganizing some of the underlying data model
for user accounts and access in Ployst.

This last weekend I wanted to test run the various migrations using (our very
limited) production data.

I wanted to be able to take a postgres data source, dump it and then load it
as the data source of a different container. I could then run the
db migrations against the new container.

Thus [ployst/postgres-backup][pg-backup-github]
(available on [dockerhub][pg-backup-dockerhub]) was created.


## Docker usage

Given two database containers named `api-db` and `api-db-backup`:

{% highlight shell %}
$ docker ps
CONTAINER ID        IMAGE               PORTS                     NAMES
8aa0377b56f9        postgres:9.5.1      0.0.0.0:32813->5432/tcp   api-db-backup
c4a023e46b2a        postgres            0.0.0.0:5432->5432/tcp    api-db
{% endhighlight %}

Running this command would create a backup of `api-db`'s `main` db and load it into
`api-db-backup` as `main2`:

{% highlight shell %}
$ docker run -it --link api-db:db  --link api-db-backup:db2 \
  -e SOURCE_PG_HOST=db -e SOURCE_PG_USER=user1 -e SOURCE_PG_DBNAME=main \
  -e TARGET_PG_HOST=db2 -e TARGET_PG_USER=user1 -e TARGET_PG_DBNAME=main2 \
  ployst/postgres-backup
{% endhighlight %}


## Kubernetes usage

To get a nice workflow for this kind of test in kubernetes, I had to script
the following steps:

 1. Create a new RC owning a postgres db
 2. Run a [job][k8s-jobs] using the [ployst/postgres-backup][pg-backup-github]
    image with env vars that direct it to dump from the source db service and
    load into the target.
 3. Get access to the container

Step 3 could be exposing it externally if you're feeling adventurous (I can't
say I'm brave enough to expose dbs to the public network). We use an
[openvpn image][openvpn-image] to connect to the container network directly.

## Future work

I can foresee the [ployst/postgres-backup][pg-backup-github]
image evolving to be able to use an existing dump (eg if you're OK with last
night's backup).

I'd also like to introduce anonymizing of data into the workflow - though we'll
do that via a new image.

[k8s-jobs]: http://kubernetes.io/docs/user-guide/jobs/
[openvpn-image]: https://github.com/ployst/docker-openvpn-k8s
[pg-backup-dockerhub]: https://hub.docker.com/r/ployst/postgres-backup
[pg-backup-github]: https://github.com/ployst/docker-postgres-backup
[ployst]: https://beta.ployst.com
