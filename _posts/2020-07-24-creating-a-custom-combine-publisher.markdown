---
layout: post
title:  "Combine Publishers, Part 1: Creating a Publisher"
date:   2020-07-24
categories: posts
---

If you're just getting started with Combine, the idea of a custom publisher can sound scary, but diving into the topic has many benefits: you'll understand how parts of the framework work together, and will be able to create your own Combine-powered APIs. 

This post will guide you through the process by creating a publisher for observing AVPlayer's playback progress.

## Getting Started

There are three key players in Combine:

#### Publishers

A Publisher represents a type that delivers values over time to Subscribers. Along with Combine, Apple also extended many well-known APIs, such as `URLSession` and `NotificationCenter` to offer built-in Publishers. Feel free to rely on them if they suit your needs instead of reinventing the wheel.

#### Subscribers

A Subscriber receives values from a Publishers. Along with those values, it may also receive lifecycle events (such as completion). Combine provides two built-in Subscriber implementation:
* `sink(receiveCompletion:receiveValue:)`: to execute arbitrary work as events occur
* `assign(to:on:)`: to assign received values to a key path of an object.
It's also possible to create your own Subscriber, which will be covered later in the series.

#### Subscriptions

Subscriptions represent the connection between a Publisher and a Subscriber. You'll look at Subscribers in detail when implementing a Publisher.

These components operate together following a sequence of events:
1. Subscriber subscribes to a Publisher
2. The Publisher creates the Subscription, and passes it to the Subscriber
3. The Subscriber will request values
4. The Publisher sends values
5. The Publisher completes (either regularly or due to an error)

## Creating a Publisher

The goal of your Publisher will be to provide playback progress updates over a given interval. To do this, you can rely on AVPlayer's `addPeriodicTimeObserver(forInterval:queue:using:)` method.

Let's start by declaring the new Publisher:

```swift
extension Publishers {
    struct PlayheadProgressPublisher: Publisher {
        
    }
}
```

This bit of code will not build, but Xcode will offer some hints on what to implement next: A Publisher must declare the type of values, and errors it can emit. This Publisher will emit the current progress in seconds, and will not emit an error. Let's update the implementation:

```swift
extension Publishers {
    struct PlayheadProgressPublisher: Publisher {
        typealias Output = TimeInterval
        typealias Failure = Never
    }
}
```

After declaring the types, you'll still get build errors, but Xcode will offer you another suggestion to get closer to your goal. Update your implementation with the following:

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

Whenever a Subscriber subscribes to a Publisher, the `receive(subscriber:)` method is invoked. From that point, it's the Publisher's responsibility to create a Subscription, and pass it back to the Subscriber. Take a moment to look at the method signature: it states that the Subscriber's Input type must match the Publisher's Output type, and the Failure types must also match. Think back to the sequence of events: Steps 1 and 2 are covered here.

You'll get back to this method implementation in just a bit, but let's take care of the Subscription first.

The Subscription is where most of the work will happen: it's responsible for performing the work based on the demand of the Subscriber. Below the `PlayheadProgressPublisher` struct, add the following declaration:

```swift
extension Publishers {
    ...
    private final class PlayheadProgressSubscription<S: Subscriber>: Subscription where S.Input == TimeInterval {
        
    }
}
```

Notice how the type signature restricts the Subscriber's Input to be `TimeInterval`. Xcode will step in agan, offering to add stubs for a couple of methods:

```swift
private final class PlayheadProgressSubscription<S: Subscriber>: Subscription where S.Input == TimeInterval {
    func request(_ demand: Subscribers.Demand) {
        
    }
    
    func cancel() {
        
    }
}
```

The first method will cover Step 3 in the sequence of events: when a Subscriber requests values from a Publisher, `request(_:)` will be invoked on the Subscription. The other method is invoked when the Subscription is cancelled; it's your chance to clean up.

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