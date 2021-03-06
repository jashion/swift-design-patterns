# Dependency Injection Design Pattern
`Dependency Injection` is a structural design pattern that is aimed to *decouple* the implementation of a behavior from its interface, or simply it provides means for giving an object its instance variables. 

> Dependency injection means giving an object its instance variables. Really. That’s it.
> 
> [James Shore](https://www.jamesshore.com/Blog/Dependency-Injection-Demystified.html)

The pattern is implemented by making the dependent properties to be passed from outside, that makes acquiring dependencies someone else's problem. It's important to note that it's preferable to use protocols rather than concrete types as *injectables*. That is because using protocols we are able to replace concrete implementation used at run-time by mock objects or subs, which makes tests far more easier and even in some circumstances  just possible. *Dependency Injection* makes coupling more weaker between a type and its dependencies. Another advantage is that a type can be flexibly configured with various injectable dependencies, make a type to be easily reconfigurable without additional compilation.

Concurrently, two or more developers can implement types that implement this pattern: while the first developer implements the type (a.k.a *client*), the other developers implement the *dependencies*. However, the communication protocol between the *client* and its *dependencies* needs to be establish first. The pattern removes all the nitty-gritty details from a client, related to the knowledge of a concrete implementation that it expects. 

The pattern is simple yet powerful. However, it has its own disadvantages, which will be overviewed in a dedicated section later on.

## Initializer Injection
`Initializer` injection is a type of *DI* that passes a dependency into the initializer and captures it for the later use. 

> **Note:**
>
> Please not note that common term for this type of injection is `Constructor Injection`. I used `Initializer` since `Swift` does not have *constructors*, only *initializers*. That was done for the sake of adaptation of the pattern and to avoid confusion with the term (*constructor*) that does not have practical application in `Swift`.

```swift
class DataStorage {

    // MARK: - Properties

    private let dataBase: DataBase

    // MARK: - Initailizers

    init() {
        dataBase = DataBase(schemeType: .queue)
    }
    // ... the rest of the implementation
}
```
In the presented example we have created a class called `DataStorage` with a single property called `dataBase` and a designated initializer that internally instantiates the `dataBase` property. The example may look ok at first, but let's take a look at the `DataBase` class implementation and try to encounter some issues:

```swift
class DataBase {
 
    // MARK: - Properties
    
    var schemeType: PersistenceSchemeType
    
    // MARK: - Initializers
    
    required init(schemeType: PersistenceSchemeType) {
        self.schemeType = schemeType
    }
    
    // MARK: - Methods
    
    func save(serializable: Serializable) {
        print(#function + " serializable is saved!")
    }
    
    func load(_ id: String, completion: @escaping (Serializable) -> Void) {
        print(#function + " serializable for id: ", id, " has been read!")
    }
}
```

`DataBase` class accepts a persistence scheme that describes one of the possible persistence strategies: `queue`, `graph` or `query`. Also the class provides initializer and two methods for setting and getting a `Serializable`. 

The main issue when creating instance of `DataBase` inside the `DataStorage` is that we don't know the details related to the initialization as well as it's very hard to test such a class.

In order to resolve the listed issue we need to make the instance of `dataBase` property to be *injectable*. That is pretty simple to do:

```swift
class DataStorage {
    
    // MARK: - Properties
    
    let dataBase: DataBase
    
    // MARK: - Initailizers
    
    init(dataBase: DataBase) {
        self.dataBase = dataBase
    }
    
    // ... the rest of the implementation
}
```
We just get rid of internal initialization of `dataBase` property, which makes the class more friendly to unit testing, since we are now able to replace the actual `DataBase` class to a *Mock Object* and simulate the desired state and behavior of the *injecting* object. That makes testing more predictable and consistent in terms of test runs. The other benefit is that we removed the code that is responsible to instantiation of `dataBase` property. That is actually a big thing and makes the relationships more loosely-coupled. 

The last issue that we are going to resolve is that we need to change the concrete class for `DataBase` to be a protocol. That change will make the relationship between `DataStorage` and `DataBase` even more loosely-coupled. As a result we will be able to change our `DataBase` to let's say `PropertyList` class, or any other persistence mechanism. 

In order to make the mentioned changes, we start off from declaring a common protocol for our persistence mechanisms:

```swift
protocol Persistence {
    
    // MARK: - Properties
    
    var schemeType: PersistenceSchemeType { get }
    
    // MARK: - Initialisers
    
    init(schemeType: PersistenceSchemeType)
    
    // MARK: - Methods
    
    func save(serializable: Serializable)
    func read(_ id: String, completion: @escaping (_ record: Serializable) -> Void)
}
```
The protocol describes the requirements for every class that is going to be responsible for saving and loading *Serializables*. Let's add conformance of the presented protocol to `DataBase` class and create a new persistence mechanism called `PropertyList`:

```swift
class PropertyList: Persistence {
    
    // MARK: - Properties
    
    var schemeType: PersistenceSchemeType
    
    // MARK: - Initializers
    
    required init(schemeType: PersistenceSchemeType) {
        self.schemeType = schemeType
    }
    
    // MARK: - Methods
    
    func save(serializable: Serializable) {
        print(#function + " serializable is saved!")
    }
    
    func load(_ id: String, completion: @escaping (Serializable) -> Void) {
        print(#function + " serializable for id: ", id, " has been read!")
    }
}
```

The `PropertyList` describes a different  mechanism for storing *Serializables* persistently and makes the `DataStorage` class less dependent on the concrete types and implementations. That is why it's preferable to use protocol as an *injectable* type, rather than concrete implementations: it enables weak-coupling and unit testing possible.

## Property Injection
Property injection a `Dependency Injection` technique, that is similar to the *Initializer Injection*. However, it's less safe, may cause some ambiguities and requires more attention from the developer's perspective. Let me explain why. The main issue with property injection is that the dependency may change during the runtime, which may cause hard to debug issues. That is why we made `dataBase` property *private* and `schemeType` read-only in the `Initializer Injection` section. 

Let's examine the following situation: 

```swift
class DataStorage {
    
    // MARK: - Properties
    
    // We have changed the accessor modifier from *private* to *internal* and made it variable instead of constant, so we will be able to change the dependency later on
    var dataBase: Persistence
    
    // MARK: - Initializers
    
    init(dataBase: Persistence) {
        self.dataBase = dataBase
    }
    
    // ... the rest of the implementation
}

// 1
let dataBase = DataBase(schemeType: .graph)
let storage = DataStorage(dataBase: dataBase)

// 2
let propertyList = PropertyList(schemeType: .query)
storage.dataBase = propertyList

// 3
storage.save(data)

// 4
storage.load(data)
```

The first thing that we have done is we slightly changed the implementation of `DataStorage` class: we made `dataBase` property *internal* and *variable*, rather than *private* and *constant*. Then we passed the `DataBase` instance as a dependency using the initializer. Later on, during the runtime of an app, another developer from our team decided to change the dependency to `PropertyList`. The third developer *saved* and then *loaded* the big chunk of data and as it turned out, the `PropertyList` type of storage, was not suitable for that kind of data. As a result the app, on some configurations started showing some significant performance issues. That is one of practical examples that actually occurred with me a while ago. You may extrapolate and think about the other dangers. 

That is why you need to very careful with *property injection* and think twice before implementing them. However, I'm not saying that are pure evil, in some cases they may be pretty suitable, for instance when there is additional logic that prevents from resetting the dependency based on business-related logic. 

## Method Injection
Method Injection is an another type of *Dependency Injection* that is aimed to inject a dependency using methods. It either can be a local method dependency or a type dependency that is injected through method. Method injection is useful in cases when a method uses an external dependency and we need to decouple it from the caller. The other benefit is that your method can have a different dependency each call - that may add some dynamism in terms of execution logic, but it also may be a source of issues. The method calling complexity is increased as well, just be careful. 

## Ambient Context
There yet another type of `DI` called `Ambient Context`. It will not be described here, it was included so will be able to reference it and make own research. Maybe later on I will create a good illustration of this type of `DI` and include it here.

<!--## Storyboards -->

<!--## View Controllers-->

<!--## Testability
This subsection may be implemented later on. -->

## Disadvantages
`Dependency Injection` pattern has its own disadvantages as well. Even though it seems good to decouple the implementation of a behavior from its interface, it can make difficult to read code because  the behavior and construction parts are separated. The number of files grows, which may increase the overall complexity from a perspective of a new developer or even existing developer who wants to reference those files.

Another issue is that not every `IDE` can reference and trace the *dependencies* when refactoring. Also, in some cases the details related to the construction of a type may not be separated from its behavior - it may be a root of security issues or simply may not be easily managed outside of the target type. Decoupled dependencies may also require an 3rd party framework for dependency management, which may not suit your requirements. 

Also, in some cases dependencies may not be `volatile`, which means they don't need to be injected. Those types of dependencies are called `stable` and should not be exposed for injection.

The last thing that I want to mention is that don't create dependency injections between modules. That will make *DI* act as anti-pattern, since by trying to make code loosely coupled, you will actually make it tightly coupled with an another module. 

All the above leads to a simple conclusion: be mindful and don't hurry to apply *Dependency Injection* pattern.


## Conclusion
`Dependency Injection` is a simple yet powerful pattern that helps to structure code, make it more loosely-coupled, testable and configurable. Another big benefit is that it enables two or more developers to work on the same type that has `volatile` dependencies. It's easy to implement and refactor existing code-base to adopt this pattern. However, the pattern may also raise some issues: refactoring on an `IDE` level becomes very hard, more granular structure makes it harder to navigate and just simply understand what is going on in a project (especially for a new developer). Just be careful of typical pitfalls and think twice before adopting this pattern.