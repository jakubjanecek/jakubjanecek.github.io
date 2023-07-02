---
title: '"No More Orphans" in Practice'
date: 2023-07-02T09:00:00-00:00
draft: false
---

A few weeks ago, I came across a [tweet from Jakub Kozlowski](https://twitter.com/kubukoz/status/1662047512931311616) that caught my attention.
It linked an older blog article from 7mind called [No More Orphans](https://blog.7mind.io/no-more-orphans.html), which describes a pattern for 
achieving two vital goals in libraries: comfort of use and avoiding dependency hell.

Specifically, I am talking about providing instances of typeclasses, such as JSON serialization. The best approach is to provide these instances 
in the companion objects of your types. This way, there's no need to import anything, and the implicit values are automatically found. However, if you 
want to be a good citizen in the open-source world, you should provide integrations with more than just a single library because there are usually 
multiple options, and everyone prefers something different. This would lead to your library depending on many other libraries, inevitably causing 
dependency hell.

Let's be clear: dependency hell is not something that worries you at the beginning of your project, and it seems reasonable to reuse code from other 
libraries. However, as your project grows and you start depending on more and more libraries, you will eventually encounter a situation where two 
libraries rely on different versions of the same library. This becomes a problem because you can't have both versions on the classpath simultaneously,
and that's when the nightmare begins. You have to choose which library you want to use and somehow replace the other. Or you may want to upgrade your 
project to a new version of a library, only to find yourself blocked because some of the libraries may no longer be maintained.

I believe that avoiding dependency hell is more important than comfort (unfortunately). Therefore, the only solution to this problem is to split your 
project by dependencies and provide typeclass instances separately, requiring an explicit import when using them.

Example of a project structure split by dependencies:

```
- library
- library-circe
- library-upickle
- library-zio-json
```

The article suggests that there is light at the end of the tunnel. It is possible to define your typeclasses in companion objects and make them 
optional, depending on whether a given dependency is available on the classpath. So, can we have our cake and eat it too?

Read the article to understand how it works, but basically, it uses some type gymnastics and the behavior of implicit search to hide implicit 
definitions that don't have their dependencies on the classpath. I was very excited about it because it seemed to have solved a problem I had 
encountered frequently in the past. So, I decided to try it out and implement it in my workplace.

This is the code I had to write to make a dependency on `zio-json` optional:

```scala {linenos=table,anchorlinenos=true,lineanchors=c1}
trait TypeclassHolder {
  type Typeclass[_]
}

trait GetTypeclass[Holder <: TypeclassHolder, ResultTypeclass[_]] {
  implicit def equiv[A]: ResultTypeclass[A] =:= Holder#Typeclass[A]
}

sealed trait ZioJsonCodec extends TypeclassHolder {
  override type Typeclass[A] = zio.json.JsonCodec[A]
}

object ZioJsonCodec {
  @inline implicit final def get: GetTypeclass[ZioJsonCodec, zio.json.JsonCodec] = {
    new GetTypeclass[ZioJsonCodec, zio.json.JsonCodec] {
      override def equiv[A] = implicitly
    }
  }
}

type ZioJsonCodecEvidence[F[_]] = GetTypeclass[ZioJsonCodec, F]
```

It may be a bit verbose, but this piece of code can be defined once and shared by everyone in your company. Now, let's see how it's actually used:

```scala {linenos=table,anchorlinenos=true,lineanchors=c2}
case class ErrorCode(name: String) extends AnyVal

object ErrorCode {
  implicit def errorCodeJsonCodec[F[_]: ZioJsonCodecEvidence]: F[ErrorCode] = DeriveJsonCodec.gen[ApiError].asInstanceOf[F[ErrorCode]]
  // implicit def errorCodeJsonEncoder[F[_]: ZioJsonEncoderEvidence]: F[ErrorCode] = DeriveJsonEncoder.gen[ApiError].asInstanceOf[F[ErrorCode]]
  // implicit def errorCodeJsonDecoder[F[_]: ZioJsonDecoderEvidence]: F[ErrorCode] = DeriveJsonDecoder.gen[ApiError].asInstanceOf[F[ErrorCode]]
}

case class ApiError(message: String, errorCode: ErrorCode)

object ApiError {
  implicit def apiErrorJsonCodec[F[_]: ZioJsonCodecEvidence]: F[ApiError] = DeriveJsonCodec.gen[ApiError].asInstanceOf[F[ApiError]]
}
```

While the code example may seem daunting to someone without prior knowledge, we decided that the technique was worth considering due to its 
comprehensive documentation in the article and the potential for further explanation in ScalaDoc. Unfortunately, we encountered a few issues along 
the way.

We were not pleased with the need for the `asInstanceOf` operation to make the code compile. This can be addressed by employing another [type trick 
documented in accompanying repository](https://github.com/7mind/no-more-orphans/blob/develop/mylib/src/main/scala/mylib/pattern/GetTc.scala#L16). 
However, IntelliJ IDEA still cannot handle this and underlines the code as an error. It was very hard to imagine that everyone would accept this in 
our codebase, considering that IDEA is the most commonly used IDE in our company.

Another problem arose when deriving `JsonCodec`. While `JsonCodec` combines `JsonEncoder` and `JsonDecoder`, in the case of deriving `JsonCodec` for 
`ApiError`, it became necessary to provide separate typeclass instances for `JsonEncoder` and `JsonDecoder` for `ErrorCode`, as shown in the example 
above. This is not necessary when defining `JsonCodec` in a regular manner.

```scala
magnolia: could not find JsonEncoder.Typeclass for type ErrorCode
    in parameter 'errorCode' of product type ApiError

  implicit def apiErrorJsonCodec[F[_]: ZioJsonCodecEvidence]: F[ApiError] = DeriveJsonCodec.gen[ApiError].asInstanceOf[F[ApiError]]
```

Lastly, we realized that there is a potential risk of runtime errors in case of binary incompatibility in one of the libraries. For instance, let's 
assume that `zio-json v1` is used as an optional dependency in your library, and you decide to use an incompatible `zio-json v2` in your project, 
which also relies on that library. Normally, `sbt` would raise an error about the use of different major versions. However, with optional dependencies,
sbt remains silent, and you would encounter a runtime error.

## Conclusion

In the end, I still don't think we can have our cake and eat it too. It all seemed to good to be true. We decided not to adopt the pattern in our 
codebase due to the additional set of problems it introduced. As is often the case, everything involves trade-offs.

## Links
- https://blog.7mind.io/no-more-orphans.html
- https://github.com/7mind/no-more-orphans
