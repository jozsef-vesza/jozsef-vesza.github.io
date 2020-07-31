---
layout: post
title:  "Combine Publishers, Part 2: Unit Testing Custom Publishers"
date:   2020-07-27
categories: posts
---

In [Part 1](https://jozsef-vesza.dev/2020/07/24/creating-a-custom-combine-publisher/) of the series, you've built a Publisher from scratch. One of the key parts of Combine's event delivery is keeping up with the Subscriber's demand. It involves some manual bookkeeping, which is known to be error-prone, so in this guide you'll learn how to ensure that your custom Publisher is not misbehaving. 

## Getting Started

Let's have a quick look at the current Subscription implementation:

```swift
func request(_ demand: Subscribers.Demand) {
    // 1.
    requested += demand
    
    guard timeObserverToken == nil else { return }
    
    let interval = CMTime(seconds: self.interval, preferredTimescale: CMTimeScale(NSEC_PER_SEC))
    timeObserverToken = player.addPeriodicTimeObserver(forInterval: interval, queue: DispatchQueue.main) { [weak self] time in
        guard let self = self, let subscriber = self.subscriber else { return }
        // 2.
        self.requested -= .max(1)
        let newDemand = subscriber.receive(time.seconds)
        // 3.
        self.requested += newDemand
        // 4.
        if self.requested == .none {
            subscriber.receive(completion: .finished)
        }
    }
}
```

Looking at the marked lines, it's becoming clear that the responsibility of coping with the demand falls entirely on the Subscription implementation:
1. First, it registers the initial demand.
2. As a value is delivered to the Subscriber, it updates the demand to avoid sending too many values.
3. The subscriber may choose to increment the demand, so the Subscription updates `requested` to keep up.
4. Finally, if no more values are requested, the Subscription can complete.

To cover all theses cases, you could divide the tests into the following categories:
* Tests, that work with unlimited demand: The most common use case for Publishers is to subscribe using `sink(receiveCompletion:receiveValue:)`, and handle values as they are emitted. So you'll have to make sure that your Publisher works well with this setup.
* Tests, that work with a fixed demand: it's fairly common that a Subscriber wants to limit the number of values emitted by a Publisher. Your tests need to verify that the Publisher emits the correct amount of values, and completes afterwards.
* Tests, that dynamically change the demand: a more advanced scenario is to update the demand during a Subscription's lifetime. This can be useful for heavier tasks, where the Publisher emits so many values that the Subscriber has a hard time keeping up. In this guide you'll create tests for the case where the demand is initially zero, but later increased.

## Setting up the tests

`PlayheadProgressPublisher` relies on AVPlayer to provide progress updates. In order to test its behavior properly, you need to be able to take control over these updates. Add the following implementation to your unit test case:

```swift
class MockAVPlayer: AVPlayer {
    var updateClosure: ((_ time: CMTime) -> Void)?
    
    // 1.
    override func addPeriodicTimeObserver(forInterval interval: CMTime,
                                          queue: DispatchQueue?,
                                          using block: @escaping (CMTime) -> Void) -> Any {
        // 2.
        updateClosure = block
        // 3.
        return super.addPeriodicTimeObserver(forInterval: interval,
                                             queue: queue,
                                             using: block)
    }
}
```

Let's look at how it works:
1. You'll hook into the AVPlayer's `addPeriodicTimeObserver(forInterval:queue:using:)` method. `PlayheadProgressPublisher` invokes this method when subscribed to, so you can be sure that the hook will be executed. 
2. The default implementation invokes the `block` parameter periodically to post updates. Your mock captures this closure into a property, so you can freely invoke it with arbitrary values in your tests.
3. `AVPlayer` returns an observer token, which is later used to stop the progress updates. Since it's an opaque value, it's better to invoke the default implementation to make sure it's correct.

With the mock in place, it's time to add some setup code to your test case:
```swift
class PlayheadProgressPublisherTests: XCTestCase {
    // 1.
    var sut: Publishers.PlayheadProgressPublisher!
    var player: MockAVPlayer!
    var subscriptions = Set<AnyCancellable>()
    
    // 2.
    override func setUp() {
        player = MockAVPlayer()
        sut = Publishers.PlayheadProgressPublisher(player: player)
    }
    
    // 3.
    override func tearDown() {
        subscriptions = []
    }
}
```
1. You'll keep references to the Publisher ("sut" is an abbreviation for "system under test"), and the mock AVPlayer implementation. Additionally there's a `subscriptions` set which will retain the subscriptions created in test cases.
2. You inject the mock AVPlayer implementation into the Publisher.
3. Each test should start with a clean slate: so once one is finished, it's good to get rid of the subscriptions.

Now that the setup is done, it's time to write your first test.

## Testing with sink

Add the following test method to your test case:
```swift
func testWhenDemandIsUnlimited_AndTimeIsUpdated_ItEmitsTheNewTime() {
    // 1.
    var receivedTimes: [TimeInterval] = []
    let expectedTimes: [TimeInterval] = [1]

    // 2.
    sut.sink { time in
        receivedTimes.append(time)
    }
    .store(in: &subscriptions)
    
    // 3.
    let progress = CMTime(seconds: 1, preferredTimescale: CMTimeScale(NSEC_PER_SEC))
    player.updateClosure?(progress)
    
    // 4.
    XCTAssertEqual(receivedTimes, expectedTimes)
}
```

This unit test represents the most common use case of a Publisher:
1. `receivedTimes` will hold the values received from the Publisher. `expectedTimes` declares the expected output.
2. You'll use `sink(receiveValue:)` to kick off the Publisher. You'll record each received value.
3. AVPlayer represents progress with `CMTime`. Here you'll wrap the progress value of one second, and post it as the update.
4. Here you'll simply check if the received value matches the expectation.

If all goes well, this test should succeed. And with that implementation you've covered the majority of the use cases! Now it's time to look into something a bit more complex.

## Implementing a Subscriber

In order to gain control over the demand, a custom Subscriber implementation is needed.

To get started, add the following declaration:

```swift
class TestSubscriber: Subscriber {
    typealias Input = TimeInterval
    typealias Failure = Never
}
```

This bit just declares the types of values and errors you're interested in: it must match the Publisher's emitted types, so you'll use `TimeInterval` and `Never`. You'll notice a few build errors now; again, you'll rely on Xcode's fix-its to show the path forward. After applying them, you should see three new method stubs:

```swift
class TestSubscriber: Subscriber {
    typealias Input = TimeInterval
    typealias Failure = Never
    
    func receive(subscription: Subscription) {
        
    }
    
    func receive(_ input: TimeInterval) -> Subscribers.Demand {
        
    }
    
    func receive(completion: Subscribers.Completion<Never>) {
        
    }
}
```

Before moving on, add the following properties to `TestSubscriber`:
```swift
class TestSubscriber<Int>: Subscriber {
    private let demand: Int
    private let onComplete: ([Int]) -> Void
    
    private var receivedValues: [Int] = []
    private var subscription: Subscription? = nil

    init(demand: Int, onComplete: @escaping (_ receivedValues: [T]) -> Void) {
        self.demand = demand
        self.onComplete = onComplete
    }
    ...
}
```
Let's look at the properties one by one:
* `demand`: the purpose of this Subscriber implementation is to gain control over the demand, so it will receive the demand as an `init` parameter.
* `onComplete`: this closure will be invoked when the Subscription completes.
* `receivedValues`: this array will hold the values received from the Publisher. Upon completion, it will be passed via `onComplete`.
* `subscription`: the Subscriber needs to hold a strong reference to the Subscription to prevent it from being deallocated.

The next part will build heavily on the sequence of events described in [Part 1](https://jozsef-vesza.dev/2020/07/24/creating-a-custom-combine-publisher/), feel free to check it out if you need a refresher. Let's look at the method stubs.

### Receiving the Subscribtion

Once the Publisher has configured the Subscription, it will pass it back to the Subscriber by calling `receive(subscription:)`. When the Subscriber receives the Subscription, it can start asking for values. It also has to retain the Subscription, otherwise it would get deallocated. Update the body of `receive(subscription:)` to match these requirements:
```swift
func receive(subscription: Subscription) {
    self.subscription = subscription
    subscription.request(.max(demand))
}
```

### Receiving values

When the Publisher produces a value, it passes it to the Subscriber by calling `receive(_:)`. The Subscriber can then process the value, and optionally update the demand. Update the body of `receive(_:)` to the following:
```swift
func receive(_ input: T) -> Subscribers.Demand {
    receivedValues.append(input)
    return .none
}
```
The Subscriber will keep track of the values received, and not request more values. In its current state, `TestSubscriber` will work with the demand passed in during initialization, and will not change it dynamically.

### Receiving completion

Finally, fill out the implementation of `receive(completion:)`:
```swift
func receive(completion: Subscribers.Completion<Never>) {
    onComplete(receivedValues)
    subscription = nil
}
```
The Publisher will call `receive(completion:)` when it finishes (either normally or with an error). Your implementation will invoke the completion handler, and nil out its reference to the Subscription. The latter step is important to break the retain cycle between the Subscriber and the Subscription.

## Testing with arbitrary demand

Now it's time to add a test for the scenario where a fixed number of values are requested. Add the following test method to your test case:
```swift
func testWhenTwoValuesAreRequested_ItCompletesAfterEmittingTwoValues() {
    // 1.
    let expectedValues: [TimeInterval] = [1, 2]
    var receivedValues: [TimeInterval] = []
    
    // 2.
    let subscriber = TestSubscriber(demand: 2) { values in
        receivedValues = values
    }
    
    sut.subscribe(subscriber)
    
    let timeUpdates: [TimeInterval] = [1, 2, 3, 4, 5]
    
    // 3.
    timeUpdates.forEach { time in player.updateClosure?(CMTime(seconds: time, preferredTimescale: CMTimeScale(NSEC_PER_SEC))) }
    
    // 4.
    XCTAssertEqual(receivedValues, expectedValues)
}
```

Let's look at it in detail:
1. Just as before, you establish the expected result, and declare an array to hold the received values.
2. You initialize a `TestSubscriber` instance which will request two values.
3. To verify that the Publisher doesn't send more values than needed, you'll use the mocked `updateClosure` to produce five values.
4. Finally, you'll validate the received values.

Run your test now, it should succeed.

## Dynamically changing the demand