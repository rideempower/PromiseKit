# FAQ

## Why should I use PromiseKit over X-Promises-Foo?

* PromiseKit has a heavy focus on **developer-experience**. You’re a developer, do you care about your experience? Yes? Then pick PromiseKit.
* Do you care about having any bugs you find fixed? Then pick PromiseKit.
* Do you care about having your input heard and reacted to in a fast fashion? Then pick PromiseKit.
* Do you want a library that has been maintained continuously and passionately for 6 years? Then pick PromiseKit.
* Do you want a library that the community has chosen to be their №1 Promises/Futures library? Then pick PromiseKit.
* Do you want to be able to use Promises with Apple’s SDKs rather than have to do all the work of writing the Promise implementations yourself? Then pick PromiseKit.
* Do you want to be able to use Promises with Swift 3.x, Swift 4.x, ObjC, iOS, tvOS, watchOS, macOS, Android & Linux? Then pick PromiseKit.
* PromiseKit verifies its correctness by testing against the entire [Promises/A+ test suite](https://github.com/promises-aplus/promises-tests).

## Do I need to worry about retain cycles?

Generally, no. Once a promise completes, all handlers are released and so
any references to `self` are also released.

However, if your chain contains side effects that you would typically
not want to happen after, say, a view controller is popped, then you should still
use `weak self` (and check for `self == nil`) to prevent any such side effects.

*However*, in our experience most things that developers consider side effects that
should be protected against are in fact *not* side effects.

Side effects include changes to global application state. They *do not* include
changing the view of a viewController. So, protect against setting UserDefaults or
modifying the application database, and don't bother protecting against changing
the text in a `UILabel`.

[This stackoverflow question](https://stackoverflow.com/questions/39281214/should-i-use-weak-self-in-promisekit-blocks)
has some good discussion on this topic.

## Do I need to retain my promises?

No, every promise handler retains its promise until the handler is executed. Once
all handlers are executed the promise is deallocated. So you only need to retain
the promise if you need to reference its final value after its chain has completed.

## Where should I put my `catch`?

`catch` deliberately terminates the chain. You should put it low in your promise
hierarchy at a point as close to the root as possible. Typically, this would be 
somewhere such as a view controller, where your `catch` can then display a message
to the user.

This means you should be writing one catch for many `then`s and returning
promises that do not have internal `catch` handlers of their own.

This is obviously a guideline; do what is necessary.

## How do branched chains work?

Suppose you have a promise:

```
let promise = foo()
```

And you call `then` twice:

```
promise.then {
    // branch A
}

promise.then {
    // branch B
}
```

You now have a branched chain. When `promise` resolves, both chains receive its
value. However, the two chains are entirely separate and Swift will prompt you
to ensure that both have `catch` handlers.

You can most likely ignore the `catch` for one of these branches, but be careful:
in these situations, Swift cannot help you ensure that your chains are error-handled.

```
promise.then {
    // branch A
}.catch { error in
    //…
}

_ = promise.then {
    print("foo")
    
    // ignoring errors here as print cannot error and we handle errors above
}
```

It may be safer to recombine the two branches into a single chain again:

```
let p1 = promise.then {
    // branch A
}

let p2 = promise.then {
    // branch B
}

when(fulfilled: p1, p2).catch { error in
    //…
}
```

> It's worth noting that you can add multiple `catch` handlers to a promise, too.
> And indeed, both will be called if the chain is rejected.

## Is PromiseKit “heavy”?

No. PromiseKit contains hardly any source code. In fact, it is quite lightweight. Any
“weight” relative to other promise implementations derives from 6 years of bug fixes
and tuning, from the fact that we have *stellar* Objective-C-to-Swift bridging, and 
from important things such as [Zalgo prevention](http://blog.izs.me/post/59142742143/designing-apis-for-asynchrony)
that hobby-project implementations don’t consider.

## Why is debugging hard?

Because promises always execute via `dispatch`, the backtraces you get have less
information than is often required to trace the path of execution.

One solution is to turn off dispatch during debugging:

```swift
// Swift
DispatchQueue.default = zalgo

//ObjC
PMKSetDefaultDispatchQueue(zalgo)
```

Don’t leave this on. In normal use, we always dispatch to avoid you accidentally writing
a common bug pattern: http://blog.izs.me/post/59142742143/designing-apis-for-asynchrony

## Where is `all()`?

Some promise libraries provide `all` for awaiting multiple results. We call this function
`when`, but it is the same thing. We chose `when` because it's the more common term and
because we think it reads better in code.

## How can I test APIs that return promises?

You need to use `XCTestExpectation`.

We also define `wait()` and `hang()`. Use them if you must, but be careful because they
block the current thread!

## Is PromiseKit thread-safe?

Yes, entirely.

However the code *you* write in your `then`s might not be!

Just make sure you don’t access state outside the chain from concurrent queues.
By default, PromiseKit handlers run on the `main` thread, which is serial, so
you typically won't have to worry about this.

## Why are there separate classes for Objective-C and Swift?

`Promise<T>` is generic and and so cannot be represented by Objective-C.

## Does PromiseKit conform to Promises/A+?

Yes. We have tests that prove this.

## How do PromiseKit and RxSwift differ?

https://github.com/mxcl/PromiseKit/issues/484

## Why can’t I return from a catch like I can in Javascript?

Swift demands that functions have one purpose. Thus, we have two error handlers:

* `catch`: ends the chain and handles errors
* `recover`: attempts to recover from errors in a chain

You want `recover`.

## When do promises “start”?

Often people are confused about when Promises “start”. Is it immediately? Is it
later? Is it when you call then?

The answer is: promises do not choose when the underlying task they represent
starts. That is up to that task. For example here is the code for a simple
promise that wraps Alamofire:


```swift
func foo() -> Promise<Any>
    return Promise { seal in
        Alamofire.request(rq).responseJSON { rsp in
            seal.resolve(rsp.value, rsp.error)
        }
    }
}
```

Who chooses when this promise starts? The answer is: Alamofire does and in this
case, it “starts” immediately when `foo()` is called.

## What is a good way to use Firebase with PromiseKit

There is no good way to use Firebase with PromiseKit. See the next question for
a more detailed rationale.

The best option is to embed your chain in your Firebase handler:

```
foo.observe(.value) { snapshot in
    firstly {
        bar(with: snapshot)
    }.then {
        baz()
    }.then {
        baffle()
    }.catch {
        //…
    }
}
```


## I need my `then` to fire multiple times

Then we’re afraid you cannot use PromiseKit for that event. Promises only
resolve *once*. This is the fundamental nature of promises and is considered a
feature since it gives you guarantees about the flow of your chains.


## How do I change the default queues that handlers run on?

You can change the values of `PromiseKit.conf.Q`. There are two variables that
change the default queues that the two kinds of handler run on. A typical
pattern is to change all your `then`-type handlers to run on a background queue
and to have all your “finalizers” run on the main queue:

```
PromiseKit.conf.Q.map = .global()
PromiseKit.conf.Q.return = .main  //NOTE this is the default
```

Be very careful about setting either of these queues to `nil`.  It has the
effect of running *immediately*, and this is not what you usually want to do in
your application.  This is, however, useful when you are running specs and want
your promises to resolve immediately. (This is basically the same idea as "stubbing"
an HTTP request.)

```swift
// in your test suite setup code
PromiseKit.conf.Q.map = nil
PromiseKit.conf.Q.return = nil
```

## How do I use PromiseKit on the server side?

If your server framework requires that the main queue remain unused (e.g., Kitura),
then you must use PromiseKit 6 and you must tell PromiseKit not to dispatch to the
main queue by default. This is easy enough:

```swift
PromiseKit.conf.Q = (map: DispatchQueue.global(), return: DispatchQueue.global())
```

Here’s a more complete example:

```swift
import Foundation
import HeliumLogger
import Kitura
import LoggerAPI
import PromiseKit

HeliumLogger.use(.info)

PromiseKit.conf.Q = (map: DispatchQueue.global(), return: DispatchQueue.global())

let router = Router()
router.get("/") { _, response, next in
    Log.info("Request received")
    after(seconds: 1.0).done {
        Log.info("Sending response")
        response.send("OK")
        next()
    }
}

Log.info("Starting server")
Kitura.addHTTPServer(onPort: 8888, with: router)
Kitura.run()
```

## My question was not answered

[Please open a ticket](https://github.com/mxcl/PromiseKit/issues/new).