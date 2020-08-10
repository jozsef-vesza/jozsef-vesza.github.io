---
layout: post
title:  "Combine Publishers, Part 2: Unit Testing Custom Publishers"
date:   2020-08-01
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
        // 2.
        guard 
            let self = self, 
            let subscriber = self.subscriber, 
            self.requested > .none else { return }
        // 3.
        self.requested -= .max(1)
        let newDemand = subscriber.receive(time.seconds)
        // 4.
        self.requested += newDemand
    }
}
```

Looking at the marked lines, it's becoming clear that the responsibility of coping with the demand falls entirely on the Subscription implementation:
1. First, it registers the initial demand.
2. When a value is sent to the Subscriber, it first checks if there'a a demand to fulfill.
3. Then, it updates the demand to avoid sending too many values.
4. The Subscriber may choose to increment the demand, so the Subscription updates the value stored in `requested` to keep up.

To cover all theses cases, you could divide the tests into the following categories:
* Tests, that work with unlimited demand: The most common use case for Publishers is to subscribe using `sink(receiveCompletion:receiveValue:)`, and handle values as they are emitted. So you'll have to make sure that your Publisher works well with this setup.
* Tests, that work with a fixed demand: it's fairly common that a Subscriber wants to limit the number of values emitted by a Publisher. Your tests need to verify that the Publisher emits the correct amount of values.
* Tests, that dynamically change the demand: a more advanced scenario is to update the demand during a Subscription's lifetime. This can be useful for heavier tasks, where the Publisher emits so many values that the Subscriber has a hard time keeping up. In this guide you'll create tests for the case where the demand is initially zero, but later increased.

## Setting Up the Tests

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

## Testing With Sink

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

This bit just declares the types of values and errors you're interested in: it must match the Publisher's emitted types, so you'll use `TimeInterval` and `Never`. You'll notice a few build errors now; Xcode's fix-its to show the path forward. After applying them, you should see three new method stubs:

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
class TestSubscriber: Subscriber {
    private let demand: Int
    private let onValueReceived: (Int) -> Void
    
    private var subscription: Subscription? = nil

    init(demand: Int, onValueReceived: @escaping (_ receivedValue: Int) -> Void) {
        self.demand = demand
        self.receivedValues = receivedValues
    }
    ...
}
```
Let's look at the properties one by one:
* `demand`: the purpose of this Subscriber implementation is to gain control over the demand, so it will receive the demand as an `init` parameter.
* `onValueReceived`: this closure will be invoked as values are received from the Publisher.
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

### Receiving Values

When the Publisher produces a value, it passes it to the Subscriber by calling `receive(_:)`. The Subscriber can then process the value, and optionally update the demand. Update the body of `receive(_:)` to the following:
```swift
func receive(_ input: Int) -> Subscribers.Demand {
    return .none
}
```
In its current state, `TestSubscriber` will work with the demand passed in during initialization, and will not change it dynamically.

### Receiving Completion

Finally, fill out the implementation of `receive(completion:)`:
```swift
func receive(completion: Subscribers.Completion<Never>) {
    onComplete()
    subscription = nil
}
```
The Publisher will call `receive(completion:)` when it finishes (either normally or with an error). Your implementation will invoke the completion handler, and nil out its reference to the Subscription. The latter step is important to break the retain cycle between the Subscriber and the Subscription.

## Testing With Arbitrary Demand

Now it's time to add a test for the scenario where a fixed number of values are requested. Add the following test method to your test case:
```swift
func testWhenTwoValuesAreRequested_ItCompletesAfterEmittingTwoValues() {
    // 1.
    let expectedValues: [TimeInterval] = [1, 2]
    var receivedValues: [TimeInterval] = []
    
    // 2.
    let subscriber = TestSubscriber(demand: 2) { value in
        receivedValues.append(value)
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

## Delaying the Publisher’s Work

The following test will add another twist to the setup: the initial demand will be zero, but you'll request additional values later. This will validate that the Publisher doesn't start emitting values prematurely. In order to manipulate the demand on the fly, add the following method to `TestSubscriber`:
```swift
func startRequestingValues(_ demand: Int) {
    guard let subscription = subscription else {
        fatalError("requestValues(_:) may only be called after subscribing")
    }
    subscription.request(.max(demand))
}
```

The method makes sure that the subscription already exists, then requests the specified values.

Now add the following test method:
```swift
func testWhenDemandIsZero_ItEmitsNoValues() {
    let expectedValues: [TimeInterval] = []
    var receivedValues: [TimeInterval] = []
    
    let subscriber = TestSubscriber(demand: 0) { value in
        receivedValues.append(value)
    }
    
    sut.subscribe(subscriber)
    
    let timeUpdates: [TimeInterval] = [1, 2, 3, 4, 5]
    timeUpdates.forEach { time in player.updateClosure?(CMTime(seconds: time, preferredTimescale: CMTimeScale(NSEC_PER_SEC))) }
    
    XCTAssertEqual(receivedValues, expectedValues)
}
```

This test verifies that the Publisher doesn't start emitting values if the initial demand is zero. To test modifying the demand, you'll use a slightly tweaked variation of the same method:

```swift
func testWhenInitialDemandIsZero_AndThenFiveValuesAreRequested_ItEmitsFiveValues() {
    let expectedValues: [TimeInterval] = [1, 2, 3, 4, 5]
    var receivedValues: [TimeInterval] = []
    
    let subscriber = TestSubscriber(demand: 0) { value in
        receivedValues.append(value)
    }
    
    sut.subscribe(subscriber)
    
    let timeUpdates: [TimeInterval] = [1, 2, 3, 4, 5]

    // Request more values
    subscriber.startRequestingValues(5)
    
    timeUpdates.forEach { time in player.updateClosure?(CMTime(seconds: time, preferredTimescale: CMTimeScale(NSEC_PER_SEC))) }
    
    XCTAssertEqual(receivedValues, expectedValues)
}
```

The only notable difference is that in this test you request five more values after the initial subscription.

## Updating the Demand When a Value Is Received

There is still a gap in the implementation to cover: when a Publisher sends a value by calling `receive(_:)`, the Subscriber has a chance to return an updated demand. One important thing to note here is that the new demand will be added to the existing one. So if the Subscriber initially demands two values, there's no way to decrement that demand; you can return `none` to keep the demand as is, or specify the additional demand. 

In order to test this setup, you'll need to hook into the `receive(_:)` method of the Subscriber. Add the following changes to the implementation of `TestSubscriber`:
In order to test this setup, you'll need to change the `TestSubscriber` to allow modifying the demand:
```swift
class TestSubscriber: Subscriber {
    ...
    private let onValueReceived: (Int) -> Int
    ...
```

Now that `onValueReceived` has a return value, it will allow tests to update the demand. Update the implementation of `receive(_:)`:
```swift
func receive(_ input: Int) -> Subscribers.Demand {
    receivedValues.append(input)
    let newDemand = onValueReceived(input)
    return .max(newDemand)
}
```
The new implementation acquires the updated demand by invoking `onValueReceived`, and passes it back to the Publisher.

To verify the behavior, add the following test method:
```swift
func testWhenInitialDemandIsOne_AndAnAdditionalValueIsRequested_ItEmitsTwoValues() {
    let expectedValues: [TimeInterval] = [1, 2]
    var receivedValues: [TimeInterval] = []
    
    let subscriber = TestSubscriber(demand: 1) { value in
        receivedValues.append(value)
        return value == 1 ? 1 : 0
    }
    sut.subscribe(subscriber)
    
    let timeUpdates: [TimeInterval] = [1, 2, 3, 4, 5]
    timeUpdates.forEach { time in player.updateClosure?(CMTime(seconds: time, preferredTimescale: CMTimeScale(NSEC_PER_SEC))) }
    
    XCTAssertEqual(receivedValues, expectedValues)
}
```

The goal of this test is to request a single value initially, and request an additional one upon receiving it. So it checks the value received, and asks for another one if it equals one.

## Conclusion

By following along, you've covered all the possible use cases of a custom Publisher with tests, which will allow you to be more confident in your implementation. Along the way you've learned about Combine's demand system, and implemented a custom Subscriber: although it was only used to support the unit tests, you may find yourself using such a Subscriber in a real-life scenario as well, when you need more control over the behavior of Publishers. 


If you would like to see these topics in context, check out [the full project on GitHub](https://github.com/jozsef-vesza/AVFoundation-Combine), which contains all the code discussed here, and has an example app you can play around.
I hope this knowledge will serve you well when working with Combine in the wild. If you have any questions or feedback about the topics covered in this series, do not hesitate to reach out to me on [Twitter](https://twitter.com/j_vesza), I would love to hear it.