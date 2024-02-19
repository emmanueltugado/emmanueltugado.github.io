# A Practical Introduction To `async/await` In `Swift`

Concurrency in the context of programming is the execution of multiple tasks at the same time.
It is the secret recipe behind pleasant user experiences. 

Without concurrency, browsing a product catalogue in your mobile app would be a horrendous experience. The app will pause and wait while images, product information, and prices load sequentially. 

## Concurrency using `async/await`
`Swift 5.5 introduced a new concurrency model through the async/await syntax. This is an improvement over the completion handler syntax and brings massive benefits  such as
1. **Compiler awareness** - the compiler now prevents you from writing asynchronousl logic in a synchronous context. This makes it easier to track and reason about your program's flow
2. **Structured concurrency** - tasks are now hierarchical. Each task has a parent and can spawn child tasks. A parent must wait for all its child tasks to finish before it can be marked itself as completed. Every task is also granted a priority and those with higher priority run first
3. **No need to weakly capture `self`** - since we won't be using closures as completion handlers, we worry be worrying about strong reference cycles as much

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
6. **actor** - reference type that can safely get and set its properties. An actor guarantees that its properties are thread-safe