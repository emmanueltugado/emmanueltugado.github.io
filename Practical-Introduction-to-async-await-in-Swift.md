# A Practical Introduction To `async/await` In `Swift`

Concurrency in the context of programming is the execution of multiple tasks at the same time.
It is the secret recipe behind pleasant user experiences. 

Without concurrency, browsing a product catalogue in our mobile app would be a horrendous experience. The app will pause and wait while images, product information, and prices load sequentially. 

## Concurrency using `async/await`
`Swift 5.5 introduced a new concurrency model through the async/await syntax. This is an improvement over the completion handler syntax and brings massive benefits  such as
1. **Compiler awareness** - the compiler now prevents you from writing asynchronousl logic in a synchronous context. This makes it easier to track and reason about your program's flow
2. **Structured concurrency** - tasks are now hierarchical. Each task has a parent and can spawn child tasks. A parent must wait for all its child tasks to finish before it can be marked itself as completed. Every task is also granted a priority and those with higher priority run first
3. **Reduced need to weakly capture `self`** - since we won't be using closures as completion handlers, we worry be worrying about strong reference cycles as much

## `async/await` vs closure-based completion handlers
Prior to `async/await`, you would fetch data from your server by using a method similar to this:
```
func getDataFromServer(
    completionHandler: @escaping (Data?) -> Void
) {
    URLSession.shared.dataTask(
        with: URL(string: "https://www.myserver.com")!
    ) { data, response, error in
        completionHandler(data)
    }
    .resume()
}

// call the method
getDataFromServer { [weak self] data in
    guard let self else { return }

    self.useFetchedData(data)
}
```
There are no assurances that `getDataFromServer` called the completion handler or that the error was handled. Forgetting to weakly capture `self` can also result in reference cycles.

Rewriting the same logic using the new `async/await` format gives us this:
```
func getDataFromServer() async throws -> Data? {
    let (data, response) = try await URLSession.shared.dat(
        from: URL(string: "https://www.myserver.com")!
    )
    return data
}

// call the method
let fetchedData = try await getDataFromServer()
getDataFromServer(fetchedData)
```

There is no chance of a reference cycle here and the logical flow is easier to follow.

## The basics
These are the types and modifiers we'll work with 
1. **async** - used to mark a method as asynchronous. This can be applied to functions, closures, and getters of computed properties
```
func asyncMethod() async {}

var asyncClosure: ((String) async -> String)

var asyncComputedProperty: String {
    get async { await asyncMethod() }
}

```
2. **await** - used to call an `async`' method. A possible suspension point to wait until the `async` returns
3. **Task** - a unit of asynchronous work. Takes a closure containing the work to execute. Can be called from a synchronous context (eg. `View`'s `.onAppear` )
4. **async-let** - syntax used to create concurrent asynchronous tasks. Starts executing once created
5. **TaskGroup** - used to group asynchronous tasks. Tasks are ran concurrently, awaited as one, and the results collected cumulatively
6. **Actor** - reference type that can safely get and set its properties. An `actor`` guarantees that its properties are thread-safe

## Using async-await
Adding the `async` keyword before a method's return type marks that function as asychronous. Any calls to the method must happen in an asynchronous context
```
func asyncMethod() async -> String {
    return "ABC"
}
```

We use the `await` keyword to call methods marked with `async`. Note that calling `await` inside another function requires said function to have `async` as well.
```
// this must now have `async` as well
func awaitAnAsyncMethod() async {
    let returnedValue = await asyncMethod()
    print(returnedValue)
}
```

## Running `async` tasks from a synchronous context
A method signature **without** the `async` keyword is an example of a synchronous context. Two of the most common synchronous contexts in `SwiftUI` `View`s are 
1. `onAppear` - signature is `onAppear(perform: (() -> Void)?) -> View`
2. `onDisappear` - signature is `onDisappear(perform: (() -> Void)?) -> View`

Calling an `async` method from these functions will result in a compiler error:
```
func someAsyncMethod() async {}

SomeView {}
.onAppear {
    await someAsyncMethod()
}
```

> Invalid conversion from 'async' function of type '() async -> ()' to synchronous function type '() -> Void'

Wrapping your `async` code inside a `Task` solves this problem.
```
func someAsyncMethod() async {}

SomeView {}
.onAppear {
    Task {
        await someAsyncMethod()
    }
}
```

A `Task` is a `Struct` with methods such as `cancel()`. This is convenient for cleanup like stopping downloads when a `View` disappears.
```
@State private var longRunningDownload: Task<Void, Never>?

SomeView {}
.onAppear {
    longRunningDownload = Task {
        await someAsyncMethod()
    }
}
.onDisappear {
    // clean up here
    downlongRunningDownloadloadTask?.cancel()
}
```

`Apple` provides us with a convenient `View` modifier, `.task`, for running `async` tasks when a `View` appears.

```
func someAsyncMethod() async {}

// This achieves the same outcome as the `onAppear` call below
View {}
.task {
    await someAsyncMethod()
}

View {}
.onAppear {
    Task {
        await someAsyncMethod()
    }
}
```
## Making sequential asynchronous calls
Each `await` is a suspension point. Our program will halt and wait for the method marked with `async` to resolve before moving forward.

```
func firstAsyncMethod() async {}

// execution will suspend here
await firstAsyncMethod()
// before moving here
print("I finished waiting")
```

This lets us chain asynchronous calls easily:
```
func firstAsyncMethod() async -> String {}
func secondAsyncMethod(data: String) async {}

let firstValue = await firstAsyncMethod()
secondValue = await secondAsyncMethod(data: firstValue)
```

## Concurrent asynchronous calls using `async-let`
What if we want to call multiple `async` methods in parallel?
```
func firstAsyncMethod() async -> String {}
func secondAsyncMethod() async -> String {}
```

`async-let` to the rescue! 
```
async let firstMethodCall = firstAsyncMethod()
async let secondMethodCall = secondAsyncMethod()

// COLLECT the results in a single `await``
let (firstResult, secondResult) = await (firstMethodCall, secondMethodCall)
```

Note that the methods fire **immediately** once we use `async-let` - they don't "wait" for an `await`.

## A better way to make concurrent asynchronous calls
`async-let` is awesome if we know the exact number of concurrent calls to make. But what if there are numerous asynchronous tasks to fire and we don't know the volume beforehand?

This is where `TaskGroup` shines. Think of it as a more scalable version of `async-let`.

To use `TaskGroup` we must:
1. create a `TaskGroup` using the `withTaskGroup` or `withThrowingTaskGroup` methods
2. add tasks to the group by calling `addTask`
3. ensure that each task added **produces** the same return type

```
func asyncReturningString() async -> String { "ABC" }
func asyncReturningAnotherString() async -> String { "DEF" }
func asyncReturningInteger() async -> Int { 1 }

await withTaskGroup(
    of: String.self, 
    returning: [String].self
) { taskGroup in
    // we add tasks individually
    taskGroup.addTask {
        await asyncReturningString()
    }
    taskGroup.addTask {
        await asyncReturningAnotherString()
    }
    
    // this `async` task returns an Integer 
    // but we are required to produce a String
    taskGroup.addTask {
        String(await asyncReturningInteger())
    }
    
    var combinedResults: [String] = []
    // we can iterate over the contents of our TaskGroup
    for await result in taskGroup {
        combinedResults.append(result.capitalized)
    }
    
    return combinedResults
}
```

**Note that all tasks must complete before the group returns.**

An example of requesting a dynamic number of images can look like this

```
func imageURLsToFetch() async throws -> [URL] {
    // hit an API endpoint and decode the response as [URL]
}

func imageFromURL(url: URL) async throws -> Data {
    // fetch and decode a URL as Data
}

let images = try await withThrowingTaskGroup(
    of: Data.self, 
    returning: [UIImage].self
) { taskGroup in
    let imageURLs = try await urlsToFetch()

    for url in imageURLs {
        // add each `Data` object representing
        // a `UIImage` to our `TaskGroup``
        taskGroup.addTask { 
            try await downloadImageFrom(url: url) 
        }
    }

    var images: [UIImage] = []
    for try await imageData in taskGroup {
        guard let image = UIImage(data: imageData) else { continue }
        
        images.append(image)
    }
    
    return images
}
```

We use `withThrowingTaskGroup` to create a `TaskGroup` that can throw an exception. 

To iterate over its contents, we use `try await` in our `for-loop`: `for try await imageData in taskGroup`.

## Actors and atomicity
A Swift `actor` is a reference type that guarantees **thread-safe (atomic)** access and mutation of its properties.

Different threads concurrently mutating and accessing a property can lead to race conditions and unexpected behavior. The `actor` type solves this [readers-writers problem](https://en.wikipedia.org/wiki/Readers%E2%80%93writers_problem). 

The class below looks harmless but if multiple threads attempt to read and write to the array, a crash can happen.
```
class UnsafeClass {
    private var unsafeArray: [Int] = []
    
    func insert(number: Int) {
        unsafeArray.append(number)
    }
    
    func getNumbers() -> [Int] {
        return unsafeArray
    }
}
```

Converting this unsafe class to a thread-safe `actor`` is simple.

```
actor SafeActor {
    private var safeArray: [Int] = []
    
    func insert(number: Int) {
        safeArray.append(number)
    }
    
    func getNumbers() -> [Int] {
        return safeArray
    }
}
```

The compiler will now make sure that we are calling our `actor`'s methods from a concurrency-aware context.

```
let actor = SafeActor()

await actor.insert(number: 1)

let numbers = await actor.getNumbers()
```

***Thread-safety is ensured because `actor`s use a **serial** executor to schedule the mutation and access of its properties.***

## Global actors
A **global** `actor` is a singleton of an `actor` type that you can access anywhere in your app. This is useful for synchronising global state such as persistence caches.

Creating a global actor is straightforward. All you need to do is
1. annotate your `actor` with `@globalActor`
2. create a static property named `shared` of your actor's instance

```
@globalActor actor ImageCache {
    `GlobaActor` protocol requirement
    static let shared = ImageCache()

    private var savedImages: [UIImage] = []

    func fetchImage(name: String) -> UIImage? {}
    func saveImage(name: String, image: UIImage) {}
}
```

Creating our own global `actor` unlocks a new annotation, @OurActorName, that we can apply to methods or entire classes. Using this annotation ensures that the methods or the class runs on that global `actor`'s serial queue.

```
class ImageProcessor {
    // this method now runs on `ImageCache`'s
    // serial executor
    @ImageCache 
    func processAndSave(image: UIImage, filename: String) async {
        // download logic
        await ImageCache.shared.saveImage(name: filename, image: image)
    }
}

// you can even annotate the entire class
// ensuring all methods of that class
// run on `ImageCache`'s serial executor
@ImageCache 
class ImageProcessor {}
```

## Running changes on the Main thread
UI changes must run on the Main thread. The new concurrency framework gives us a global `actor` specifically for this purpose: `MainActor`. Any method or class annotated with `@MainActor` runs on the Main thread. 

This is useful for `ObservedObject` instances that directly influence the UI.

```
@MainActor
class SomeViewModel: ObservableObject {
    @Published var titles: [String] = []
}

struct SomeContentView: View {
    @ObservedObject var viewModel = SomeViewModel()
    
    var body: some View {
        VStack {
            ForEach(viewModel.titles, id: \.self) {
                Text($0)
            }
        }
    }
}
```
## Unit testing
The same rules apply for test methods: add `async` to the test method's signature and use `await` to call your function.
```
class SomeClass {
    func someAsyncMethod() async -> String {
        return "ABC"
    }
}

// `XCTestCase` method
func test_someAsyncMethod_returnsUppercaseString() async {
    let sut = SomeClass()
    let asyncMethodResult = await sut.someAsyncMethod()
    
    XCTAssertEqual(asyncMethodResult, "ABC")
}
```
## Bridging closure-based legacy code
Legacy asynchronous code relied heavily on closures as completion handlers. 

```
class LegacyClass {
    func legacyMethod(completion: @escaping (String) -> Void) {
        completion("Old Method")
    }
}
```

We will use `continuation`s to bridge this to the new `async/await` model.

```
// 1
let completionString: String = await withCheckedContinuation { continuation in
    // 2
    legacyMethod { legacyResult in
        // 3
        continuation.resume(with: .success(legacyResult))
    }
}
```

There are three steps:
1. Wrap the legacy method in a `continuation` using `withCheckedContinuation` or `withCheckedThrowingContinuation` 
2. Call the legacy method
3. Ensure that the `resume()` method is called exactly once on your `continuation`. Failing to do so results in Xcode prompting you with
> SWIFT TASK CONTINUATION MISUSE: newConcurrencyMethod() leaked its continuation! 