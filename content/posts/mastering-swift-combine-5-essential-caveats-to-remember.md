---
title: "Mastering Swift Combine: 5 Essential Caveats to Remember"
slug: "mastering-swift-combine-5-essential-caveats-to-remember"
date: 2025-06-26T10:00:00+05:30
draft: false
---
_Combine framework’s caveats: Learn key pitfalls and actionable insights for effective integration into your code._


# Mastering Swift Combine: 5 Essential Caveats to Remember

### Combine framework’s caveats: Learn key pitfalls and actionable insights for effective integration into your code.

Combine, Apple’s own reactive programming framework, holds immense potential. However, effectively using its capabilities demands one to be aware of specific caveats and pitfalls.

This article explores these caveats and provides actionable insights for integrating Combine effectively into your code. While some considerations here may seem intuitive, they are often overlooked until you grasp Combine thoroughly.

This article assumes you have familiarity with the reactive paradigm and some practical experience with Combine.

📝Here’s your checklist for mastering Combine:

### Caveat #1: Memory Management and Retain Cycles

Combine extensively uses closures. And closures, notorious for creating retain cycles if not managed carefully, can lead to memory leaks.

Always use[weak self]or[unowned self]in closures to prevent strong captures ofselfand avoid creating retain cycles.

`[weak self]`

`[unowned self]`

`self`

Combine’s cancellables manage resources and automatically clean up when objects are deallocated, making the usage of[unowned self]generally safe. Check the code example below:

`[unowned self]`

```swift
final class Main {
    var cancellable: AnyCancellable?
    var currentValue: Int?
    
    deinit {
        print("Deinitialised \(self)")
    }
    
    func run() {
        cancellable = dummyPublisher()
            .sink { [unowned self] value in
                print("Received value: \(value)")
                self.currentValue = value
            }
    }
    
    /// Publisher that emits a value every second
    /// - Returns: The number of seconds passed since invoke
    private func dummyPublisher() -> AnyPublisher<Int, Never> {
        Timer.publish(every: 1, on: .main, in: .common)
            .autoconnect()
            .scan(0) { partialResult, _ in
                partialResult + 1
            }
            .eraseToAnyPublisher()
    }
}
```

In the example above,dummyPublisher()function returns a non-completing publisher. In therun()function, we usesinkoperator and captureselfasunownedin the closure.

`dummyPublisher()`

`run()`

`sink`

`self`

`unowned`

```swift
var object: Main? = Main()
object?.run()

DispatchQueue.main.asyncAfter(deadline: .now() + 2) {
    object = nil
}

----------------------
/// OUTPUT: 
Received value: 1
Received value: 2
Deinitialised __lldb_expr_9.Main
```

Later, when we set the object tonil, we expect a crash because theunowned selfclosure expects a validselfinstance, butselfis set tonil.

`nil`

`unowned self`

`self`

`self`

`nil`

However, Combine effectively manages cancellables. Upon setting the object tonil, the object releases its reference to thecancellable, which then callscancel()on its members, effectively cancelling the publisher chain.

`nil`

`cancellable`

`cancel()`

If you are certain about when the subscription will no longer be needed, you can also manually cancel it.

```swift
func run() {
    cancellable = dummyPublisher()
        .sink { [self] value in
            // strongly holding self
            print("Received value: \(value)")
            self.currentValue = value
        }
}
```

```swift
DispatchQueue.main.asyncAfter(deadline: .now() + 2) {
    object?.cancellable = nil
}

----------------------
/// OUTPUT: 
Received value: 1
Received value: 2
Deinitialised __lldb_expr_9.Main
```

In the modified code above, despite holding a strong reference toselfin the closure, as we cancel the subscription after 2 seconds, the object is subsequently released without creating a retain cycle.

`self`

However, this approach is generally not recommended due to the need for precise timing to avoid creating retain cycles, especially in multi-developer projects where it can be more error-prone.

- Always include a test case for an unreferenced controller in your tests to ensure there are no memory leaks.

Always include a test case for an unreferenced controller in your tests to ensure there are no memory leaks.

```swift
func testViewControllerNotRetained() {
    // Create two variables for the view controller,
    // one strong and one weak
    var sut: ViewController? = ViewController()
    weak var weakSut = sut
    
    // Nilling out the strong reference should release the object,
    // making the weak reference also nil
    sut = nil
    XCTAssertNil(weakSut)
}
```

### Caveat #2: Error Handling

Combine’s error handling can be less intuitive compared to traditional mechanisms, which can cause confusion among beginners and lead to hours of debugging.

First Caveat in Error Handling:

Always keep in mind that Combine operators that handle upstream failures and successes are mostly mutually exclusive — each operator typically specialises ineitherhandling failures or successes, butnot both. (This generalisation of course excludes operators likesink, but keeping this in mind can save hours of debugging.)

`sink`

To clarify, here are some examples:

- Operators that only consume success from upstream publishers:map,flatMap,filter,tryFilteretc.

Operators that only consume success from upstream publishers:map,flatMap,filter,tryFilteretc.

`map`

`flatMap`

`filter`

`tryFilter`

- Operators that only consume error from upstream publishers:catch,tryCatch,retryetc.

Operators that only consume error from upstream publishers:catch,tryCatch,retryetc.

`catch`

`tryCatch`

`retry`

Check the below example.

```swift
let numbers = [4, 5]

enum CustomError: Error {
    case failure
}

let cancellable = numbers.publisher
    .tryMap { number in
        let isEven = number % 2 == 0
        print("Inside tryMap block for \(isEven ? "even" : "odd") number")
        if isEven {
            return number
        } else {
            throw CustomError.failure
        }
    }
    .tryCatch { error -> AnyPublisher<Int, Error> in
        print("Inside tryCatch block...")
        let isSuccess = Bool.random()
        if isSuccess {
            print("Replacing error with random success publisher")
            return Just(-1).setFailureType(to: Error.self).eraseToAnyPublisher()
        } else {
            print("Re-throwing error")
            throw CustomError.failure
        }
    }
    .map { value -> Int in
        print("Inside map block...")
        return value * 2
    }
    .sink(receiveCompletion: { completion in
        print("Received completion: \(completion)\n")
    }, receiveValue: { value in
        print("Received value: \(value)\n")
    })
```

Output scenario 1 whenBool.random()returnstrue

`Bool.random()`

`true`

```swift
----------------------
/// OUTPUT Scenario #1: 
Inside tryMap block for even number
Inside map block...
Received value: 8

Inside tryMap block for odd number
Inside tryCatch block...
Replacing error with random success publisher
Inside map block...
Received value: -2

Received completion: finished
```

Output scenario 2 whenBool.random()returnsfalse

`Bool.random()`

`false`

```swift
----------------------
/// OUTPUT Scenario #2:
Inside tryMap block for even number
Inside map block...
Received value: 8

Inside tryMap block for odd number
Inside tryCatch block...
Re-throwing error
Received completion: failure(__lldb_expr_153.CustomError.failure)
```

In the example above, we emit success for an even number and an error for an odd number. For odd numbers, we useBool.random()to decide whether to emit success or failure.

`Bool.random()`

Notice how thetryCatchblock is skipped when there is an upstream success, and similarly, how themapblock gets skipped when there is an upstream failure.

`tryCatch`

`map`

📝 Pay attention to the operators you use and understand the conditions under which they may skip execution, and account for negative scenarios effectively in your code.

Second Caveat in Error Handling:

If your publisher throws an error, itterminatesthe chain and doesnotpass control to downstream operators that expect success. As we discussed in the previous caveat, the operators that expect success, only execute if we have success in the upstream publisher.

When an upstream publisher encounters an error, you have several options:

- Emit the error as-is or repackage it into a different error and complete the chain.

Emit the error as-is or repackage it into a different error and complete the chain.

- Replace the error with an output publisher to continue execution.

Replace the error with an output publisher to continue execution.

Understanding which operators allow you to continue execution and which operators completes the chain is crucial for building a robust flow. Below are some of the most commonly used error handling operators:

- mapError— Converts any failure from the upstream publisher into a new error.

mapError— Converts any failure from the upstream publisher into a new error.

`mapError`

- retry— Attempts to retry a failed subscription with the upstream publisher up to a specified number of times.

retry— Attempts to retry a failed subscription with the upstream publisher up to a specified number of times.

`retry`

- catch(andtryvariant) — Handles errors from an upstream publisher by replacing them with another publisher.

catch(andtryvariant) — Handles errors from an upstream publisher by replacing them with another publisher.

`catch`

`try`

- replaceError— Replaces errors from the upstream publisher with a specified output publisher.

replaceError— Replaces errors from the upstream publisher with a specified output publisher.

`replaceError`

### Caveat #3: Threading issues

This is the most often overlooked or misunderstood concept as you shift to Combine paradigm. Combine operators operate on the same thread where the publisher sends the eventsby default.

❗️If you don’t explicitly specify the thread context, you might end up executing code on wrong thread.

We have two operators to determine the execution context of your upstream and downstream publishers. Let’s check how Apple documentation defines them.

- subscribe(on:options:)—Specifies the scheduler on which to perform subscribe, cancel, and request operations. This operator changes the execution context of all your upstream publishers.

subscribe(on:options:)—Specifies the scheduler on which to perform subscribe, cancel, and request operations. This operator changes the execution context of all your upstream publishers.

- receive(on:options:)—In contrast withsubscribe(on:options:), which affects upstream messages,receive(on:options:)changes the execution context of downstream messages. You use thereceive(on:options:)operator to receive results and completion on a specific scheduler, such as performing UI work on the main run loop.

receive(on:options:)—In contrast withsubscribe(on:options:), which affects upstream messages,receive(on:options:)changes the execution context of downstream messages. You use thereceive(on:options:)operator to receive results and completion on a specific scheduler, such as performing UI work on the main run loop.

`subscribe(on:options:)`

`receive(on:options:)`

`receive(on:options:)`

Now that you know whatsubscribe(on:options:)andreceive(on:options:)are, can you determine the output of the code below? We'll figure how non-intuitive this can be for most.

`subscribe(on:options:)`

`receive(on:options:)`

```swift
let numbers = [1]
let cancellable = numbers.publisher
    .map { eachNumber -> Int in
        print("1st map: IsMainThread -- \(Thread.isMainThread)")
        return eachNumber
    }
    .subscribe(on: DispatchQueue.global()) // Upstream publisher runs on global queue
    .map { eachNumber -> Int in
        print("2nd map: IsMainThread -- \(Thread.isMainThread)")
        return eachNumber * 2
    }
    .receive(on: DispatchQueue.main) // Downstream subscribers run on the main thread
    .sink { value in
        print("Received value: \(value) : IsMainThread -- \(Thread.isMainThread)")
    }

----------------------
/// Choose the correct output from the two options below.

/// Option A:
1st map: IsMainThread -- false
2nd map: IsMainThread -- false
Received value: 2 : IsMainThread -- true

/// Option B
1st map: IsMainThread -- true
2nd map: IsMainThread -- false
Received value: 2 : IsMainThread -- true
```

👉 Will the output of the firstmapoperator be on main thread? If you chose option A above, then the next section is for you.

`map`

Upstream vs Downstream:

In a stream, messages can flow from top to bottom, which is intuitive. Most of your values and completion blocks execute top to bottom; this is referred to asdownstream.

However, some values can flow up the chain; this is referred to asupstream.Understanding the publisher lifecycle is crucial to knowing which messages flow upstream and which flow downstream.Let’s add a small snippet of code before subscribe(on:) in the previous example:

```swift
.handleEvents(receiveSubscription: {
        print("handleEvents receiveSubscription: \($0) : IsMainThread -- \(Thread.isMainThread)")
    }, receiveRequest: {
        print("handleEvents receiveRequest: \($0.description) : IsMainThread -- \(Thread.isMainThread)")
    })
```

Now, if you run the code, you’ll notice a change in output — the thread on which the firstmapoperates is no longer the main thread. We get option A mentioned in the previous example as the output.

`map`

```swift
----------------------
/// OUTPUT
handleEvents receiveSubscription: [1] : IsMainThread -- false
handleEvents receiveRequest: unlimited : IsMainThread -- false
1st map: IsMainThread -- false
2nd map: IsMainThread -- false
Received value: 2 : IsMainThread -- true
```

So, what happens here?

- In our initial example,numbers.publisherand themapoperator only emit downstream values. However, when you addhandleEvents, the closures within it demand an upstream movement and gets called synchronously during subscription.

In our initial example,numbers.publisherand themapoperator only emit downstream values. However, when you addhandleEvents, the closures within it demand an upstream movement and gets called synchronously during subscription.

`numbers.publisher`

`map`

`handleEvents`

- We’ve specified using thesubscribe(on:)operator that upstream execution should happen in background context.

We’ve specified using thesubscribe(on:)operator that upstream execution should happen in background context.

`subscribe(on:)`

- Consequently, the emission of values and all subsequent operations, including the firstmapoperator, take place on the background thread specified bysubscribe(on:).

Consequently, the emission of values and all subsequent operations, including the firstmapoperator, take place on the background thread specified bysubscribe(on:).

`map`

`subscribe(on:)`

Now, imagine if you had a task that needs to be performed on the main thread in the firstmapoperator in the above example. This underscores the importance of NEVER assuming the thread on which messages will flow.

`map`

📝Always specify the thread on which you want to receive messages. Usereceive(on:)and assume nothing. Be mindful of where your code executes, as it is easy to overlook this detail.

`receive(on:)`

Listed below are some events that create an upstream flow.

- Subscription— When a subscriber subscribes to a publisher, the subscription message flows upstream to establish the connection.

Subscription— When a subscriber subscribes to a publisher, the subscription message flows upstream to establish the connection.

- Demand —After subscribing, the subscriber can request a certain number of items from the publisher, sending a demand message upstream.

Demand —After subscribing, the subscriber can request a certain number of items from the publisher, sending a demand message upstream.

- Cancellation —If the subscriber decides to stop receiving items, it sends a cancellation message upstream.

Cancellation —If the subscriber decides to stop receiving items, it sends a cancellation message upstream.

### Caveat #4: Long chain of operators

Even experienced developers can fall into the trap of building long chains of operators when deeply immersed in a reactive mindset. This can make your code difficult to read and maintain, and debugging can become a pain.

- Always adhere to the Single Responsibility Principle (SRP). Break down complex chains into smaller, more manageable parts.

Always adhere to the Single Responsibility Principle (SRP). Break down complex chains into smaller, more manageable parts.

- Divide your streams of tasks into smaller blocks.

Divide your streams of tasks into smaller blocks.

- Use intermediate publishers and extensions wherever possible to improve readability.

Use intermediate publishers and extensions wherever possible to improve readability.

### Caveat #5:Avoid assign(to: on:) operator at all times

Theassign(to:on:)operator creates a strong reference to the target object. If the object has a longer lifecycle than the subscription, it can lead to retain cycles and memory leaks. This is especially problematic with view controllers or other UI components that have a defined lifecycle.

`assign(to:on:)`

```swift
final class Counter {
    var count = 0 {
        didSet {
            print("Counter set to: \(count)")
        }
    }
    private var cancellable: AnyCancellable?
    
    init(publisher: CurrentValueSubject<Int, Never>) {
        cancellable = publisher
            .assign(to: \.count, on: self) // This creates a retain cycle
    }
    
    deinit {
        print("Counter deinitialized")
    }
}


let publisher = CurrentValueSubject<Int, Never>(8)

var counter: Counter? = Counter(publisher: publisher)
counter?.count = 5

// Release the reference to counter
counter = nil

----------------------
/// OUTPUT
Counter set to: 8
Counter set to: 5 // Counter is not deinitialized due to the retain cycle
```

Notice how in the above example, thecounterinstance is not deallocated despite setting it tonil. This is due to the retain cycle created by theassign(to:on:)operator.

`counter`

`nil`

`assign(to:on:)`

```swift
counter?.cancellable = nil
```

You can still useassign(to:on:)if you explicitly cancel the subscription at the right time using the assigned value. However, we'll discuss safer alternatives toassign(to:on:)in the next section.

`assign(to:on:)`

`assign(to:on:)`

You can replaceassign(to:on:)with the good oldsinkoperator as given below.

`assign(to:on:)`

`sink`

```swift
cancellable = publisher
      .sink { [weak self] value in
          self?.count = value
      }
```

You can also use operators likemap,filter, andcatchto transform and handle values before assigning them to properties.

`map`

`filter`

`catch`

You can use theassign(to:)variant if your property is marked with@Published. Unlikeassign(to:on:), this variant does not returnAnyCancellableand therefore does not require you to hold a strong reference to it in your class.

`assign(to:)`

`@Published`

`assign(to:on:)`

`AnyCancellable`

```swift
publisher.assign(to: &self.$count)
```

Overall, here are the key considerations to remember when writing your Combine code:

- Memory management and potential retain cycles.

Memory management and potential retain cycles.

- Nuances of error handling.

Nuances of error handling.

- Threading considerations.

Threading considerations.

- Challenges posed by long chains of operators.

Challenges posed by long chains of operators.

- Avoidance of theassign(to:on:)operator.

Avoidance of theassign(to:on:)operator.

`assign(to:on:)`

By understanding and addressing these caveats, you can use Combine more effectively, avoid common pitfalls, and master your skills with Combine.

If you found this helpful, click the 💚 and give it a clap below so others can discover it on Medium. For any questions or suggestions, feel free to leave a comment or reach out to me onTwitterorLinkedIn.

- Apple documentation

Apple documentation

- https://www.marisibrothers.com/2017/04/memory-leak-in-swift-assigning-function.html

https://www.marisibrothers.com/2017/04/memory-leak-in-swift-assigning-function.html

- https://trycombine.com/posts/subscribe-on-receive-on/

https://trycombine.com/posts/subscribe-on-receive-on/

- https://stackoverflow.com/questions/57980476/how-to-prevent-strong-reference-cycles-when-using-apples-new-combine-framework

https://stackoverflow.com/questions/57980476/how-to-prevent-strong-reference-cycles-when-using-apples-new-combine-framework
