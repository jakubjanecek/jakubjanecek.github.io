---
title: 'Forking and Interruption in ZIO'
date: 2024-05-27T09:00:00-00:00
draft: false
---

## Introduction

I recently encountered a bug in my application that caused it to hang during startup. The only change made was upgrading ZIO from version `2.0.x` to
`2.1.0`. After some investigation, I discovered the issue was related to how I was forking fibers and a change in the behavior of `Reloadable`
introduced in the new version. Let me explain what happened because I was very much surprised by the unexpected change. However, in retrospect, it all
makes sense and works correctly; you just need to be aware of it.

## Original Code

This is the original code that was "working" in ZIO `2.0.x`:

<!-- scalafmt:off -->

```scala 3
def layer(config: Config): ZLayer[Any, Throwable, Socket] =
  ZLayer.scoped:
    val zio = for
      _ <- ZIO.logDebug(s"Connecting socket: ${config.host}:${config.port}")
      socket <- makeSocket(config)
      _ <- ZIO.addFinalizer(socket.close *> ZIO.logInfo("Socket closed"))
      hearbeat <- socket.checkHeartbeat(config.heartbeatTimeout).repeat(Schedule.spaced(1.second)).fork
      _ <- ZIO.addFinalizer(hearbeat.interrupt)
    yield socket

    zio.timeoutFail(TimeoutException("Socket initialization took more than 5 seconds!"))(5.seconds)

def reloadable(config: Config): ZLayer[Any, Throwable, Reloadable[Socket]] = Reloadable.manual(layer(config))
```

<!-- scalafmt:on -->

The `layer` function creates a `ZLayer` that initializes a `Socket` and starts a heartbeat check in a separate fiber. The whole layer times out if it
takes more than 5 seconds. The `reloadable` function makes the socket `Reloadable` because we need to be able to restart it (e.g., when the heartbeat
fails).

I put "working" in quotes because the heartbeat was not actually running in this version; I just didn't notice it. The problem was introduced by
adding the `timeoutFail` method to the entire initialization. What happens is that it runs the process in its own fiber and races it against the
specified timeout. That fiber becomes the parent of all the fibers forked inside, and once the process finishes (the socket is initialized), the
parent fiber dies along with all its children. It does not seem very intuitive that adding `timeoutFail` changes the behavior so drastically, but as I
said, it makes sense when you think about it.

## ZIO Upgrade

The new version of ZIO `2.1.0` was recently released, containing this presumably [small change](https://github.com/zio/zio/pull/8638) which
made `acquire` in `ScopedRef` uninterruptible. This change also affects `Reloadable` because it uses `ScopedRef` internally. I noticed
an [immediate fix](https://github.com/zio/zio/releases/tag/v2.1.1) was made in `2.1.1`, which mentions `Reloadable` and hanging forever. This seemed
very similar to my situation, as my application would also hang forever during socket initialization.

It took me a while to debug and understand what was happening. However, knowing how `timeoutFail` works and that acquisition was made uninterruptible
in `Reloadable`, it all clicked. The heartbeat finalizer is run inside `Reloadable` acquisition, thus in an uninterruptible context. So it can't be
interrupted, and it is also being immediately interrupted due to the `timeoutFail` bug I already had. As a result, it all hangs forever.

## Fix

Two things must be done to fix the situation. First, we need to make the heartbeat interruptible. That's easy. Second, we need to fork the heartbeat
in a different scope so that it does not get interrupted by `timeoutFail`. After discussions with my colleagues, I think it's always a good idea to
fork long-running tasks in a specific scope so that the introduction of a combinator like `timeoutFail` can't break your application.

<!-- scalafmt:off -->

```scala 3
def layer(config: Config): ZLayer[Any, Throwable, Socket] =
  ZLayer.scoped:
    val zio = for
      _ <- ZIO.logDebug(s"Connecting socket: ${config.host}:${config.port}")
      socket <- makeSocket(config)
      _ <- ZIO.addFinalizer(socket.close *> ZIO.logInfo("Socket closed"))
      scope <- ZIO.service[Scope]
      hearbeat <- socket.checkHeartbeat(config.heartbeatTimeout).repeat(Schedule.spaced(1.second)).interruptible.forkIn(scope)
    yield socket

    zio.timeoutFail(TimeoutException("Socket initialization took more than 5 seconds!"))(5.seconds)

def reloadable(config: Config): ZLayer[Any, Throwable, Reloadable[Socket]] = Reloadable.manual(layer(config))
```

<!-- scalafmt:on -->

## Conclusion

I hope this post helps someone understand the details of how forking and interruption work in ZIO. It was very puzzling for me at first, but I now
have a better understanding of ZIO and how to use it correctly.

To reiterate the main takeaways:

- Fork long-running tasks in a specific scope to control when they are interrupted.
- Be aware of the interruptibility of your fibers, especially when using `Reloadable`.
