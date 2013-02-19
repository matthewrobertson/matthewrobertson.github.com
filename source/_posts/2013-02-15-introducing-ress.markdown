---
layout: post
title: "Introducing RESS"
date: 2013-02-15 01:45
comments: true
categories: Rails Ruby Javacript Mobile Ress
---

Introducing RESS

As the diversity of internet connected devices increases, the challenge of building web applications that work well across all of them is becoming both more difficult and more important. Currently, there is a shortage of great ideas in the Rails community about how best to tackle this problem. Most developers lean heavily on one of two approaches: creating a responsive design using CSS media queries or pushing a mobile optimized UI based on user agent string detection<!-- more -->.

Responsive design is quickly becoming dogma for the best way to build mobile friendly websites. While, CSS media queries are great for tweaking the layout of an HTML page to better conform to a variety of screen sizes, they fall short of being the one and only tool needed to provide a truly mobile optimized user experience. This is because there are more factors to consider than screen size: less powerful CPUs, slower network connections and touch oriented interfaces all need to be addressed. When all these limitations are taken into consideration, it is obvious that adding more CSS for the user to download and interpret is not a silver bullet solution.

While less popular, server side solutions tend to be more flexible and capable in terms of the device limitations that can be addressed. The approach of pushing an alternate UI based on the user agent string has been demonstrated in a [Railscast episode](http://railscasts.com/episodes/199-mobile-devices) and implemented in the [Mobile Fu gem](https://github.com/brendanlim/mobile-fu). Unfortunately, both of these solutions suffer from some serious flaws. First of all, serving different versions of a web page at the same URL is not RESTful and can have some unwanted consequences (eg caching and SEO). Furthermore, user agent detection can be inaccurate and requires constant maintenance to keep up as new devices and software comes to market. From an implementation perspective, the practice of aliasing a mime type and explicitly setting `request.format` means that templates cannot be shared between the mobile and canonical versions of the site.

## A Better Solution

Clearly server side approaches and responsive design each have their own strengths and weaknesses. So I have come up with an approach that attempts to combine the strengths of both and packaged it up as gem called RESS. The gem provides three major features that you can use while optimizing your application for mobile platforms.

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

As an alternative to redirecting users based on user agent string matching, ress provides a  mechanism to redirect users based on feature detection that is performed on the client side. When a request comes into your site, the javascript included with RESS will parse all of the `[rel="alternate"]` links in your markup, and evaluate their media queries to determine if there is an alternate version available that matches the client. If there is, the user is redirected to the url for that version. The idea for this style of client side redirection (and much of the implementation) was adapted from the [devicejs library](https://github.com/borismus/device.js) by Boris Smus.

## 3. Server side progressive enhancement:

When RESS detects that a request is coming in to one of the alternate versions of your application (via the subdomain), it prepends a path to the list of view paths available for the controller handling the request. Any templates, partials or layouts available in the prepended view path take precedence over those in `app/views` allowing you to overload individual templates for any version of your site. For example, if I wanted to customize the signup form for the `mobile` version of my app I would simply need to create a new partial called `app/mobile_views/users/new.html.erb` with my custom markup. If you do not provide a mobile template, Rails will fall back to `app/views` as normal.

For smaller tweeks RESS also creates view helpers and controller methods for detecting which version of the app the request has come into. eg:

```erb
<% if mobile_request? %>
  <%= image_tag 'low-res.png' %>
<% else %>
  <%= image_tag 'high-res.png' %>
<% end %>
```

## Final Thoughts

RESS may not be a great for a fit for all applications but, I have done my best to implement the features in an a la carte manner so that you can pick and chose what you want. For example, if you don’t want to suffer the performance costs of client side redirects, you can leave the `ress.js` file out of your application and do your own server side user agent redirects, but keep the `ress_annotation_tags` to stay inline with Google’s SEO best practices.

I am keen to hear your thoughts, criticism and feature requests. Feel free to add them to the comments section below, or visit the project [on github](https://github.com/matthewrobertson/ress) and create issues.


## References

- [devicejs](http://www.html5rocks.com/en/mobile/cross-device/) [github](https://github.com/borismus/device.js)
- [Building Smartphone-Optimized Websites](https://developers.google.com/webmasters/smartphone-sites/details)
- [RESS: Responsive Design + Server Side Components](http://www.lukew.com/ff/entry.asp?1392) (inspiration for the name RESS)
- [Response Design Missing the Point](http://bradfrostweb.com/blog/web/responsive-web-design-missing-the-point/)
