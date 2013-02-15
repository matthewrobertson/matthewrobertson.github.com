---
layout: post
title: "Introducing RESS"
date: 2013-02-15 01:45
comments: true
categories: Rails Ruby Javacript Mobile Ress
---

As the diversity of internet connected devices increases, the challenge of building web applications that work well accross all of them is becoming more difficult and more important. Currently, there is a shortage of great ideas in the Rails community about how best to tackle this problem. Most developers lean heavily on one of two approaches: reponsive design or user agent detection.<!-- more -->

## 1. User Agent Detection

[Mobile Fu Gem](https://github.com/brendanlim/mobile-fu)

[in his Railscast episode](http://railscasts.com/episodes/199-mobile-devices)

The problem is that user agent detection is a shady business.

## 2. Media Queries

Media queries are great but sometimes you need to do a little more...

## A Better Solution

After spending some time researching this issue I have come up with a better solution and packaged it up as gem called RESS. The gem provides three major features that you can use while optimizing your application for mobile platforms.

## 1. Annotation of mobile versions of your app:

RESS allows you to specify alternate versions of your rails app. Each alternate version has two mandatory attributes: a subdomain under which the version will be served and a media query that describes the type of clients that should be redirected to that site. Once registered, RESS provides a helper method called `ress_anotation_tags` that should be used to add annotations to the `<head>` of your document that describe where these pages are located and which devices should be redirected to them.

For example, a typical alternate version for a mobile site might include a tag like this:

```html
<link rel="alternate" media="only screen and (max-width: 640px)" href="http://m.example.com/page-1" >
```

The mobile version of the page would also have a link pointing back the canonical version of the app, eg:

```html
<link rel="canonical" href="http://www.example.com/page-1" >
```
These annotations conform to SEO best practices for mobile optimized websites [as documented by Google](https://developers.google.com/webmasters/smartphone-sites/details).

## 2. Client side feature detection and redirects:

As an alternative to redirecting users based on user agent string matching, ress provides a  mechanism to redirect users based on feature detection that is performed on the client side. When a request comes into your site, the javascript included with RESS will parse all of the [rel="alternate"] links in your markup, and evaluate their media queries to determine if there is an alternate version available that matches the client. If there is, the user is redirected to the url for that version.

## 3. Server side progressive enhancement:

When RESS detects that a request is coming in to one of the alternate versions of your application, it prepends a path to the list of view paths available for the controller handling the request. Any templates, partials or layouts available in the prepended view path take precedence over those in `app/views` allowing you to overload individual templates for any version of your site. For example, if I wanted to customize the signup form for the `mobile` version of my app I would simply need to create a new partial called `app/mobile_views/users/new.html.erb` with my custom markup.

By default the prepended view path is derived from the name of the alternate version you registered in the RESS config block, but it can be configured. For details on how this works see the documentation in the [ress.rb intitializer](https://github.com/matthewrobertson/ress/blob/master/lib/generators/ress/templates/ress.rb) that is generated when you install the gem.

## Final Thoughts

Obviously, RESS may not be a great for a fit for all applications. If you think you can get away with a fully responsive design, all the power to you. That said, I have done my best to implement the features in an a la carte manner so that you can pick and chose what you want. For example, if you don’t want to suffer the performance costs of client side redirects, you can leave the `ress.js` file out of your application and do your own server side user agent redirects, but keep the `ress_annotation_tags` to stay inline with Google’s SEO best practices.

I am keen to hear your thoughts, criticism and feature requests. Feel free to add them to the comments section below, or visit the project [on github](https://github.com/matthewrobertson/ress) and create issues.