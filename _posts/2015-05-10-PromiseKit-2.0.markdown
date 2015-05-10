---
category: news
layout: default
---

# PromiseKit 2.0

Swift was a surprise to all of us, and no more than to the barely-released PromiseKit. I quickly wrote an implementation of Promises in Swift to see how it would feel.
Type-safe promises had their charm, though the compiler was quite difficult to
appease in those pre Swift 1.2 days. However we had two different promise
implementations and they could not bridge.

It was not immediately obvious how this should be solved. Objective-C will never be able to use Swift promises because they are *generic*. But Swift could not (trivially) use our Objective-C promises because they have this syntax:

{% highlight objective-c %}
- (PMKPromise *(^)(id))then;
{% endhighlight %}

Here we have a method that returns a block that takes `id`. This syntax allows
PromiseKit to be delightful in use:

{% highlight objective-c %}
[self wait].then(^{
    return [self fetch];
}).then(^(NSArray *results){
    NSLog(@"%@", results);
})
{% endhighlight %}

Chaining with dot notation is more readable. Taking `id` instead of a specific
block format allows you to pass any block format, and allows us to do cool
tricks like having up to three optional parameters on any `then`.

But Swift was not designed for our edge cases.

## Our Solution

PromiseKit 2.0 has two promise types:

 * `Promise<T>` (Swift)
 * `AnyPromise` (Objective-C)
 
Each is designed to be an approproate promise implementation for the strong points of its language.

`Promise<T>` is strict, defined and precise. `AnyPromise` is loose, flexible and dynamic.

In use `AnyPromise` behaves like `PMKPromise` did:

{% highlight objective-c %}
[NSURLConnection GET:@"http://placekitten.org/%@/%@", width, height].then(^(UIImage *image){

    // PromiseKit determined the returned data was an image by
    // inspecting the HTTP response headers.

    self.imageView.image = image;
});

// or add optional parameters:

[NSURLConnection GET:…].then(^(UIImage *image, NSHTTPURLResponse* rsp, NSData *data){

    // With AnyPromise adding parameters to your then handler is
    // always safe for our NSURLConnection categories, we will
    // detect this and include the NSHTTPURLResponse and the raw
    // NSData response if you do so.
    
    self.imageView.image = image;
});
{% endhighlight %}

`Promise<T>` however is strict. Specify the data type you expect by specializing
your `then`:

{% highlight swift %}
NSURLConnection.GET("http://placekitten.com/\(width)/\(height)").then { (image: UIImage) in
    self.imageView.image = image
}.catch { error in
    // If PromiseKit could not decode an image (because you made
    // a mistake and actually the endpoint provides JSON let’s
    // say). Then you get an error. With AnyPromise you’d get a
    // JSON dictionary and then you would crash when trying to
    // set that to the imageView.image. This is typical to
    // Objective-C.
}

// but if you want just data, ask for just data:

NSURLConnection.GET("http://placekitten.com/\(width)/\(height)").then { (data: NSData) in
    self.imageView.image = UIImage(data: image)
}
{% endhighlight %}


# 2.0 Features

## Bridging Promises

If you have an `AnyPromise` you can use it in a `Promise<T>` chain:

{% highlight swift %}
import PromiseKit.SystemConfiguration

NSURLConnection.POST(url, multipartFormData: formData).then {
    return SCNetworkReachability()
}.then { (obj: AnyObject?) in
    // AnyPromise always resolves with `AnyObject?`
}
{% endhighlight %}

If you need to bridge a `Promise<T>` to Objective-C land, you have to write some  Swift (it is simply impossible for objc to **see** instances
of generic classes so our only option is to bridge via Swift):

{% highlight swift %}
class MyObject {
    func promise() -> Promise<MyObjectResponse> {
        return Promise { fulfill, reject in
            //…
        }
    }
    
    @objc func promise() -> AnyPromise {
        return AnyPromise(bound: promise())
    }
}
{% endhighlight %}
    
## Cancellation

PromiseKit now supports the idea of cancellation (spelled like Apple spell it!).

If the underlying asynchronous task that the promise represents can be cancelled then the author of that promise can register a specific error domain/code pair with PromiseKit and that error will not trigger a catch handler. For example using an alert view:

{% highlight swift %}
let alert = UIAlertView(…)
alert.promise().then {
    // continue
}.catch {
    // this handler won’t execute if the user pressed cancel
}
{% endhighlight %}

This is typically what you want. If an alert is part of a chain then provided
the user taps a button other than cancel, you want to continue; yet if the
user cancels, then you want to abort the chain, but since the user explicitly
cancelled the operation, you don't want to run a catch handler that shows an
error message.

However you can opt-in to catch cancellation. Here’s a more elaborate (and thus
typical example) where cancellation is caught:

{% highlight swift %}
UIApplication.sharedApplication().networkActivityIndicatorVisible = true

NSURLConnection.GET(url).then {
    //…
    return UIAlertView(…).promise()
}.then {
    //…
}.catch(policy: .AllErrors) { error in
    if error.cancelled {
        //…
    } else {
        //…
    }
}.finally {
    UIApplication.sharedApplication().networkActivityIndicatorVisible = false
}
{% endhighlight %}

The PromiseKit categories have all been amended to handle cancellation for types that support it.


## `recover`

To work around ambiguity in `catch` we provide the alternatively named `recover` for `Promise<T>`:
    
{% highlight swift %}
promise.then {
    return CLLocationManager.promise()
}.recover { error -> Promise<CLLocation> in
    if error.domain == kCLErrorDomain && error.code == kCLErrorRestricted {
        return Promise(CLChicagoLocation)
    } else {
        return Promise(error)
    }
}.then { location in
    //…
}
{% endhighlight %}

With `AnyPromise` `catch` itself can be used in this manner:
    
{% highlight objective-c %}
promise.then(^{
    return [CLLocationManager promise];
}).catch(^id(NSError *error) {
    if ([error.domain == kCLErrorDomain && error.code == kCLErrorRestricted) {
        return CLChicagoLocation
    } else {
        return error
    }
}).then(^(CLLocation *location) {
    //…
})
{% endhighlight %}


## `zalgo` and `waldo`

Promises always execute `then` handlers via <abbr title='Grand Central Dispatch'>GCD</abbr> (by default on the main queue). We provide `zalgo` and `waldo` for high performance situations where this slight delay must be avoided.

<center class="big" style="font-size: 1.1rem">
Please note, there are <a href="http://blog.izs.me/post/59142742143/designing-apis-for-asynchrony">excellent reasons</a> why you should never use <code>zalgo</code>. We provide it (mostly) for library authors that use promises. In such situations you should write tests to verify that you have not created possible race conditions.
</center>

Normally a `then` can be dispatched to the queue of your choice:

{% highlight swift %}
let bgq = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)

NSURLConnection.GET(url).then(on: bgq) {
    // we’re on the queue we asked for
}
{% endhighlight %}

Thus to zalgo we:

{% highlight swift %}
NSURLConnection.GET(url).then(on: zalgo) {
    // we will execute on the queue that the previous promise
    // resolved thus consider the queue completely *undefined*
}
{% endhighlight %}

Because the queue we execute on is undefined we provide `waldo`, which will unleash Zalgo unless the queue is the main queue. In that case we dispatch to the default background queue.

Again, **please** don’t use these in a misguided attempt to improve performance: the performance gain is neglible. We provide them for situations when it is imperative that there is minimal delay and for libraries that should be as performant as possible.


# Other Niceties

## `firstly`

Readability can be improved for simple chains if all promises are at the same level of indentation; we provide `firstly`:

{% highlight swift %}
firstly {
    CLLocationManger.promise()
}.then {
    UIView.animate { foo.alpha = 1 }
}.then {
    NSURLConnection.POST(url)
}
{% endhighlight %}

## 100% Test Coverage

PromiseKit 1.x was well tested, including a port of the entire Promises/A+ testsuite. PromiseKit 2.x has tests wherever possible, including testing categories that typically involve user interaction.


## Carthage Support

PromiseKit 1.x supports Carthage too, but you end up having all the categories
compiled in and thus your application links against almost all system frameworks
which is rarely desired. PromiseKit 2’s xcodeproj only builds `CorePromise`. If
you choose to use Carthage you will then have to copy any categories into your
sources in order to use them. Carthage will check out the category sources into
`/Carthage/Checkouts/PromiseKit/Categories` for you.

CocoaPods, as ever, will compile categories into the framework itself, and since
they are all subspecs, you can pick and choose which ones you get. By default CocoaPods will only bundle the `Foundation` and `UIKit` categories.


# Caveats In Use

## `@import`

We worked quite hard to make a single framework that has a different public
interface for objc and Swift. The single caveat to this is that in `.m` files
you must import with the old syntax:

{% highlight objective-c %}
#import <PromiseKit/PromiseKit.h>

// using @import will not break anything, but it will not
// import everything either:

@import PromiseKit;

// so if you must use it, use both:
@import PromiseKit;
#import <PromiseKit/PromiseKit.h>
{% endhighlight %}

## Swift Compiler Issues

The Swift compiler will often error with `then`. To figure out the issue first
try specifying the full signature for your closures:

{% highlight swift %}
foo.then {
    doh()
    return bar()
}

// will need to be written as:

foo.then { obj -> Promise<Type> in
    doh()
    return bar()
}

// because the Swift compiler cannot infer closure types very
// well yet alternatively one line closures almost always
// compile without explicitness. We hate this as it makes
// using promises in Swift ugly, but I sincerely hope that
// Apple intend to improve the detection of closure types to
// make using Promises in Swift as delightful as in objc.

foo.then {
    return bar()
}
{% endhighlight %}

If that doesn’t work then it is probably in fact unhappy about the syntax inside
the closure, but has become confused and is blaming the syntax of your `then`. Move the code out of the closure and try to compile it at the level of a plain
function, when it is fixed, move it back.

If you have further issues, feel free to open a ticket **with a screenshot** of
the error and I can assist. Hopefully Swift 1.3 will be better with our kind of
usage.

It is notable that a lot of our above examples won’t compile right now, and that
we are hopeful that this is just temporary.

## AnyPromise Resolves With `AnyObject?`

Because `AnyPromise` is for Objective-C it can only hold objects that Objective-C can understand. Thus if it cannot be `id` it cannot resolve an `AnyPromise`.

# Porting From PromiseKit 1.x

Be aware of:

* Cancellation
* `AnyPromise` no longer catches most exceptions. You can still `@throw` strings and `NSError` objects. We decided in the end that exceptions in Objective-C mostly represent serious programmer errors and should be allowed to crash the program. [Discussion here](https://github.com/mxcl/PromiseKit/issues/13)
* `PMKPromise` will continue to work as a namespace, but is considered deprecated
* Features like `when` have been moved to top-level functions, eg. `[PMKPromise when:]` is now `PMKWhen`. For swift they are the same (`when`, `join`, etc.)
* `PMKJoin` has a different parameter order, see the documentation
* PromiseKit 2.0 has an iOS 7 minimum deployment target. Though users who want convenience it is in fact 8.0. This is because CocoaPods and Carthage will only build Swift projects for iOS 8. We intend to explore building a static library that will work on iOS 7, so stay tuned if you must use PromiseKit 2 on iOS 7 and don’t want to manually compile the framework. The other option is PromiseKit 1.x which provided you don’t use the Swift version supports back to iOS 6.
* Few exceptions are caught by `AnyPromise`. Because we explicitly encouraged it in the PromiseKit 1.x documentation, we still catch thrown `NSString` objects and thrown `NSError` objects. As before, `Promise<T>` will not catch anything since you can’t throw nor can you catch anything in Swift.
* PromiseKit 2 is mostly written in Swift, this means you will have to check the relevant project settings to embed a swift framework.

In a few months we will delete the Swift portion of PromiseKit 1.x. It was never officially endorsed and 2.x is better in every way.


# The Future

PromiseKit is under active development and is used in hundreds of apps on the store. We will continue to improve and maintain it, provided you use it!

[PromiseKit on Github](https://github.com/mxcl/PromiseKit)