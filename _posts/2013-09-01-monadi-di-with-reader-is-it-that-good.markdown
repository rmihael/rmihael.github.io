---
layout: post
title:  "Monadic DI with Reader: is it that good?"
description: "This post will start my blog and also a series of thoughts about dependency injection in Scala. I want to explore this topic because I'm still confused about it even after spending multiple days digging into."
og_image_url: "/assets/img/di-solder.jpg"
tags: scala, dependency injection, architecture, design
visible: true
---
<img src="/assets/img/di-solder.jpg" align="right" width="50%"/>

1. **Monadic DI with Reader: is it that good?**
2. [Monadic DI with Reader: interfaces]({% post_url 2013-09-07-monadic-di-with-reader-interfaces %})

This post will start my blog and also a series of thoughts about dependency injection in Scala. I want to explore this topic because I'm still confused about it even after spending multiple days digging into.

There is no lack of blog posts, slides and presentations that explain the idea of doing dependency injection in Scala through Reader monad. Here're just some of them for the reference:

 * [Dead-Simple Dependency Injection](http://www.youtube.com/watch?v=ZasXwtTRkio) by Runar Bjarnason
 * Dependency Injection Without the Gymnastics: [video](http://vimeo.com/44502327) and [PDF](http://phillyemergingtech.com/2012/system/presentations/di-without-the-gymnastics.pdf) by Tony Morris & Runar Bjarnason
 * [Using Reader Monad for Dependency Injection](http://stackoverflow.com/questions/11276319/using-reader-monad-for-dependency-injection) and other StackOverflow questions
 * and many, many more...

At the same time the development team of Twitter uses and [recommends](http://twitter.github.io/effectivescala/#Object%20oriented%20programming-Dependency%20injection) a simple and all-familiar constructor-based dependency injection. On the other hand there's more advanced technique in a form of [cake pattern](http://jonasboner.com/2008/10/06/real-world-scala-dependency-injection-di/) with it's own share of a well-grounded [critique](http://igstan.ro/posts/2013-06-08-dependencies-and-modules-in-scala.html).

The constructor-based DI and the cake pattern are both well described in many sources. But all descriptions of the monadic DI approach that I have seen suffer from severe incompleteness. Here're just some problems:

 * dependency injection is treated in a very narrow sense of making some objects available in specific contexts
 * a configuration is always modeled as a single object without any form of separation per component
 * presentations emphasize on transparent passing of a configuration rather than it's usage inside of functions
 * almost no mentions about an additional complexity in a form of monad transformers all around

Among these problems the first one is most serious for me. After all, dependency injection isn't some trendy thing that we just must have in our code. The goal of dependency injection is to help with dependencies management of an application as it will grow bigger and bigger. There's two ways how dependency injection helps with it: pushing us into decoupling specific implementations from their interfaces and improving visibility into how components are interconnected. Without paying attention to an application structure it's absolutely possible and in fact very easy to end up with a poorly structured code and spaghetti-like dependencies using any DI framework or even without one.
From a visibility standpoint, any person caring for keeping an application complexity under control would like to understand few things about the code easily:

   * how many dependencies each specific application component have?
   * how many components are depending on some specific component?
   * how does a graph of dependencies looks like? Any loops? Is it tree-like or web-like?

Reader dependency injection in a form that's described in aforementioned sources makes it very difficult to answer these questions. In fact it's impossible to understand what are real dependencies of some component without going through it's code looking for all references of configuration passed through Reader. A global configuration accessible from every place whenever a developer wants makes the [Service Locator anti-pattern](http://igstan.ro/posts/2013-06-08-dependencies-and-modules-in-scala.html), which is nothing I would like to see in my application.

I had a blitzkrieg plan to build a simple Reader-based solution for DI that would give better visibility into how application components are connected. Pretty soon, I realized that this task is far from a trivial one. I'm not done yet, but by merely trying I've got some good insights I'm going to share in my following posts. Stay tuned!