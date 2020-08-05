---
layout: post
title:  "Combine Publishers, Part 1: Creating a Custom Publisher"
date:   2020-07-24
categories: posts
---

If you're just getting started with Combine, the idea of a custom publisher can sound scary, but diving into the topic has many benefits: you'll understand how parts of the framework work together, and will be able to create your own Combine-powered APIs. 

## Getting Started

Along with introducing Combine, Apple also extended many well-known APIs, such as `URLSession` and `NotificationCenter` to offer built-in Publishers. Make sure to have a look in the documentation before deciding to roll your own implementation.

AVFoundation is a good candidate to extend: is has events delivered via KVO, NotifcationCenter, and some of them you'll have to query yourself. With Combine's Publishers you could unify the event delivery. This post will guide you through the process by creating a publisher for observing AVPlayer's playback progress.

There are multiple ways for creating your own Publishers: you can use the `@Published` property wrapper, or use a `PassthroughSubject` instance to send values on demand. In this guide however, you'll build a Publisher from scratch.

Let's have a look at Combine's the key components.

### Publishers

A Publisher represents a type that delivers values over time to Subscribers. A Publisher's job is to accept a Subscriber, which it will later notify as events occur. Combine also offers various operators; these are Publishers, that receive data from an upstream Publishers, manipulate the data (e.g. map the received values to another type), and send the results downstream.

### Subscribers

A Subscriber receives values from a Publishers. Along with those values, it may also receive lifecycle events (such as completion). Combine provides two built-in Subscriber implementation:
* `sink(receiveCompletion:receiveValue:)`: to execute arbitrary work as events occur
* `assign(to:on:)`: to assign received values to a key path of an object.
It's also possible to create your own Subscriber, which will be covered later in the series.

### Subscriptions

Subscriptions represent the connection between a Publisher and a Subscriber. You'll look at Subscribers in detail when implementing a Publisher.

### Sequence of Events

These components operate together following a sequence of events:
1. Subscriber subscribes to a Publisher
2. The Publisher creates the Subscription, and passes it to the Subscriber
3. The Subscriber will request values
4. The Publisher sends values
5. The Publisher completes (either regularly or due to an error)

## Creating the Publisher

The goal of your Publisher will be to provide playback progress updates over a given interval. To do this, you can rely on AVPlayer's `addPeriodicTimeObserver(forInterval:queue:using:)` method, which will periodically invoke its closure parameter to report the progress.

Let's start by declaring the new Publisher:

```swift
extension Publishers {
    struct PlayheadProgressPublisher: Publisher {
        
    }
}
```

This bit of code will not build, but Xcode will offer some helpful hints on how to proceed: A Publisher must declare the type of values, and errors it can emit. This Publisher will emit the current progress in seconds, and will not emit an error. Let's update the implementation:

```swift
extension Publishers {
    struct PlayheadProgressPublisher: Publisher {
        typealias Output = TimeInterval
        typealias Failure = Never
    }
}
```

After declaring the types, you'll still get build errors, but thankfully Xcode will offer another round of fix-its. Update your implementation with the following:

```swift
extension Publishers {
    struct PlayheadProgressPublisher: Publisher {
        typealias Output = TimeInterval
        typealias Failure = Never
        
        func receive<S>(subscriber: S) where S : Subscriber, Self.Failure == S.Failure, Self.Output == S.Input {
            // TODO:
            // 1. Create Subscription
            // 2. Pass Subscription to Subscriber
        }
    }
}
```

Whenever a Subscriber subscribes to a Publisher, the `receive(subscriber:)` method is invoked. From that point, it's the Publisher's responsibility to create a Subscription, and pass it back to the Subscriber. Take a moment to look at the method signature: it states that the Subscriber's Input type must match the Publisher's Output type, and the Failure types must also match. Think back to the sequence of events: **Steps 1 and 2** are covered here.

To be able to pass the Subscription to the Subscriber, you'll first need to take a detour to implement it. Let's have a look.

## Creating the Subscription

The Subscription is where most of the work will happen: it's responsible for performing the work based on the demand of the Subscriber. Below the `PlayheadProgressPublisher` struct, add the following declaration:

```swift
extension Publishers {
    ...
    private final class PlayheadProgressSubscription<S: Subscriber>: Subscription where S.Input == TimeInterval {
        
    }
}
```

Notice how the type signature restricts the Subscriber's Input to be `TimeInterval`. Xcode will step in again, offering to add stubs for a couple of methods:

```swift
private final class PlayheadProgressSubscription<S: Subscriber>: Subscription where S.Input == TimeInterval {
    func request(_ demand: Subscribers.Demand) {
        
    }
    
    func cancel() {
        
    }
}
```

The first method will cover **Step 3** in the sequence of events: when a Subscriber starts requesting values from a Publisher, `request(_:)` will be invoked on the Subscription. The other method, `cancel()` is invoked when the Subscription is cancelled; it's your chance to clean up.

Before diving into the implementation of these methods, let's add a few properties, and an init method to the Subscription:

```swift
private final class PlayheadProgressSubscription<S: Subscriber>: Subscription where S.Input == TimeInterval {
    private var subscriber: S?
    private var requested: Subscribers.Demand = .none
    private var timeObserverToken: Any? = nil
    
    private let interval: TimeInterval
    private let player: AVPlayer
    
    init(subscriber: S, interval: TimeInterval = 0.25, player: AVPlayer) {
        self.player = player
        self.subscriber = subscriber
        self.interval = interval
    }
    ...
}
```

Let's look at them one by one:
* `subscriber`: the Subscription retains the Subscriber to be able to notify it as events occur
* `requested`: the Subscription keeps track of the demand coming from a Subscriber. There is an initial value passed via `request(_:)`, but it can also increase during the lifetime of the Subscription.
* `timeObserverToken`: will be used to hold the return value of AVPlayer's `addPeriodicTimeObserver(forInterval:queue:using:)`
* `interval`: the time interval at which values should be provided
* `player`: the AVPlayer instance to observe

Now it's time to implement `request(_:)`. Update your implementation to the following:

```swift
func request(_ demand: Subscribers.Demand) {
    // 1.
    requested += demand
    // 2.
    guard timeObserverToken == nil else { return }
    
    // 3.
    let interval = CMTime(seconds: self.interval, preferredTimescale: CMTimeScale(NSEC_PER_SEC))
    timeObserverToken = player.addPeriodicTimeObserver(forInterval: interval, queue: DispatchQueue.main) { [weak self] time in
        // 4.
        guard 
            let self = self, 
            let subscriber = self.subscriber, 
            self.requested > .none else { return }
        // 5.
        self.requested -= .max(1)
        // 6.
        let newDemand = subscriber.receive(time.seconds)
        self.requested += newDemand
    }
}
```

Okay, that's a lot of new code, let's go over the changes:
1. When the Subscriber requests values, it can specify how many values it wants by passing the initial demand. The Subscription is responsible for keeping track of the demand, so you'll increment `requested` by the received amount.
2. The goal is to only start emitting events once a Subscriber is attached to a Publisher. If `timeObserverToken` is nil, that means that the Subscription hasn't started producing values yet. Checking the demand is also important: it could be that you're dealing with a custom Subscriber instance which only requests values if a certain condition is true: your Subscription shouldn't complete right away, just defer the work until there's actual demand for values.
3. At this point the Subscription starts to query the playback progress, with the frequency specified in `interval`.
4. Once there is a new value to emit, the implementation checks if there are values demanded.
5. Then `requested` is decremented to avoid sending more values than needed.
6. The value is then delivered to the Subscriber (**Step 4** in the event sequence). Upon receiving a value, the Subscriber may choose to update the demand, so the Subscription must update `requested` to keep up with the new demand.

> In some use cases it might make sense to send a completion event at some point (e.g. if the demand dropts to zero). This implementation however will only complete when the Subscription is cancelled, to work better with Combine's demand system.

Notice how it's entirely up to the Subscribtion implementation to honor the demand. This is a crucial point: if you forget to decrement `requested`, the Subscriber may emit more values than requested; if you don't keep track of the updated demand, the Subscriber can end up delivering fewer values. There is no automatic behavior you can rely on to update the demand, and manual bookkeeping can be error-prone, which is why it's important to unit test your custom Publishers, which will be covered in Part 2 of the series.

There is one final piece the for the Subscription, which is cancellation. Update the implementation of `cancel()`:

```swift
func cancel() {
    if let timeObserverToken = timeObserverToken {
        player.removeTimeObserver(timeObserverToken)
    }
    timeObserverToken = nil
    subscriber = nil
}
```

These are cleanup steps: the Subscription stops observing the playback progress, and nils out `timeObserverToken` and its reference to the Subscriber (the Subscriber alredy retains the Subscription, so this step is necessary to break the retain cycle).

And with that, the implementation of Subscription is complete. Now it's time to connect the parts.

## Passing the Subscription to the Subscriber

The final piece of the puzzle is to pass your new Subscription implementation to the Subscriber upon subscription. Before doing that, you'll need to add a few more properties to the Publisher:

```swift
private let interval: TimeInterval
private let player: AVPlayer

init(interval: TimeInterval = 0.25, player: AVPlayer) {
    self.player = player
    self.interval = interval
}
```

Notice how the properties mirror the input parameters of the Subscription. This is no accident, you'll use them to initialize it. Now update the implementation of `receive(subscriber:)`:

```swift
func receive<S>(subscriber: S) where S : Subscriber, Self.Failure == S.Failure, Self.Output == S.Input {
    let subscription = PlayheadProgressSubscription(subscriber: subscriber,
                                                    interval: interval,
                                                    player: player)
    subscriber.receive(subscription: subscription)
}
```

Here you'll create a new Subscription and pass it to the Subscriber to kick off the event stream.

## Trying it out

Now that your custom Publisher is ready to use, it's time to finally give it a try. To make the new Publisher easier to access, declare the following AVPlayer extension:

```swift
extension AVPlayer {
    func playheadProgressPublisher(interval: TimeInterval = 0.25) -> Publishers.PlayheadProgressPublisher {
        Publishers.PlayheadProgressPublisher(interval: interval, player: self)
    }
}
```

Now you can start observing the playback progress:

```swift
var subscriptions = Set<AnyCancellable>()
let videoURL = URL(string: "https://commondatastorage.googleapis.com/gtv-videos-bucket/sample/BigBuckBunny.mp4")!

let player = AVPlayer(url: videoURL)

player.playheadProgressPublisher()
    .sink { (time) in
        print("received playhead progress: \(time)")
    }
    .store(in: &subscriptions)
```

## Conclusion

I hope you enjoyed this guide on custom Publishers. It may seem like a lot to digest at first, so definitely take your time to play around with the concept, try to apply it on some of your own existing code. 


If you would like to see these topics in context, check out [the full project on GitHub](https://github.com/jozsef-vesza/AVFoundation-Combine), which contains all the code discussed here, and has an example app you can play around.

As noted previously, it's definitely a good idea to back your Publisher implementations up with unit tests, and in [Part 2](https://jozsef-vesza.dev/2020/08/01/unit-testing-publishers/), you'll learn about how to just that.