+++
title = "Asynchronous programming with streams"
date = "2023-06-26T11:16:07+02:00"
description = "Introduction to Combine Publishers, Subscribers and Subscriptions"
image = ["static/images/asynchronous-programming-with-streams.png"]
tags = ["ios","swift","combine","rxswift"]
+++

&nbsp;

## Publisher

------

Using standard types (Int, String...) allows one to store a piece of information as the *current* value of some variable, but these variables don't know anything about the *past* or *future values* that were and will be part of the application state.

**`Int`**  
*Independent values*  
  **3     5  
 4   6   9**  

Combine framework introduces a concept of [**Publisher**](https://developer.apple.com/documentation/combine/publisher) which abstracts this *change of value over time* in a declarative and functional way. The concept of publisher is used interchangeably with the word **stream**. Streams offer a natural way for dealing with sequence of *events.* 

**`Publisher<Int>`**  
*Values over time (stream)*  
── **6** ── **3** ── **4** ── **9** ── **5** ──►  

&nbsp;

> Combine is not the first framework with such paradigm. RxSwift (and other reactive frameworks) use a concept of an **Observable** which is a counterpart to publisher and many of the concepts introduced here are directly correlating with those in other frameworks.  
  
&nbsp;

## Publisher events

------

There are three main **events** that a publisher can emit:

- **Value** event
- **"Failure" completion** event
- **"Finished" completion** event

Value events are events that carry value with them. Publisher can emit any number of these events as long as there hasn't been any completion event.

── **value** ── **value** ── **value** ── **value** ──►

Completion events are events that terminate the stream. There will be no more value events after this type of event which effectively stops the stream. There are two types of completion events, one carrying the underlying error, and the other just signalling that the stream is finished.

── **value**  ── **value** ── **.failure(error)** ──**X**─► *(stream is terminated)*

── **value** ── **value** ── **.finished** ────**X**─► *(stream is terminated)*

&nbsp;

## Creating publishers

------

The statement that allows this paradigm to be useful in the application programming context is that *everything is a stream* or at least can be converted into a stream. There are several ways in which Combine framework gives you abilities to create one.

### [**Just**](https://developer.apple.com/documentation/combine/just)

This is a *convenience publisher* that emits a value event just once, and then finishes. It is often used for mock implementations of services, or when you need to convert certain variable into a stream with a single value event.

```swift
let publisher: AnyPublisher<Int, Never> = Just(25)
											.eraseToAnyPublisher()
```

### [**Publisher.Sequence**](https://developer.apple.com/documentation/combine/publishers/sequence)

This is a *convenience publisher* that emits a sequence of value events, and then finishes.

```swift
let publisher: AnyPublisher<Int, Never> = Publishers.Sequence(sequence: [1, 2, 3])
    .eraseToAnyPublisher()

// Alternatively, you can utilize built-in extension on Sequence struct that exposes a .publisher method

let publisher: AnyPublisher<Int, Never> = [1, 2, 3].publisher
    .eraseToAnyPublisher()
```

### [**Future**](https://developer.apple.com/documentation/combine/future)

This is a *convenience publisher* that eventually emits an output just once, and then finishes or fails. It is often used for wrapping existing code into Combine publishers, thus creating *a bridge* for any sync/async task that needs to be converted into a stream.

```swift
let publisher: AnyPublisher<Int, Error> = Future<Int, Error>() { promise in
    // Do anything here, and if it succeeds:
    promise(.success(25))
    // and if it fails
    promise(.failure(NSError()))
}
.eraseToAnyPublisher()
```

### DataTaskPublisher

The [`dataTaskPublisher(for:)`](https://developer.apple.com/documentation/foundation/urlsession/3329708-datataskpublisher) is a wrapper for URLSession's [`dataTask(with:completionHandler:)`](https://developer.apple.com/documentation/foundation/urlsession/1410330-datatask) method. It is used for making network calls and returning the data/response/error as a stream event instead of values in a closure.

```swift
struct Todo {}
let url = URL(string: "https://jsonplaceholder.typicode.com/todos/1")!

let publisher: AnyPublisher<Todo, Error> = URLSession.shared.dataTaskPublisher(for: url)
    .tryMap { (data, response) in
        return data as! Todo
    }
    .eraseToAnyPublisher()
```

&nbsp;

> Each time an operator is used with publisher, type wrapping occurs which will produce very long type names which are not relevant to the consumers, and that is the reason for type erasure call at the end of the publisher declaration chain.
>
> Another point is that `Publisher` is a protocol with associated types, while `AnyPublisher` is a type-erased struct that conforms the `Publisher` protocol. If the protocol has associated types that each underlying type provides on its own, Swift will strictly forbid you from referring to this protocol if you're also not providing appropriate generic constraints. You can read more about type erasure [`in this article`](https://swiftrocks.com/whats-any-understanding-type-erasure-in-swift).

&nbsp;

## Subscription

------

Streams can be instantiated or exist on their own, but they aren't of much use if they aren't *observed*. The way in which we would observe the events happening on a stream is by using a specific [**Subscriber**](https://developer.apple.com/documentation/combine/subscriber), connecting it to a publisher and getting back a connection called [**Subscription**](https://developer.apple.com/documentation/combine/subscription).

```Swift
var subscriptions = Set<AnyCancellable>()

let subscription: AnyCancellable = [1, 2, 3].publisher
    .sink { value in 
		print("Received \(value)") 
		}
		.store(in: &subscriptions)
```

`Subscription` protocol implements `Cancellable` protocol, which just defines a single method called `cancel()`, and subscribers (once connected to the publisher instance) return a subscription object as a type-erased `AnyCancellable`.

The concept of controlling flow by signaling a subscriber’s readiness to receive elements is called *back pressure*. Many apps just use the operators [`sink(receiveValue:)`](https://developer.apple.com/documentation/combine/publisher/sink(receivevalue:)) and [`assign(to:on:)`](https://developer.apple.com/documentation/combine/publisher/assign(to:on:)) to create the convenience subscriber types [`Subscribers.Sink`](https://developer.apple.com/documentation/combine/subscribers/sink) and [`Subscribers.Assign`](https://developer.apple.com/documentation/combine/subscribers/assign), respectively. These two subscribers issue a demand for [`unlimited`](https://developer.apple.com/documentation/combine/subscribers/demand/unlimited) when they first attach to the publisher.

Since publishers emit events which are asynchronous, leaving the scope of publisher definition would cause Swift's memory management to delete that weak reference and the stream would be deallocated, which is the reason why a strong reference of a stream must be kept, usually with a set of AnyCancellables and a [`store(in:)`](https://developer.apple.com/documentation/combine/anycancellable/store(in:)-3hyxs) method.

&nbsp;

## TLDR;

> To recap, within the Combine framework, one can create streams from scratch, use stream extensions from different built-in APIs (URLSession, Timers etc.) or create one's own stream wrappers for any other use. These streams will asynchronously emit events related to their purpose, namely value events and completion events. Subscribing to those streams allows one to react to them and act accordingly.
> If this was the only functionality of stream paradigm, this would be nothing other than a new way of dealing with asynchronous code (along with already numerous way one can do it in the Apple ecosystem).
> The true power lies in combining and attaching streams to different streams, using different operators to filter, map or delay incoming data. 
> The next part in the series will continue with demonstration of the various use-cases of stream-oriented development.

### Resources
- [Combine - Apple documentation](https://developer.apple.com/documentation/combine) 📖
- [Understanding Combine - Flo writes code](https://www.youtube.com/watch?v=rz0yx0Qz2jE) ▶️

### Next read
- Coming soon... ✍️
