---
layout: post
title: "Javascript Templates and Chrome's Content Security Policy"
date: 2012-07-10 23:34
comments: true
categories: 
---

Lately, I have been spending a lot of time working on an opensource EPUB3 viewer called [Readium](https://github.com/readium/readium). The project is built entirely with HTML5 / javascript and distributed as a chrome extension. A while ago while testing in [Chrome Canary](https://tools.google.com/dlpage/chromesxs/) I noticed a warning that support for manifest version 1 is being phased out and I should upgrade to version 2. This required a couple very minor changes to my `manifest.json` (a list of changes is avaiable [here](http://code.google.com/chrome/extensions/manifestVersion.html). But these things are never easy. Sure enough, when I tried to use the extension, everything was busted. I opened the developer console and found the following error message: 

`Uncaught Error: Code generation from strings disallowed for this context`

It turns out the biggest change in manifest version two is the addition of default [conent security policy](http://code.google.com/chrome/extensions/contentSecurityPolicy.html).

Javascript templates basically work by taking some text content, do some basic lexical parsing to split out the code bits and then uses them to compile a function that returns an HTML string. John Resig wrote a [blog post](http://ejohn.org/blog/javascript-micro-templating/) that goes into a more depth. The problem with Resig's approach (which is very similar to how most templating libraries work) is that it uses `new Funtion()` to create a function from a string. This is banned by Chrome's CSP.

## The Solution

In an [I/O talk](http://www.youtube.com/watch?v=x9KOS1VQgqQ&html5=1) the chrome extions team suggested that one way to use javascript templates with the new CSP would be to create a sandboxed page (for which `eval()` is allowed), load it in an `iframe` and then pass messages back and forth to create templates. They have even created a [sample](https://github.com/GoogleChrome/chrome-app-samples/tree/master/eval-in-iframe) that demonstrates how to do it.

If the idea of passing messages to a sandboxed iframe has your eyes glazing over, __DON'T PANIC__ there is an easier way: precompile your templates before you deploy your extension. If you peform the `template => function` conversion ahead of time, you remove the need for your users to `eval` strings. Currently the only template library I am aware of that offers precompilation as a feature is [handlebarsjs](http://handlebarsjs.com/) and it is extremely simple to set up. Here are the steps that I had to take:

### 1. Install handlebarsjs

Handlebarsjs is distributed as an npm package, so if you don't have nodejs installed get it from [here](http://nodejs.org/#download). Once you have node you can install handlebarsjs using npm:

`$ npm install handlebars -g`

(the `-g` option gives you global installation so you can use handlebars for any project on your system)

### 2. Put each template into a file of its own.

Had a set up like me, your templates were all in `<script type="text/template">` tags in your html files. To get precompilation working you need to put each template into its own (uniquely named) file.
 

### 3. Convert your templates to handlebarsjs tempates

 If you weren't already using handlebarsjs, you will need to convert your templates to use thier format. This will require varying amounts of effort depending on what template library you are coming from. I was using [Underscore's microtemplates](http://underscorejs.org/#template), but I kept the logic in my templates very thin so a global find and replace that swapped `<%=` and `%>` for `{{` and `}}` took care of 90% of my needs. A good overview of how handlebarsjs works is available on [thier site](http://handlebarsjs.com/)

### 4. Compile your templates

You can compile all of your `.handlebars` templates into one `.js` file in one shot like this:

```$ handlebars path/to/templates/*.handlebars -f path/to/scripts/templates.js```

A more in depth overview of precompilation is available [here](http://handlebarsjs.com/precompilation.html).

### 5. Include some scripts

Before you can start using your templates need to include them in your code. Add a `<script>` tag to our html that includes the `templates.js` file we just generated. You also need to include the handlebars runtime, which you can download from [github](https://github.com/wycats/handlebars.js/archives/master).

### 6. Use 'em

Handlebars makes all of your templates accesable via the global `Handlbars.templates` object. Each template is named according to its file name, so if your input file is `person.handlebars` will be available as `Handlebars.templates.person`.

### Bonus

If you like to over optimize your development environment like I do, you can use this [Guard configuration](https://github.com/aiwilliams/guard-handlebars) to watch your `.handlebars` files and re-compile them automatically whenever you make changes.

## Added Perks

My main motivation for doing this was to move my extension off the depricated standard and have it continue to function but there are other benefits as well. Having each template in its own file is much tidier, and thus, maintainable. And, because your clients no longer have to perform the compilation step when loading the page, you get a performance bonus!

