---
layout: post
title:  "Combine Publishers, Part 2: Unit Testing Custom Publishers"
date:   2020-07-27
categories: posts
---

In [Part 1](https://jozsef-vesza.dev/2020/07/24/creating-a-custom-combine-publisher/) of the series, you've built a Publisher from scratch. One of the key parts of Combine's event delivery is keeping up with the Subscriber's demand. It involves some manual bookkeeping, which is known to be error-prone, so in this guide you'll learn at how to ensure that your custom Publisher is not misbehaving. 

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