---
layout: post
title:  "Kubernetes + Letsencrypt"
date:   2015-12-17 09:41:53 +0000
categories: development
tags: kubernetes
---

Below I will describe an approach as to how to get a proxy running that handles
SSL termination and certificate regeneration using letsencrypt.org.

## Problem

You've got a cat naming service, you've containerized it and it is running
happily in kubernetes. It's only serving http, and you want to start securing
connections made.

Here's the state of your cat-name serving API:

[TODO: Image, including pods, services, RCs]

## SSL Termination proxy

Google has provided a [nginx-ssl-proxy][nginx-ssl-proxy-walkthrough]
container that can be configured to route traffic through to any service.

A [very useful walkthrough](http://blog.kubernetes.io/2015/07/strong-simple-ssl-for-kubernetes.html)
is available on the kubernetes blog to get an idea of how this termination works

## Getting a certificate

[Letsencrypt](https://letsencrypt.org/about/) is great to use - it'll give us
a free 90-day certificate for the cat name service, and we can completely
automate fetching new ones.

The verification step is done by expecting a shared secret to be present at
a particular url on the domain being certified.

[TODO: put this in a box or something]

WARNING: There are strict low quotas on the production letsencrypt endpoint,
[use the --server flag][letsencrypt-staging-please] to specify the staging
server during testing or face being locked out for a week.

But for letsencrypt to work with our proxy, we need to be able to:

 - trigger a certificate fetch
 - serve the challenge that letsencrypt gives us (verification)
 - store the certificate somewhere that the proxies will have access to.

## Secrets magic

Kubernetes [secrets](http://kubernetes.io/v1.1/docs/user-guide/secrets.html)
allow sensitive information to be stored outside of your application but
accessible by any containers in the namespace. This is a great way to store
private keys and certificates that will be updated infrequently but read by
potentially many proxies.

The [walkthrough][nginx-ssl-proxy-walkthrough] shows how these can be used with
the nginx-ssl-proxy to provide your proxy with certificates


## A letsencrypt container.

[ployst/letsencrypt](https://hub.docker.com/r/ployst/letsencrypt/) is a docker
container that provides:

 - Monthly certificate/key regeneration
 - Storage of artifacts
 - Restarting of containers that use the certificates + keys.

 Example configuration:

{% highlight yaml %}
---
kind: ReplicationController
apiVersion: v1
metadata:
  name: letsencrypt-rc
  labels:
    name: letsencrypt
    role: cert-app
spec:
  replicas: 1
  selector:
    name: letsencrypt
    role: cert-app
  template:
    metadata:
      name: letsencrypt-rc
      labels:
        name: letsencrypt
        role: cert-app
    spec:
      containers:
      - name: letsencrypt
        image: ployst/letsencrypt:0.0.3
        env:
        - name: EMAIL
          value: foo@bar.com
        - name: DOMAINS
          value: example.com foo.example.com
        - name: RC_NAMES
          value: nginx-ssl-proxy-api
        ports:
        - name: ssl-proxy-http
          containerPort: 80
{% endhighlight %}


This will attempt to get a certificate for `example.com` and `foo.example.com`,
to be registered to `foo@bar.com`.

It will store the result in a secret named
`certs-example.com` with filenames that are usable by the `nginx-ssl-proxy`
container.

It will restart all containers that are owned by the rc named:
`nginx-ssl-proxy-api`

It also supports using the letsencrypt staging endpoint. simply add this to
your yaml env section:

{% highlight yaml %}
- name: LETSENCRYPT_ENDPOINT
  value: https://acme-staging.api.letsencrypt.org/directory
{% endhighlight %}

## A letsencrypt-friendly ssl-terminating proxy

[ployst/nginx-ssl-proxy](https://github.com/ployst/nginx-ssl-proxy) is a fork
of the GoogleCloudPlatform repo that supports an additional env variable:

    CERT_SERVICE

All requests to /.well-known/acme-challenge will be routed through to that
service.

It can be constructed from two additional env variables in the same way that
the TARGET_SERVICE is by the proxy:

    CERT_SERVICE_HOST_ENV_NAME
    CERT_SERVICE_PORT_ENV_NAME

In kubernetes, we can use the env variables that are exposed in all containers
to create the correct configuration. Assuming that you have a service named
'CERTS' pointing to the letsencrypt container:

{% highlight yaml %}
    spec:
      containers:
      - name: nginx-ssl-proxy-api
        image: ployst/nginx-ssl-proxy:0.0.3
        env:
        - name: CERT_SERVICE_HOST_ENV_NAME
          value: CERTS_SERVICE_HOST
        - name: CERT_SERVICE_PORT_ENV_NAME
          value: CERTS_SERVICE_PORT
{% endhighlight %}

## Resulting picture

## Future Changes

### SSL Termination

I'm led to believe that from kubernetes 1.2 the ingress resource will support
https and SSL termination. When this happens, the proxy will probably be
unnecessary.

### Secrets

There is some discussion about changes to secrets propagating through to their
mounted points. When this happens we won't need to restart containers after an
update to a Secret.[TODO: link]

### Cron

Cron is not the long term solution to scheduled jobs in kubernetes. Soon, it
will be possible to schedule jobs to run at particular times in which case the
current cron can be moved into such a job [TODO: discussion link].

[nginx-ssl-proxy-walkthrough]: https://github.com/GoogleCloudPlatform/nginx-ssl-proxy
[letsencrypt-staging-please]: https://community.letsencrypt.org/t/testing-against-the-lets-encrypt-staging-environment/6763
