---
layout: post
title:  "Websockets with Aurelia and Django Channels"
date:   2016-05-31 17:15:53 +0000
categories: development
author: Alex Couper, Carles Barrobés
tags: aurelia django websockets
---

WebSockets are awesome.

In the past, using them in a Django app hasn't been perfect. A
separate server performing the legwork and then everyone reinventing the wheel
to communicate with that server about what to send out and to whom.

And then [Django Channels](https://channels.readthedocs.io) came along and
changed all that.

In this walkthrough, I'll show how we're using Channels at the backend and
where we plug this into Aurelia to give our users timely updates to the
application state.

If you're not familiar with Channels, you may benefit from reading [Channels Concepts ](https://channels.readthedocs.io/en/latest/concepts.html) first.

## Basic config

We use Redis to handle the messaging between interface and worker servers, so
our `CHANNEL_LAYERS` looks like this:

{% highlight python %}
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "asgi_redis.RedisChannelLayer",
        "CONFIG": {
            "hosts": [os.environ.get('REDIS_URL', 'redis://localhost:6379')],
        },
        "ROUTING": "ployst.routing.channel_routing",
    },
}
{% endhighlight %}

## Connecting

In Aurelia, we have a `WebSocketConnection` class that is used to manage the
connection with Django.

{% highlight js %}
@inject(EventAggregator, Authentication)
export class WebSocketConnection {

    constructor(events, authentication) {
        this.connect();
        this.events = events;
        this.events.subscribe(LoggedIn, () => { this.connect(); });
        this.events.subscribe(LoggedOut, () => { this.disconnect(); });
    }
{% endhighlight %}

This sets things up so that a connection lifespan revolves around logging in
and out.

We're using [jwt tokens](http://blog.ployst.com/development/2016/02/29/aurelia-drf-authentication.html)
to authenticate with the backend. This poses a slight problem in Django Channels
land in that we're not using the Django sessions layer so there's no built in
way of knowing which user a connection belongs to.

{% highlight js %}
connect() {
    let token = this.authentication.getToken();
    if ( !token ) {
        return;
    }
    this.socket = new WebSocket(config.wsBaseUrl + '?JWT=' + token);
    this.socket.onmessage = (e) => {this.onmessage(e);};
    this.socket.onclose = (e) => {this.onclose(e);};
    this.socket.onerror = (e) => {this.onerror(e);};
    this.socket.onopen = (e) => {this.onopen(e);};
}
{% endhighlight %}

The `connect()` method passes in the JWT token as part of the WebSocket path
so that the backend can figure out which user this belongs to.

{% highlight python %}
def user_from_jwt_token(func):
    @functools.wraps(func)
    def inner(message, *args, **kwargs):
        key, value = message.content['query_string'].split('=')
        if key == 'JWT':
            token = parse_token(value)
            username = token['username']
            user = User.objects.get(username=username)
            add_to_session(message.channel_session, user)
        return func(message, *args, **kwargs)
    return inner
{% endhighlight %}

The `user_from_jwt_token` decorator performs the leg work of getting the JWT
token out of the query_string and associating the user with the channel_session.

We can then use this decorator around the `websocket.connect` linked method:

{% highlight python %}
# Connected to websocket.connect
@enforce_ordering(slight=True)
@user_from_jwt_token
@channel_session_user
def ws_connect(message):
    log.debug('Connected: %s', message.reply_channel)
    for action in subscription.ON_CONNECT:
        action(message)
{% endhighlight %}


## Handling disconnects

The front end handles disconnects by retrying to open the WebSocket at
increasing intervals.

{% highlight js %}
    incrementBackoff() {
        this.backoff = Math.min(this.MAX_BACKOFF, this.backoff + this.BACKOFF_INCR);
    }

    onclose(e) {
        this.incrementBackoff();
        setTimeout(
            () => {
                if (this.active) this.connect();
            },
            this.backoff * 1000
        );
        this.events.publish(new ConnectionChanged(e));
    }
{% endhighlight %}

And in order to display a message to the user about connectivity status, we
publish events on both open and close of a WebSocket.

{% highlight js %}
    onopen(e) {
        this.backoff = 0;
        this.events.publish(new ConnectionChanged(e));
    }
{% endhighlight %}


## Extra: Production Configuration

We're running on Kubernetes, so we have a set of asgi interface containers
running behind [nginx-ssl-proxy](https://github.com/ployst/docker-nginx-ssl-proxy)
that put requests into Redis.

We have a set of worker containers consuming from Redis and forming
responses.

We've chosen to use the same docker image for both, but we run with different
commands in Kubernetes for each.
