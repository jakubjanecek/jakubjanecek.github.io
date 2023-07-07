---
title: 'Scala Server Toolkit – Creating HTTP Server'
date: 2019-12-18T09:00:00-00:00
draft: false
---

> This blog post was originally published at [Avast's Engineering blog](https://engineering.avast.io/scala-server-toolkit-creating-http-server/).

In the previous post I introduced and explained the rationale behind Scala Server Toolkit. Today I would like to continue and get into more detail of
some of the offered modules.

## Getting Started

Our goal is to initialize and start a simple HTTP server using [http4s](https://http4s.org/). There are four basic things you will need in order to
achieve that:

1. Configuration,
1. thread pool (execution context),
1. HTTP handler,
1. and router (specification of which URL should call which handler).

Let’s cover these one by one.

## Configuration

Every application requires some configuration. It is common in Scala world to use Lightbend Config which is a great library to
load [HOCON](https://github.com/lightbend/config/blob/master/HOCON.md) configuration files and retrieve the configuration values by their name and
type. The loaded Config is usually passed around your application components so that each one can retrieve its configuration values. This works well
for small applications but it is not robust for large applications and has its downsides.

The downsides are that loading of configuration values is scattered around all your codebase and it can fail at any time if you make a mistake in the
name or the expected type of the value (it can even fail much later in the lifetime of your app if some components are loaded lazily). Your code is
also tied with the Config library and you cannot easily change it for something else (e.g. in tests).

There is a better approach. Use case classes to model your configuration. Case classes are pure Scala solution (no extra dependency needed), everyone
understands them well, they can model the configuration with specific types (e.g. Port) and they are easy to use from tests. What you need to solve
then is how to fill them with values from a configuration file because in practice you do not want to do that manually. Luckily there are many
libraries for that and the one I would recommend is [PureConfig](https://pureconfig.github.io/).

Scala Server Toolkit already contains a configuration case class for the server: `com.avast.sst.http4s.server.Http4sBlazeServerConfig` in subproject
`sst-http4s-server-blaze` which contains all the possible configuration options for the blaze implementation of http4s server. There is subproject
`sst-http4s-server-blaze-pureconfig` which provides implicit `ConfigReader` instances which you need to import so that PureConfig knows how to fill
the
case class with values from configuration file. And finally there is `sst-pureconfig` which allows you to put these together and actually load the
configuration:

```hocon
listen-address = "localhost"
listen-port = 8080
```

```scala {linenos=table,anchorlinenos=true,lineanchors=c1}
import com.avast.sst.http4s.server.Http4sBlazeServerConfig
import com.avast.sst.http4s.server.pureconfig.implicits._
import com.avast.sst.pureconfig.PureConfigModule
import zio.Task
import zio.interop.catz._

val serverConfig = PureConfigModule.makeOrRaise[Task, Http4sBlazeServerConfig]
```

The import `com.avast.sst.http4s.server.pureconfig.implicits._` is important because it gives you the `ConfigReader` instances for
the `Http4sBlazeServerConfig` case class. The default naming convention is kebab-case which means that field named `configuration-value` in the file
is filled into field named `configurationValue` in the case class. It is also possible to
import `com.avast.sst.http4s.server.pureconfig.implicits.CamelCase._` in which case both the case class field and field in the file have to match.

The import `zio.interop.catz._` is important because it gives you instances for cats-effect typeclasses such as `ContextShift` and `Sync` for
the `Task` effect type of ZIO.

## Thread Pool

Every application also requires some thread pools to execute on. It
is [recommended](https://gist.github.com/djspiewak/46b543800958cf61af6efa8e072bfd5c) to have at least two pools – one bounded for CPU-bound operations
and one unbounded for blocking (IO) operations. Scala Server Toolkit offers subproject `sst-jvm` which contains `ExecutorModule` which gives you
exactly that out of the box. You can also wrap already existing thread pool if you want to reuse it from your “runtime” (
e.g. [ZIO](https://zio.dev/)). There are more methods which allow you to create more thread pools of different kinds and with different configuration.

```scala {linenos=table,anchorlinenos=true,lineanchors=c2}
import cats.effect.Resource
import com.avast.sst.bundle.ZioServerApp
import com.avast.sst.jvm.execution.ExecutorModule
import zio.Task
import zio.interop.catz._

object Application extends ZioServerApp {

  val executorModule = ExecutorModule.makeDefault[Task]

  val fromZIO = ExecutorModule.makeFromExecutionContext[Task](runtime.platform.executor.asEC)

}
```

## HTTP Handler

The next thing you need is an HTTP handler. Simply put, it is just a function from HTTP request to HTTP response. In practice it is a bit more
complicated because such handler usually does side effects so in http4s HTTP handler has the following type
signature: `Request[Task] => Task[Response[Task]]`. http4s provides DSL to easily create a response. See
the [documentation](https://http4s.org/v0.20/dsl) for more information.

```scala {linenos=table,anchorlinenos=true,lineanchors=c3}
import org.http4s.dsl.Http4sDsl
import org.http4s.{Request, Response}
import zio.Task
import zio.interop.catz._

object Handler extends Http4sDsl[Task] {

  def hello(request: Request[Task]): Task[Response[Task]] = Ok("Hello World!")

}
```

## Router

And the last piece is a router. We need to tell http4s server which URLs should be routed to which handlers. It is customary to add a default route
which return `HTTP 404 Not Found`.

```scala {linenos=table,anchorlinenos=true,lineanchors=c4}
import com.avast.sst.http4s.server.Http4sRouting
import org.http4s.dsl.Http4sDsl
import org.http4s.{HttpApp, HttpRoutes}
import zio.Task
import zio.interop.catz._

object Router extends Http4sDsl[Task] {

  val routes = HttpRoutes.of[Task] {
    case request @ GET -> Root / "hello" => Handler.hello(request)
  }

  val router: HttpApp[Task] = Http4sRouting.make(routes)

}
```

HttpRoutes is an object from http4s which allows you to define the actual routing using `Http4sDsl`. You can see that any GET request at
endpoint `/hello` will call our HTTP handler.

Scala Server Toolkit gives you `Http4sRouting` object which allows you to combine multiple `HttpRoutes` objects into one and also adds the
default `HTTP 404 Not Found` route. The result is `HttpApp` which can be passed directly to http4s server.

## Conclusion

Now we have everything we need to start a simple HTTP server from scratch. Note that we need to wrap everything in `cats.effect.Resource` to achieve
resource-safety which is very important. Some of the modules return Resource directly (e.g. `Http4sBlazeServerModule`), some do not and then you need
to use `Resource.liftF` to make the types align.

```scala {linenos=table,anchorlinenos=true,lineanchors=c5}
import cats.effect.Resource
import com.avast.sst.bundle.ZioServerApp
import com.avast.sst.http4s.server.pureconfig.implicits._
import com.avast.sst.http4s.server.{Http4sBlazeServerConfig, Http4sBlazeServerModule, Http4sRouting}
import com.avast.sst.jvm.execution.ExecutorModule
import com.avast.sst.pureconfig.PureConfigModule
import org.http4s.dsl.Http4sDsl
import org.http4s.server.Server
import org.http4s.{HttpRoutes, Request, Response}
import zio.Task
import zio.interop.catz._
import zio.interop.catz.implicits._

object Application extends ZioServerApp with Http4sDsl[Task] {

  private object Handler {

    def hello(request: Request[Task]): Task[Response[Task]] = Ok("Hello World!")

  }

  private val routes = HttpRoutes.of[Task] {
    case request @ GET -> Root / "hello" => Handler.hello(request)
  }

  private val router = Http4sRouting.make(routes)

  override def program: Resource[Task, Server[Task]] = {
    for {
      serverConfig <- Resource.liftF(PureConfigModule.makeOrRaise[Task, Http4sBlazeServerConfig])
      executorModule <- ExecutorModule.makeFromExecutionContext[Task](runtime.platform.executor.asEC)
      server <- Http4sBlazeServerModule.make[Task](serverConfig, router, executorModule.executionContext)
    } yield server
  }

}
```

If you run this application you should see the following output:

```
15:10:53.126 [zio-default-async-1-225493257] INFO org.http4s.server.blaze.BlazeServerBuilder - http4s v0.20.15 on blaze v0.14.11 started at http://[0:0:0:0:0:0:0:0]:8080/
15:10:53.131 [zio-default-async-1-225493257] INFO Application$ - Server started @ 0:0:0:0:0:0:0:0:8080
```

and if you call the endpoint (using [HTTPie](https://httpie.org/)) we created you get the expected HTTP response.

```
> http get http://localhost:8080/hello
HTTP/1.1 200 OK
Content-Length: 12
Content-Type: text/plain; charset=UTF-8
Date: Tue, 10 Dec 2019 14:12:42 GMT

Hello World!
```

And that’s it!

It might be hard to see the benefits of this approach in such a simple example. You could probably write it in less number of lines of code but the
point of all of this is that usually our applications are more complex and are extended over time and only then you start seeing the benefits of
principled approach to type safe configuration, proper resource management and unified approach to component initialization. This approach scales very
well and all the components and their relationships are directly visible. This is what we have found to work best for us at Avast.
