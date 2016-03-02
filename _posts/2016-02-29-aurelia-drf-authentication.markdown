---
layout: post
title:  "Authentication using JWT with Aurelia and Django"
date:   2016-02-29 16:00:00 +0000
categories: development
author: Carles Barrobés
tags: aurelia
---

This is the first article in a series that will describe some of
the various ways we have been using Aurelia, our Javascript
framework of choice.

Today's topic will be how we handle login to Ployst.


## A Tiny Tech Overview of Ployst UI

Just to give some background to the rest of this article, the Ployst
web UI is implemented as an SPA using [Aurelia][aurelia] - a forward-thinking
Javascript framework - and [Django][django] and [ReST framework][drf] as the
API backend (this API backend is the same Ployst public API that third parties
can use).


## JWT tokens

[JWT (JSON Web Tokens)][jwt] is the sensible choice nowadays for implementing
authentication between an SPA web application and the backend. It is
a general solution for "representing claims securely between two parties",
- which means it is not just limited to our scenario depicted above, but it's
a good fit for it.

JSON web tokens are well supported in multiple languages and frameworks, which
simplifies implementation. In our case, we could use out of the box solutions:

- [Aurelia-auth][au-auth] on the client side
- [Django Rest Framework JWT][drf-jwt] on the server side

Both libraries include well documented examples on how to set them up, so it
shouldn't be much hassle for you to get up and running. The only thing you
need to make sure is that you use consistent settings (i.e. the same *prefix*
for the HTTP `Authorization` header, as it has different defaults in both
libraries - `Bearer` in `aurelia-auth` and `JWT` in `djangorestframework_jwt`).

We won't go into much detail here about JWT and when you should use them,
you can check [JWT.io][jwt] should you want to dive
deeper. As a few key points for our use case:

- Tokens are self-contained: the token itself can be used to encode information
  about the user (like username and email)
- Tokens are secure: the data is protected cryptographically to guarantee
  origin
- Tokens can contain an expiry date as part of the payload
- Tokens can be passed along across different services for delegated access

So a token is an *attached digital signature*, meaning that it contains both
the data and the signature. It means you do not need to keep a record
of issued tokens or session information in the server for authentication
purposes.

Things we include in the token payload:

- Username
- Expiry date, after which the token is no longer valid
- Refresh expiry date, after which the token can not be renewed any more

A token can be renewed for as long as it hasn't expired. Typically you use:

- Short lived tokens (e.g. one day or some hours)
- Longer renewal cycles (e.g. one week)

When token refresh has expired, the user will need to log in again to start a
new cycle.

We want to make sure that we are not logging our users out too frequently,
while still keeping the tokens short lived, so we automate the process of token
renewal so that it's to some degree transparent to the user.


### The cycle of Tokens


How we use JWT in our browser SPA vs restful backend scenario:

- A user opens up [Ployst][ployst]
- If she has no token yet, we take notice of her intended destination and
  redirect her to a login page
- The user sumbits the login form, which triggers a request to obtain a token
  from the API
- A token is returned and stored in the browser (local storage)
- The user is then directed to the page she wanted to get to

While the user is using the UI, we handle token renewal behind the scenes:

- We periodically check for the token expiration date
- If the token is close enough to expiry, we issue a token refresh request to
  the API. If successful, we replace the token with the new one
- If the token can no longer be renewed, we let it expire
- When the token end of life date is reached, the user is prompted to log in
  again

![The authentication flow](/assets/images/ployst-auth.png)


## Handling Redirect after Login in Aurelia

When a user comes to ployst after a session is expired, and they are thus
redirected to a login page, we want them to be taken back to their desired
destination once login is successful. That behaviour is not supported out of
the box by [Aurelia auth][au-auth], but it's not difficult to implement by
taking advantage of Aurelia's pipeline steps.

Aurelia auth lets you define a fixed route where users will be redirected
after login. The thing we had to do was to make that destination route the
one that handled the post-login redirect.


### The Aurelia pipeline

Pipeline steps in Aurelia are called during the processing of a route. They
would be the front-end equivalents of Django middleware.

Authentication in Aurelia is already handled as a pipeline step. We added an
extra step that runs before authentication, to take care of storing the next
URL.

{% highlight js %}
    config.addPipelineStep('authorize', NextUrl);
    config.addPipelineStep('authorize', AuthorizeStep);
    config.addPipelineStep('authorize', RefreshToken);
    ...
{% endhighlight %}


{% highlight js %}
// next-url.js

export class NextUrl {

    run(routingContext, next) {
        // store current route the user is trying to get to, with some
        // exceptions
        let skip_urls = ['/', '/login-redirect', '/logout'];
        if (routingContext.getAllInstructions().some(i => i.config.auth)) {
            if (skip_urls.indexOf(routingContext.fragment) === -1) {
                this.url = routingContext.fragment;
            }
        }
        return next();
    }

    clear() {
        this.url = undefined;
    }
}
{% endhighlight %}


This is an excerpt of our `aurelia-auth` configuration parameters:

{% highlight js %}
var config = {
    authToken: 'JWT',   // for consistency with our backend expectation

    // backend API
    loginUrl: '/api-token-auth/',

    // front-end app
    loginRoute: '#/login',
    loginRedirect: '#/login-redirect'
};


// aurelia-auth configured within `main.js` as:
export function configure(aurelia) {
    aurelia.use
        .standardConfiguration()
        .plugin('aurelia-auth', baseConfig => {
            baseConfig.configure(config);
        });
}
{% endhighlight %}


And finally these are the relevant bits of our `login-redirect` view model:

{% highlight js %}
// login-redirect.js
import {inject} from 'aurelia-framework';
import {Router} from 'aurelia-router';

import {NextUrl} from '../services/next-url';


@inject(NextUrl, Router)
export class LoginRedirect {

    constructor(nextUrl, router) {
        this.nextUrl = nextUrl;
        this.router = router;
    }

    async activate() {
        let nextUrl = this.nextUrl.url;
        if (nextUrl) {
            this.nextUrl.clear();
            this.router.navigate(nextUrl);
        } else {
            this.router.navigateToRoute('all-the-things');
        }
    }
}
{% endhighlight %}


{% highlight js %}

{% endhighlight %}


[jwt]: http://jwt.io/
[aurelia]: http://aurelia.io
[drf-jwt]: https://github.com/GetBlimp/django-rest-framework-jwt
[drf]: http://www.django-rest-framework.org/
[django]: https://www.djangoproject.com/
[au-auth]: https://github.com/paulvanbladel/aurelia-auth
[ployst]: https://beta.ployst.com
