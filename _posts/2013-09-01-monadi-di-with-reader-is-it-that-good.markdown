---
layout: post
title:  "Monadic DI with Reader: is it that good?"
description: "This post will start my blog and also a series of thoughts about dependency injection in Scala. I want to explore this topic because I'm still confused about it even after spending multiple days digging into."
og_image_url: "http://blog.korbakov.com/assets/img/di-solder.jpg"
tags: scala, dependency injection
---
<img src="/assets/img/di-solder.jpg" align="right" width="50%"/>
This post will start my blog and also a series of thoughts about dependency injection in Scala. I want to explore this topic because I'm still confused about it even after spending multiple days digging into.

There is no lack of blog posts, slides and presentations that explain the idea of doing dependency injection in Scala through Reader monad. Here're just some of them for the reference:

 * [Dead-Simple Dependency Injection][dead-simple-di] by Runar Bjarnason
 * Dependency Injection Without the Gymnastics: [video][di-no-gym-video] and [PDF][di-no-gym-pdf] by Tony Morris & Runar Bjarnason
 * [Using Reader Monad for Dependency Injection][di-on-so] and other StackOverflow questions
 * and many, many more...

At the same time Twitter development team uses and [recommends][twitter-effective-scala] simple and all-familiar constructor-based dependency injection. On other end of the spectrum there's more advanced technique in form of [cake pattern][cake-pattern] with it's own share of well-grounded [critique][cake-pattern-critique].

Constructor-based DI and cake pattern are well described in many sources. But all descriptions of monadic DI approach that I have seen suffer from severe incompleteness. Here're just some problems:

 * dependency injection is treated in very narrow sense of making some objects available in specific contexts
 * configuration is always modeled as single object without any form of separation per component
 * presentations emphasize the transparence on context passing rather its usage inside of functions
 * almost no mentions about additional complexity in form of monad transformers all around

Among these problems the first one is most serious for me. After all, dependency injection isn't some trendy thing that we just must have in our code. The goal of dependency injection is to help with dependencies management of application as it will grow bigger and bigger. There is two ways how dependency injection helps with it: pushing us into direction of decoupling specific implementations from their interfaces and improving visibility into how components are interconnected. Without paying attention to application structure it's absolutely possible and in fact very easy to get poorly structured code with spaghetti-like dependencies using any DI framework or even without one.
From visibility standpoint, any person caring for keeping application complexity under control would like to understand few things about the code easily:

   * how many dependencies each specific application component have?
   * how many components are depending on some specific component?
   * how does the graph of dependencies looks like? Any loops? Is it tree-like or web-like?

Reader dependency injection in form that's described in aforementioned sources makes it very difficult to answer these questions. In fact it's impossible to understand what are real dependencies of some component without going through it's code looking for all references of configuration passed through Reader. Global configuration accessible from every place developer wants makes [Service Locator anti-pattern][sl-anti-pattern], which is nothing I would like to see in my application.

I had a blitzkrieg plan to build a simple Reader-based solution for DI that would give better visibility into how application components are connected. Pretty soon, I realized that this task is far from the trivial one. I'm not done yet, but merely trying I've got some good insights I'm going to share in my following posts. Stay tuned!

[dead-simple-di]: http://www.youtube.com/watch?v=ZasXwtTRkio
[di-no-gym-video]: http://vimeo.com/44502327
[di-no-gym-pdf]: http://phillyemergingtech.com/2012/system/presentations/di-without-the-gymnastics.pdf
[di-on-so]: http://stackoverflow.com/questions/11276319/using-reader-monad-for-dependency-injection
[twitter-effective-scala]: http://twitter.github.io/effectivescala/#Object%20oriented%20programming-Dependency%20injection
[cake-pattern]: http://jonasboner.com/2008/10/06/real-world-scala-dependency-injection-di/
[cake-pattern-critique]: http://igstan.ro/posts/2013-06-08-dependencies-and-modules-in-scala.html
[sl-anti-pattern]: http://igstan.ro/posts/2013-06-08-dependencies-and-modules-in-scala.html