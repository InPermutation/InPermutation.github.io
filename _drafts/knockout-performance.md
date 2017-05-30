---
layout: post
title: Knockout performance
---
[Knockout] is a [Model-View-View Model][mvvm] (MVVM) framework for JavaScript user interfaces.
I've enjoyed using it, both professionally in [FogBugz] and [Kiln], and also when writing a [two-factor authenticator][2fa] for a recent symposium at Fog Creek.

Recently, a customer reported a memory leak in FogBugz.
I was surprised: I expected most memory leaks would be noticed fairly quickly, as we [dogfood] all our software, and our internal FogBugz install is one of the largest instances we manage.
There were two pieces of information that helped me track down the issue.
One is that the leak was reported to happen more quickly when you have lots of browser tabs open with FogBugz.
The other is that this customer uses FogBugz a bit differently than we do.
Instead of having one person hold on to a case until they're done with it, users assign cases back and forth quite often.
This led me to believe our notifications popup, which is present on every page, was responsible for the memory leak.

<h2>Simplified version of existing code</h2>

The popup uses declarative bindings; something like this:

{% highlight html %}
<div id='notification-popup'>
    <div id='count' data-bind='text: count'></div>
    <div id='notifications'
            data-bind='foreach: notification'>
        <a class='notification'
                data-bind='attr: { href: url }'>
            <div class='title' data-bind='text: title'>
            </div>
            <span class='date' data-bind='text: date'></span>
            <div class='text' data-bind='text: body'></div>
            <a data-bind='
                click: toggleRead,
                css: readButtonClass,
                text: readButtonTitle'></a>
        </a>
    </div>
</div>
{% endhighlight %}

As you might guess, the viewmodel is a `NotificationViewModel` with an array of `NotificationContainer`s underneath it. There are some computations to get a `NotificationContainer` to be ready to view, so the actual constructors looked something like this:

{% highlight js %}
function NotificationViewModel() {
    this.notifications = ko.observableArray([]);
    this.count = ko.computed(function() {
        return this.notifications().length;
    }, this);
    this.dateRefresher = ko.observable();
}

var notificationsModel = new NotificationViewModel();

function NotificationContainer(
        isRead, dt, id, header, url, body) {
    this.isRead = ko.observable(isRead);
    this.dt = ko.observable(dt);
    this.id = ko.observable(id);
    this.header = ko.observable(header);
    this.url = ko.observable(url);
    this.body = ko.observable(body);

    this.title = ko.computed(function() {
        return this.id() + ': ' + this.header();
    };
    this.date = ko.computed(function() {
        //this is here so I can tell the dateRefresher
        //to refresh and this function will refresh its output
        //used for when we have something like
        //'n minutes ago' and want to be able to refresh that
        notificationsModel.dateRefresher();
        return getFullFancyDateTime(this.dt());
    }, this);
    this.readButtonClass = ko.computed(function () {
        var base = "change-read-button";
        return this.isRead() ?
            base + " mark-as-unread" :
            base + " mark-as-read";
    }, this);
    this.readButtonTitle = ko.computed(function () {
        return this.isRead() ?
            "Mark as unread" :
            "Mark as read";
    }, this);
}
{% endhighlight %}

A handful of `computed`s, a handful of `observable`s, times the number of notifications on screen.
That shouldn't take much RAM.
But I created a little `curl` loop to generate hundreds of notifications per second, and sure enough the memory usage climbed up and up.
Bingo.

<h2>Problem #1: <tt>computed</tt></h2>
Knockout 3.2.0 introduced [pure computed observables][pureComputed].
The difference between `computed` and `pureComputed` is that a `pureComputed` doesn't maintain subscriptions to its dependencies when it has no subscribers itself.
As the documentation states, this prevents memory leaks, and reduces computation overhead by not re-calculating computed observables whose value isn't being observed.

Every single one of the `computed` properties on this view model can be `pureComputed` instead.
**`%s/.computed/.pureComputed/`**!

This fixes the memory leak.

**Why?** See that `dateRefresher`? Calling that observable creates a subscription from the `NotificationContainer` to the `NotificationViewModel`.
But subscriptions are maintained on the target observable, so this effectively means `NotificationViewModel` has a reference to `NotificationContainer`, and so no `NotificationContainer` can ever be garbage collected, since they're permanently live.

Changing to `pureComputed` means an unloaded `NotificationContainer` binding will release its subscriptions (since it has no downstream subscriptions), and so it is eligible for immediate garbage collection.

<h2>Are we done?</h2>

I mean, yeah, it works, and it no longer leaks memory, so we can quit here if we want.
But the notification popup is on every FogBugz page, so any easy wins would be nice to have while we're in here.

<h2>Problem #2: unnecessary bindings</h2>
My teammate, William Zimrin, successfully fixed some memory bloat on a different Knockout page.
He recommended avoiding bindings that are never changed, or are used for intermediate calculations.

What does that mean?

As [this Stack Overflow answer explains][so11527760], there are 3 benefits to using `pureComputed` over a regular function:

 * the value is memoized
 * you can subscribe to it
 * JSON serialization works

None of these even really apply to us!
We don't have complicated chains of subscriptions; just one or two `pureComputed` functions deep.

Here's what that looks like in practice:

{% highlight js %}
function NotificationViewModel() {
    this.notifications = ko.observableArray([]);
    this.count = ko.computed(function() {
        return this.notifications().length;
    }, this);
    this.dateRefresher = ko.observable();
}

var notificationsModel = new NotificationViewModel();

function NotificationContainer(
        isRead, dt, id, header, url, body) {
    this.isRead = ko.observable(isRead);
    this.dt = ko.observable(dt);
    this.id = ko.observable(id);
    this.header = ko.observable(header);
    this.url = ko.observable(url);
    this.body = ko.observable(body);

    this.title = ko.computed(function() {
        return this.id() + ': ' + this.header();
    };
    this.date = ko.computed(function() {
        //this is here so I can tell the dateRefresher
        //to refresh and this function will refresh its output
        //used for when we have something like
        //'n minutes ago' and want to be able to refresh that
        notificationsModel.dateRefresher();
        return getFullFancyDateTime(this.dt());
    }, this);
    this.readButtonClass = ko.computed(function () {
        var base = "change-read-button";
        return this.isRead() ?
            base + " mark-as-unread" :
            base + " mark-as-read";
    }, this);
    this.readButtonTitle = ko.computed(function () {
        return this.isRead() ?
            "Mark as unread" :
            "Mark as read";
    }, this);
}
{% endhighlight %}

[knockout]: http://knockoutjs.com/
[mvvm]: http://knockoutjs.com/documentation/observables.html#mvvm-and-view-models
[fogbugz]: http://www.fogcreek.com/fogbugz/
[kiln]: http://www.fogcreek.com/fogbugz/devhub "Kiln Code Management and Code Reviews"
[2fa]: https://2fa.glitch.me/
[dogfood]: https://blog.fogcreek.com/video-guide-software-development-essentials/6-dogfooding/
[pureComputed]: http://knockoutjs.com/documentation/computed-pure.html
[so11527760]: https://stackoverflow.com/a/11528135/3140
