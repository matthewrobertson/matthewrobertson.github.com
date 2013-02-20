---
layout: post
title: "Introducing RESS"
date: 2013-02-15 01:45
comments: true
categories: Rails Ruby Javascript Mobile RESS
---

## TL;DR

I have created [a gem](https://github.com/matthewrobertson/ress) that provides a strategy for building mobile optimized Rails applications by combining the strengths of client-side feature detection with the flexibility of server-side configuration, and respects Google’s best practices for SEO and web-caching.<!-- more -->

## Background

As the diversity of internet-connected devices increases, the challenge of building great web applications that work well on all platforms is becoming more important - and more difficult. Currently, there is a shortage of great ideas in the Rails community about how best to tackle this problem.  Most developers lean heavily on one of two approaches: creating a responsive design using CSS media queries, or serving a custom UI based on user agent string detection. However, mobile optimization is a complex problem that involves configuring many aspects of a site, so it is best not to limit yourself to a purely client side or server side approach.

### Responsive Design

Responsive design is quickly becoming dogma for the best way to build mobile-friendly websites. While CSS media queries are great for tweaking the layout of HTML on different devices, when used alone, they fall short of being the one and only tool needed to provide a truly optimized mobile user experience. This is because there is much more to consider than just screen size. Less powerful CPUs, slower network connections and touch-oriented interfaces all impact the performance and user experience of your site on mobile devices. When all these limitations are taken into consideration, it is obvious that adding more CSS for the user to download and interpret is not a silver bullet.

### Detecting user-agent strings on the server

While less popular, server-side solutions tend to be more flexible and capable of addressing device limitations. The approach of pushing an alternate UI, based on the user agent string, has been demonstrated in a [Railscast episode](http://railscasts.com/episodes/199-mobile-devices) and implemented in the [Mobile Fu gem](https://github.com/brendanlim/mobile-fu). Unfortunately, both of these solutions suffer from some serious flaws.

First, serving different versions of a web page at the same URL is not RESTful and can have unwanted consequences (eg caching and SEO). Furthermore, user agent detection can be inaccurate and requires constant maintenance as new devices and software come to market. From an implementation perspective, the practice of aliasing a mime type and explicitly setting `request.format` makes it [difficult to share templates](https://github.com/rails/rails/issues/3855) between the mobile and canonical versions of a site.

## A Better Solution

Responsive design and server-side approaches each have their own strengths and weaknesses. While working on this problem at [Medeo](https://www.medeo.ca/), I developed an approach that attempts to combine the strengths of both and packaged it as a [gem called RESS](https://github.com/matthewrobertson/ress). This gem provides three major features that you can use to optimize your application for different platforms.

### 1. Annotating mobile versions of your app:

RESS allows you to specify alternate versions of your rails app. Each alternate version has two mandatory attributes: 1) A subdomain under which the version will be served and 2) a media query that describes the type of clients that should be redirected to that site.

Once registered, RESS provides a helper method called `ress_anotation_tags` that can be used to add annotations to the `<head>` of your document. These tags describe where the alternate versions of your site are located and which devices should be redirected to them.

For example, an alternate version of a mobile site might include a tag like this:

```html
<link rel="alternate" media="only screen and (max-width: 640px)" href="http://m.example.com/page-1" >
```

The mobile version of the page would also have a link pointing back the canonical version of the app, eg:

```html
<link rel="canonical" href="http://www.example.com/page-1" >
```
These annotations conform to SEO best practices for mobile optimized websites [as documented by Google](https://developers.google.com/webmasters/smartphone-sites/details).

### 2. Client-side feature detection and redirects:

As an alternative to redirecting users based on user agent string matching, RESS provides a mechanism to redirect users based on client-side feature detection. When a request comes into your site, the javascript included with RESS will parse all of the `[rel="alternate"]` links in your markup and evaluate their associated media queries to determine if there is an alternate version that matches the client. If there is, the user is redirected to the url for that version of your application.

The idea for this style of client-side redirection (and much of the implementation) was adapted from the [devicejs library](https://github.com/borismus/device.js) by Boris Smus.

### 3. Server-side progressive enhancement:

When RESS detects that a request is coming in to one of the alternate versions of your application (via the subdomain), it prepends a path to the list of view paths available for the controller handling the request. Any templates, partials or layouts available in the prepended view path take precedence over those in `app/views`. This allows you to overload individual templates for any version of your site.

For example, if you wanted to customize the signup form for the `mobile` version of your app, you would simply need to create a new partial called `app/mobile_views/users/new.html.erb` with custom markup. If you do not provide a mobile template, Rails will fall back to `app/views`, as normal.

For smaller tweaks, RESS creates view helpers and controller methods for detecting which version of the app the request has come into. eg:

```erb
<% if mobile_request? %>
  <%= image_tag 'low-res.png' %>
<% else %>
  <%= image_tag 'high-res.png' %>
<% end %>
```

## Caveats

RESS may not be a great for a fit for all applications but I have done my best to implement the features in an a la carte manner so that you can pick and chose which are best for you. For example, if you don’t want to suffer the performance costs of client-side redirects, you can leave the `ress.js` file out of your application and do your own server-side user agent redirects. But keep the `ress_annotation_tags` to stay consistent with Google’s SEO best practices.

## Feedback Welcome!

I welcome your thoughts, criticisms and feature requests. Feel free to add them to the comments section below, or visit the project [on github](https://github.com/matthewrobertson/ress) and create issues in the issue tracker.

### References

- [devicejs](http://www.html5rocks.com/en/mobile/cross-device/) [github](https://github.com/borismus/device.js)
- [Building Smartphone-Optimized Websites](https://developers.google.com/webmasters/smartphone-sites/details)
- [RESS: Responsive Design + Server Side Components](http://www.lukew.com/ff/entry.asp?1392) (inspiration for the name RESS)
- [Response Design Missing the Point](http://bradfrostweb.com/blog/web/responsive-web-design-missing-the-point/)