---
layout: post
title:  "Monadic DI with Reader: interfaces"
description: "Interfaces are foundation of dependency injection. This post explores how Reader monad and interfaces works together."
og_image_url: "/assets/img/elephants-turtle-earth.jpg"
tags: scala, dependency injection, design
visible: true
---
<img src="{{ page.og_image_url }}" align="right" width="50%"/>

 1. [Monadic DI with Reader: is it that good?]({% post_url 2013-09-01-monadi-di-with-reader-is-it-that-good %})
 2. **Monadic DI with Reader: interfaces**

The sole reason I can see that justifies efforts for exploring the Reader-based dependency injection versus a common constructor-based is that monadic DI builds a nice and comfortable *immutable* environment. Software components need not to be initialized and whole wiring of an application logic happens completely out-of-context. And I must admit that this reason is big enough to give it a try.

Let's consider a typical situation in software engineering: some interface, an interface's implementation and a configuration required for this implementation. For convenience sake I'll use a pair of host and port values as the configuration example:

{% highlight scala %}
trait FooConfiguration {
    val hostname: String
    val port: Int
}
{% endhighlight %}

Now the interface:

{% highlight scala %}
trait IFooService {
    def doFiz(a: Int): Reader[FooConfiguration, Int]
    def doBuzz(b: String): Reader[FooConfiguration, String]
}
{% endhighlight %}

Here's the first problem: `Reader` in return type of interface's operations requires a type of configuration as it's first type parameter. `FooConfiguration` is specific to some concrete implementation of `IFooService`. Some other implementation would use a different configuration, e.g. filename. We can dodge this problem by pulling configuration to the trait's type parameter:

{% highlight scala %}
trait IFooService[Conf] {
    def doFiz(a: Int): Reader[Conf, Int]
    def doBuzz(b: String): Reader[Conf, String]
}
{% endhighlight %}

...or using Scala's feature of abstract types:

{% highlight scala %}
trait IFooService {
    type Conf
    
    def doFiz(a: Int): Reader[Conf, Int]
    def doBuzz(b: String): Reader[Conf, String]
}
{% endhighlight %}

I prefer the abstract type approach because it's less intrusive and isolates implementation details better. Type parameters of lower-level interfaces will leak to their clients. Although they carry little of implementation specifics it's still a boilerplate code that no one is happy to see. I can also see ugly [type lambdas](http://stackoverflow.com/questions/8736164/what-are-type-lambdas-in-scala-and-what-are-their-benefits) in some distance at a type parameters direction. Here's an example how things can get bad for a configuration trait of some service on top of `IFooService` and his twin brother:

{% highlight scala %}
trait HighLevelServiceConfiguration[FooConf, BarConf] {
    val foo: IFooService[FooConf]
    val bar: IBarService[BarConf]
}
{% endhighlight %}

Each interface will add at least one type parameter and with an application building layers on top of other layers it will quickly get out of control.

One possible and trivial implementation of `IFooService` can looks like this:

{% highlight scala %}
object SomeFooService extends IFooService {
    // defining the concrete form of configuration required by the service
    trait Conf {
        val hostname: String
        val port: Int
    }
    
    def doFiz(a: Int): Reader[Conf, Int] = Reader { conf =>
        conf.port + a
    }
    
    def doBuzz(b: String): Reader[Conf, String] = Reader { conf =>
        conf.hostname + b
    }
}
{% endhighlight %}

Looks nice so far. There's some boilerplate code in a form of constructing new Reader instances for every interface method but depending on a point of view it can be either tolerated, or fixed by some sort of an implicit conversion. I'm not going to bother with it now, as there's much more troublesome things ahead.

Let's make a step back for a moment and see if different Readers can be transparently combined in same context. Let's assume that there's another interface `IBarService` with the implementation `SomeBarService`:

{% highlight scala %}
trait IBarService {
  type Conf

  def doBar(a: String): Reader[Conf, Int]
}

object SomeBarService extends IBarService {
  trait Conf {
    val filename: String
  }

  def doBar(a: String): Reader[Conf, Int] = Reader { conf =>
    (conf.filename + a).size
  }
}
{% endhighlight %}

Now between `SomeBarService` and `SomeFooService` I can write some nice for-comprehension:

{% highlight scala %}
val program = for {
  fiz <- SomeFooService.doFiz(1)
  bar <- SomeBarService.doBar("one")
} yield fiz + bar

val configuration = new SomeFooService.Conf with SomeBarService.Conf {
  val filename: String = "some_file"
  val hostname: String = "some_host"
  val port: Int = 5432
}

val result: Int = program(configuration)
{% endhighlight %}

Surprisingly enough this code will compile and the type of `program` value would be inferred as `Reader[SomeFooService.Conf with SomeBarService.Conf, Int]`. Now it's something to be both happy and worrying about. A happy part comes from the fact that compiler did it's job perfectly by inferring the correct value type. And worries are because that inferring was possible thanks to Reader's [contravariance](http://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)) on first of it's type arguments. It means that we have to use a class inheritance mechanic to build up the configuration that makes the `program` live and breathe. 

The Scala 2.10 compiler will give you a warning about inferring [existential type](http://en.wikipedia.org/wiki/Existential_type#Existential_types) scalaz.Kleisli and propose to add `import scala.language.existentials` to get rid of this warning. Currently I don't understand how Scalaz managed to infer this type and don't know if this warning something to worry about. Hopefully I'll have some time soon to grok it.

In the next post I'll try to workaround some difficulties that comes from the need to use class inheritance to build up the configuration value. One problem that is particularly important is applying different configurations to the same services depending on some context. Constructor-based dependency injection makes it trivial by creating different stateful instances of services. It doesn't seems to be the case for Reader-based injection.