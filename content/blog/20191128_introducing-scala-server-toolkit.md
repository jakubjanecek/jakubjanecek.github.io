---
title: 'Introducing Scala Server Toolkit'
date: 2019-11-28T09:00:00-00:00
draft: false
---

> This blog post was originally published at [Avast's Engineering blog](https://engineering.avast.io/introducing-scala-server-toolkit/).

Today I would like to introduce a new project called [Scala Server Toolkit](https://github.com/avast/scala-server-toolkit). In short it is culmination
of years of Scala server development at Avast representing the best practices we have gained. We decided we want to share it with everyone because we
believe it could be very useful not just to us. Let me describe to you the genesis of this project, the rationale behind it and show you what it
offers.

## Genesis & Rationale

Avast backend developers have been using Scala since around 2011. Over the years they have went through the usual stages of a Scala developer:

- Using Scala as better Java: imperative code full of mutability and side effects.
- Idiomatic Scala: expression-oriented programming, full potential of Scala collections, convenient mix of OOP and FP.
- FP Scala: taming side effects, monads and algebraic laws, all about referential transparency, composability and separation of concerns.

The evolution process from imperative Scala towards FP Scala was very natural. As we have been learning Scala more and more it lead us to the more
functional side of the language. Now we are in a phase in which we are dedicated to go “full FP”.

However not everything was rose-colored. The different styles of programming we have used in the past mean that we have very diverse codebases and
teams/people are used to do things differently. Functional programming is very popular nowadays and the amount of innovation in that space is
incredible but we are still lacking some kind of library that would give us unified approach to programming server applications. Thus we have decided
to write our own. It should help us unify our programming style, promote best practices, functional programming and we believe it could solve the same
problems for others too.

Basically the problem we are trying to solve is that there are lots of great open source libraries which we want to use but putting them all together
in unified way and teaching all of that to newcomers is quite some task. And we think this should not be so much difficult.

It should be pointed out that we do not intend to write some full-blown framework that would try to solve all your server-side problems. We are
actually trying to do quite the opposite. Reuse as most open-source libraries as we can and provide only the missing integration pieces to simplify
everyday programming. **Most of the code revolves around initialization and integration of existing OSS libraries in unified way.**

## Design

There are certain design decisions and constraints that we put in place to guide the development of this library.

- **Modular** design: small, cohesive, orthogonal and composable components.
    - The project is split into completely separate modules mostly based on dependencies. Every module provides some functionality, usually something
      small. The idea is to compose multiple modules together to get a working application but be able to replace any module with different
      implementation in case it is needed.
- Keep the **number of dependencies as low** as possible.
    - Adding a dependency seems like a great idea to avoid having to implement some piece of existing logic however every dependency is also a burden
      that can cause a lot of problems. Therefore every dependency should be considered well and only very important and well-maintained libraries
      should be added to our core.
- **Functional programming.**
    - We believe in functional programming so this library also utilizes its concepts.
    - We do not want to force anyone into any specific effect data type so the code is written in so-called tagless final style.
- Type safe configuration and **resource lifecycle**.
    - Application initialization is often overlooked. By using type safe configuration and proper resource management (to never leak resources) we can
      make it correct without much hassle.
- No need for dependency injection.
    - Dependency injection is not needed in most cases – plain constructors and good application architecture usually suffice.
    - Use a dependency injection framework if your application is huge and it justifies its cost.
- Strive for [Scalazzi Safe Scala Subset](https://slides.yowconference.com/yowwest2014/Morris-ParametricityTypesDocumentationCodeReadability.pdf).

## Why Choose It?

I assume you are a Scala developer doing functional programming who wants to build a server application using [http4s](https://http4s.org/)
and [ZIO](https://zio.dev/). But not only that you want to load your configuration into case classes ([PureConfig](https://pureconfig.github.io/)),
you want to [manage the resources](https://typelevel.org/cats-effect/datatypes/resource.html) and of course you want to [monitor the health of your
application via metrics](https://micrometer.io/).

As you can see from the links in the above paragraph there are great tools for all of this but how do you combine them? There is a huge space for
error because each library does things a bit differently. For example Http4s server cannot be configured via case class so you have to use the
provided builder and fetch the configuration values from somewhere manually. Loading of configuration via PureConfig is side-effectful so you have to
wrap it in `F` manually. Micrometer is a Java library so it does not provide Resource for initialization.

All of these things are small warts that need to be handled in each and every server application which is tedious. Scala Server Toolkit tries to
abstract away the differences and gives you **unified interface to initialization** of components. Interface that is **pure**, **resource-safe** and *
*composable**.

## Example

How does it look in practice? I am going to show you the minimal example on how to start some HTTP server. Keep in mind that “minimal” is not about
the number of lines. You do have to write some code to put everything together but that is fine because you only have to write that code once in the
lifetime of your application and all the dependencies between components are explicitly visible in your code – there is no hidden magic.

```scala {linenos=table,anchorlinenos=true,lineanchors=c1}
import cats.effect.Resource
import com.avast.sst.bundle.ZioServerApp
import com.avast.sst.http4s.server.pureconfig.implicits._
import com.avast.sst.http4s.server.{Http4sBlazeServerConfig, Http4sBlazeServerModule, Http4sRouting}
import com.avast.sst.jvm.execution.ExecutorModule
import com.avast.sst.pureconfig.PureConfigModule
import org.http4s.dsl.Http4sDsl
import org.http4s.server.Server
import org.http4s.{HttpApp, HttpRoutes}
import zio.Task
import zio.interop.catz._
import zio.interop.catz.implicits._

object ServerApp extends ZioServerApp with Http4sDsl[Task] {

  private val router: HttpApp[Task] = Http4sRouting.make {
    HttpRoutes.of[Task] {
      case GET -> Root / "hello" => Ok("Hello World!")
    }
  }

  def program: Resource[Task, Server[Task]] = {
    for {
      serverConfig <- Resource.liftF(PureConfigModule.makeOrRaise[Task, Http4sBlazeServerConfig])
      executorModule <- ExecutorModule.makeFromExecutionContext[Task](runtime.platform.executor.asEC)
      server <- Http4sBlazeServerModule.make[Task](serverConfig, router, executorModule.executionContext)
    } yield server
  }
}
```

This is just a short example. There will be follow-up articles explaining the individual parts of the library in more detail. If you want to find out
more now you can look at the [documentation](https://github.com/avast/scala-server-toolkit/blob/master/docs/index.md) (it is compiled so it is always
up-to-date) or an [example](https://github.com/avast/scala-server-toolkit/tree/master/example/src/main/scala/com/avast/sst/example) server application
in the repository.

## Conclusion

Scala Server Toolkit is currently going through rapid development phase since we are at the beginning of the project. That being said it is already
being used in some of our projects. It still requires some work before we can release the first stable version but I encourage everyone to try it out
and let us know what they think. The first stable version is coming in the following months.
