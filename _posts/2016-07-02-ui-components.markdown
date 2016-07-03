---
layout: post
title:  "How Aurelia helps us with UI componentization"
date:   2016-07-02
categories: development
author: Carles Barrobés
tags: aurelia ui
comments: true
---

We'd like to talk a little about how we've been addressing
working with UI elements using a highly componentised
approach, and how [Aurelia][aurelia] is really helpful to get you into
that kind of thinking.

## UI components

When we talk about a *highly componentised* approach, we
mean how every element that is bound to appear in more
than one place in our UI should be encapsulated into its
own component. And even things that will not appear in
more than one place, but represent a distinct visual
or interface function.

No matter how insignificant one such component may seem,
chances are it will grow and evolve in the future. You want
to keep your codebase DRY, and your code easy to understand.

An illustrative example for us would be the *follow* button
one can use in Ployst to follow/unfollow a stream. Initially
it looked just like a toggleable option that could be
implemented as a checkbox directly as part of a stream
component (e.g. the stream card we use when we display a list
of available streams). But once that code (both presentational
and behavioural) starts getting mixed
up in the stream component, it becomes hard to reuse - we
all know that when in a hurry, some developers may be tempted
to use "reuse by copy paste" and end up with a harder to
maintain codebase, inconsistent presentation etc.

## The Aurelia way

In Aurelia, [custom components][au-components] are very easy
to create, and based on the convention of having two matched
files with the same base name:

- Presentation: HTML markup (e.g. `follow-button.html`) - in Aurelia parlance, the *view*
- Behavior: javascript code (e.g. `follow-button.js`) - Aurelia calls this the *view-model*

Aurelia is really strong on both convention and separation of
concerns. This makes it easy to structure your codebase
cleanly, and there is much to thank for the thought that has
been put into the framework.

One of the key points of Aurelia is that if your component
doesn't (yet) have any behaviour that requires Javascript, you
can start with just the HTML file. This is great because you
can draft all of the presentational aspects of your
functionality first using small specialised components, and add
behaviour later as required.
With this in mind, there is no excuse not to have very small
specialised components. This helps you avoid spaghetti
markup and have everything broken down into much simpler
components.

This, as an example, is our `blip-card.html`, where we display
a *blip* (the basic abstraction for any type of message in
Ployst):

{% highlight html %}
<template>
  <require from="components/avatar.html"></require>
  <require from="blips/blip-appearance-labels.html"></require>
  <require from="blips/blip-children.html"></require>
  <require from="blips/blip-date.html"></require>
  <require from="blips/blip-text"></require>
  <require from="blips/blips-count.html"></require>

  <div class="blip blip-card">
    <blip-appearance-labels if.bind="showAppearances" blip.bind="blip"></blip-appearance-labels>
    <avatar persona.bind="blip.creator" class="pull-left"></avatar>
    <strong>${blip.creator.first_name}</strong> &middot; <blip-date blip.bind="blip"></blip-date>
    <blips-count blip.bind="blip" class="clickable" click.delegate="toggleComments()"></blips-count>
    <blip-text blip.two-way="blip"></blip-text>
    <blip-children blip.bind="blip" if.bind="showComments"></blip>
  </div>

</template>
{% endhighlight %}

The `require` tags at the top represent the components we import
into the page. Notice that in some cases, we import an `html` file,
and in others the extension is missing: in this second case, Aurelia
will load a custom component with both the `.js` and `.html` parts
as outlined earlier.

You can see in the example how easy it is to understand the content
of that component, because the markup is so simple, and has semantic
content: things are named for what they represent (e.g. `blips-count`).
You don't get lost in the details of how these are implemented using
actual HTML tags, classes etc.


## The case of our Follow button

We decided to make the follow button its own component from
the beginning. This was probably the first instance of an element that
was very simple, only used in one place, but had potential for
growing and becoming more pervasive.

Its apperance was naive (a bootstrap button), and the code was simple
enough. These were the off and on states:

![Follow](/assets/images/follow.png)
![Following](/assets/images/following.png)


This was the original code for the view and view-model:

{% highlight html %}
<!-- follow-button.html -->
<template>
    <button type="button" class="btn btn-sm btn-block"
            class.bind="stream.is_subscribed? 'btn-primary':'btn-outline btn-default'"
            click.delegate="click()">
       <i class="fa fa-star"></i>
       ${stream.is_subscribed? 'Following' : 'Follow'}
    </button>
</template>
{% endhighlight %}


{% highlight js %}
// follow-button.js
import {bindable, inject} from 'aurelia-framework';
import {Api} from '../services/api';

@inject(Api)
export class FollowButton{
    @bindable stream = null;

    constructor(api){
        this.api = api;
    }

    click(){
        if (!this.stream.is_subscribed){
            this.api.streamSubscribe(this.stream).then(() => {
                this.stream.is_subscribed = true;
            });
        }
    }
}
{% endhighlight %}


And this is what is needed to use this button:


{% highlight html %}
<require from='components/follow-button'></require>
<!-- ... -->
<follow-button stream.bind="stream"></follow-button>
{% endhighlight %}


Over time, all of the following things happened:

- We started using the button both on a list of available streams
  and on individual stream pages
- We changed the appearance of the button
- We added the number of followers on the side of the button
- We added displaying a list of followers on hover

All of these changes to the button's behaviour and appearance required
no modification of the *using* code, and were fully isolated inside
the component itself.

This is what the button looks like nowadays, when hovered upon:

![Followers](/assets/images/followers.png){: .center-image }


## Conclusion

As a conclusion, my advice to you is to think in terms of components,
no matter how small these seem to be, and your code will be easier
to read, maintain and grow.



[aurelia]: http://aurelia.io
[au-components]: http://aurelia.io/hub.html#/doc/article/aurelia/framework/latest/creating-components/
