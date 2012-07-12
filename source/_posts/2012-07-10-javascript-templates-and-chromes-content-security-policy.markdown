---
layout: post
title: "Javascript Templates and Chrome's Content Security Policy"
date: 2012-07-10 23:34
comments: true
categories: 
---

Lately, I have been spending a lot of time working on an opensource EPUB3 viewer called [Readium](https://github.com/readium/readium). The project is built entirely with HTML5 / javascript and distributed as a chrome extension. A while ago while testing in [Chrome Canary](https://tools.google.com/dlpage/chromesxs/) I noticed a warning that support for manifest version 1 is being phased out and I should upgrade to version 2. This required a couple very minor modifications to my `manifest.json` (a list of changes is avaiable [here](http://code.google.com/chrome/extensions/manifestVersion.html). But these things are never easy. Sure enough, when I tried to use the extension, everything was busted. I opened the developer console and found the following error message: 

`Uncaught Error: Code generation from strings disallowed for this context`

It turns out the biggest change in manifest version 2 is the addition of a default [conent security policy](http://code.google.com/chrome/extensions/contentSecurityPolicy.html). Two features that have been strictly locked down by this policy are execution of inline javascipt and the evalutation of strings using both `eval()` and `new Function()`. This is a GOOD thing, because it closes down some potential openings for exploits, and you probably shouldn't be doing either of those things anyway. But, if you are using a Javascript templating library you are going to run into problems.

Javascript templates basically work by taking some text content, do some basic lexical parsing to split out the code and html peices and then uses those peices to compile a function that returns an HTML string. John Resig wrote a nice [blog post](http://ejohn.org/blog/javascript-micro-templating/) that goes into a more depth. The problem with Resig's approach (which is very similar to how many client side templating libraries work) is that it uses `new Funtion()` to create a function from a string; an action that is banned by Chrome's CSP.

## The Solution

In a recent [I/O talk](http://www.youtube.com/watch?v=x9KOS1VQgqQ&html5=1) the chrome extions team suggested that one way to use javascript templates in compliance with the new CSP is to create a sandboxed page (in which `eval()` and `new Function` are allowed), load it in an `iframe` and then pass messages back and forth to render templates. They have even created a basic [sample](https://github.com/GoogleChrome/chrome-app-samples/tree/master/eval-in-iframe) that demonstrates how to do it.

But, if the idea of passing messages to a sandboxed iframe has your eyes glazing over, __DON'T PANIC__ there is an easier way: precompile your templates before you deploy your extension. If you peform the `template => function` conversion as part of your build process, you remove the need evade the CSP at run time. Two libraries I am aware of that offer precompilation as a feature are [handlebarsjs](http://handlebarsjs.com/) and [ECO](https://github.com/sstephenson/eco/). I decided to go with handlebars for my project and found it extremely simple to set up. Here are the steps that I had to take:

### 1. Install handlebarsjs

Handlebarsjs is distributed as an npm package, so if you don't have nodejs installed get it from [here](http://nodejs.org/#download). Once you have node you can install handlebarsjs using npm:

`$ npm install handlebars -g`

(the `-g` option gives you global installation so you can use handlebars for any project on your system)

### 2. Put each template into a file of its own.

If you had a set up like me, your templates were all in `<script type="text/template">` tags in your html files. In order to get precompilation working you will need to put each template into its own (uniquely named) file. Each file should have a `.handlebars` extension.
 
### 3. Convert your templates to handlebarsjs tempates

 If you weren't already using handlebarsjs, you will need to convert your templates to use thier format. This will require varying amounts of effort depending on what template library you are coming from. I was using [Underscore's microtemplates](http://underscorejs.org/#template), but I kept the logic in my templates very thin so a global find and replace that swapped `<%=` and `%>` for `{{` and `}}` took care of 90% of my needs. A good overview of how handlebarsjs works is available on [thier site](http://handlebarsjs.com/)

### 4. Compile your templates

You can compile all of your `.handlebars` templates into a single `.js` file in one shot like this:

```$ handlebars path/to/templates/*.handlebars -f path/to/scripts/templates.js```

A more in depth overview of precompilation is available [here](http://handlebarsjs.com/precompilation.html).

### 5. Include some scripts

Before you can start using your templates need to include them in your code. Add a `<script>` tag to our html that includes the `templates.js` file we just generated. You also need to include the handlebars runtime, which you can download from [github](https://github.com/wycats/handlebars.js/archives/master).

### 6. Use 'em

Handlebars makes all of your templates accesable via the global `Handlbars.templates` object. Each template is named according to its file name, so if your input file is `person.handlebars` will be available as `Handlebars.templates.person`.

### Bonus

If you like to over optimize your development environment like I do, you can use this [Guard configuration](https://github.com/aiwilliams/guard-handlebars) to watch your `.handlebars` files and re-compile them automatically whenever you make changes.

## Added Perks

My main motivation for doing this was to move my extension off the depricated standard and have it continue to function, but there are other benefits as well. Having each template in its own file is much tidier, and thus, more maintainable. And, because your clients no longer have to perform the compilation step when loading the page, you get a performance bonus too!

