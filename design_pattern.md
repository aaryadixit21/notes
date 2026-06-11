##### 1)Creational design pattern



Creational Design Patterns are focused on handling object creation mechanisms where objects are created in a manner suitable for the situation. In traditional object-oriented programming, instantiation involves using constructors directly within the client code. However, this can lead to tight coupling between the class implementation and the client.



Creational patterns solve this problem by abstracting the process of instance creation so that the system can remain independent of how its objects are defined, created, and represented.





Types:



1. ###### Factory design pattern



According to Gang of Four (GoF), a factory is an object used to create other objects. In technical terms, a factory is a class with a method. That method creates and returns different objects based on the received input parameter.

The basic principle behind the Factory Design Pattern is that, at run time, we get an object of a similar type based on the parameter we pass. So, the client will get the appropriate object and consume the object without knowing how the object is created and initialized.



loose coupling, open closed principle





* by using IF-ELSE conditions -- tight coupling (we need to keep the client code free from any changes)
* by creating factory class in which if else is written, which generates the particular obj



The following are some specific scenarios where the Factory Design Pattern can be beneficial in real-time applications:



* **Complex Creation Logic**: When the creation of an object is complex and requires more than simple instantiation. The Factory pattern can encapsulate this complexity.
* **Similar Object Families**: When dealing with families of related or dependent objects designed to be used together, you need a way to create these objects so that they can easily be swapped with other families without modifying the client code.
* **Subclass Selection**: When you need to support the dynamic instantiation of one or more derived classes from a common base class or interface based on the input or configuration data provided.
* **System Scalability**: As real-time applications grow, new object types can be added with minimal changes to the existing application code. Factories can handle the creation logic for these new types without altering existing code, enhancing the scalability of the application.
* **Conditional Object Creation**: When the instantiation of an object depends on certain external conditions or configuration settings, and you want to abstract these conditions from the main application logic.
* **Centralized Object Management**: When you need to centralize the control of object creation to ensure consistency in behavior or configuration among the instances that are produced.





examples: credit card, cloud storage system





###### factory method design pattern



the Factory Method Design Pattern Defines an interface for creating an object but lets the subclasses decide which class to instantiate. The Factory method lets a class defer instantiation to subclasses.



credit card interface and credit card factory abstract class, sub class for factories also. sub class creates the obj of particular class

client code calls the factories class method to create objs



objs creation code written inside factory



Advantages of the Factory Method Design Pattern:



* **Loose Coupling:** The pattern promotes loose coupling by separating object creation from its usage. This means the client code does not need to know the specific class that needs to be instantiated.
* **Single Responsibility Principle**: It adheres to the Single Responsibility Principle by separating the creation and business logic, making the system easier to manage and extend.
* **Open/Closed Principle:** The pattern supports the Open/Closed Principle. It’s easy to introduce new types of products without changing the existing client code.
* **Flexibility and Extensibility**: It provides more flexibility in code since it delegates the instantiation logic to subclasses. This makes the system more adaptable and extendable.
* **Ease of Testing**: The pattern simplifies unit testing as it allows for the use of mock objects or stubs in place of actual implementations.









###### 2\. Abstract factory design pattern



The Abstract Factory Design Pattern provides a way to encapsulate a group of factories with a common theme without specifying their concrete classes.



Abstract Factory Pattern is a software design pattern that provides a way to encapsulate a group of individual factories that have a common theme.



In simple words, the Abstract Factory is a super factory that creates other factories. It is also called the Factory of Factories. The Abstract Factory design pattern provides an interface for creating families of related or dependent products but leaves the actual object creation to the concrete factory classes.





diff btw abstract factory and factory method
Level of Abstraction:

Abstract Factory provides an interface for creating families of related or dependent objects without specifying their concrete classes.

Factory Method is a method in a class that creates objects, and its subclasses decide which class to instantiate.



Purpose:

Abstract Factory is used when your system needs to create multiple families of products, or you need a factory of factories.

Factory Method is used when one product has multiple implementations, and the client does not need to know which it uses.









###### 3\. Builder design pattern



Builder Design Pattern builds a complex object using many simple objects and a step-by-step approach. The Process of constructing the complex object should be so generic that the same construction process can be used to create different representations of the same complex object.



it is about separating the construction process of a complex object from its representation, allowing the same construction process to create different representations. When the construction process of the object is very complex, we only need to use the Builder Design Pattern.







adv



* **Separation of Construction and Representation:** The Builder pattern separates the construction process of an object from its representation, allowing the same construction process to create different representations. This separation enhances the modularity of the code.
* **Encapsulation of Complex Creation Logic**: The pattern encapsulates complex object creation logic within the builder, making the client code cleaner and simpler. This is particularly useful when creating objects with numerous initialization parameters, some of which may be optional.
* **Improved Readability and Maintainability**: Using the Builder pattern, you can avoid telescoping constructor anti-patterns (where a constructor may have many parameters, not all of which are always needed). This improves code readability and maintainability.
* **Immutability of Objects**: The Builder pattern can be used to build immutable objects (objects whose state cannot change after they are constructed). Immutable objects are simpler to understand and work with and are also thread-safe.
* **Control Over Object Construction Process:** The pattern provides more control over the construction process than direct instantiation. It allows building an object step-by-step, and the product is only returned when it’s entirely ready.
* **Fluent Interface and Method Chaining:** Many builder implementations use fluent interfaces and method chaining, making client code more readable and easily understood.
* **Flexibility in Object Instantiation**: Builders can construct complex objects that would require a lot of setup code to create otherwise. They can also instantiate subclasses of an immutable class.
* **Ease of Adding New Parameters:** Adding new parameters to the object’s constructor through the builder is easy and does not require changing the constructor signature, thus not affecting existing client code.





componenets

* Abstract Builder: The Builder is an interface defining all the steps to make the concrete product.
* Concrete Builder: The Concrete Builder Classes implement the Abstract Builder interface and provide implementation to all the abstract methods. By implementing the builder interface, the concrete builder is responsible for constructing and assembling the individual parts of the product.
* Director: The Director takes those individual processes from the Builder and defines the sequence to build the product.
* Product: The Product is a class, and we want to create this product object using the builder design pattern. This class defines different parts that will make the product.



client calls director





###### 3\. Fluent interface design pattern



The Fluent Interface Design Pattern in C# is a method for designing object-oriented APIs that provides more readable and easily maintainable code. It is based on the concepts of method chaining, where each method returns an object itself, allowing multiple actions to be invoked in a single statement sequence. It is useful for creating an API where the code reads like a series of natural language statements, enhancing readability and ease of use.



Method Chaining: Each method returns an instance of the object itself, typically this, which allows for the chaining of multiple method calls in a single statement.





Method Chaining in C# is a common technique where each method returns an object, and all these methods can be chained together to form a single statement.





###### 4\. Prototype design pattern



Prototype Design Pattern specifies the kind of objects to create using a prototypical instance and creates new objects by copying this prototype. it creates objects by copying an existing object, known as the prototype.



When we talk about Object Cloning in C#, it means it is all about the Call By Value. So, if we change one object, it will not affect the other. Let us see how to clone the object to another object. To do so, C# provides one method called "MemberwiseClone" , which will create a new complete copy of the object with a different memory.



use case:

it is expensive to create one obj, and you want new obj with minor changes.

private attributes bhi clone ho skte hain, which are not accessible from outside.



shallow copy



Value Types: Each value-type field is copied directly. Value types (such as integers and structs) store data directly, so a direct value copy is made.

Reference Types: For reference types fields, the reference is copied but not the underlying object. Thus, the original object and its copy will refer to the same object.





deep copy



Value Types: Like with shallow copies, value-type fields are copied directly.

Reference Types: Instead of copying just the reference, a new copy of the referred object is created. Each reference in the original object leads to the creation of a new object in the copy.



deep copy is done in prototype dp by cloning.





###### 5\. Singleton design pattern



a class has only one instance and provides a global point of access to it. In C#, the Singleton Design Pattern is useful when we need exactly one instance of a class to coordinate actions across the system. It is commonly used in scenarios where multiple objects need to access a single shared resource, such as configuration settings, access to a shared resource like a file system, or managing a connection to a database.



Key Characteristics of the Singleton Design Pattern in C#

* Single Instance: This Design Pattern ensures that only one instance of the Singleton class is created throughout the application.
* Global Access: Provides a global access point to that instance.
* Lazy Initialization: We can lazily initialize the singleton instance, which means it is created when it is needed for the first time, not when the application starts.
* Thread Safety: When used in a multi-threaded application, the singleton needs to be thread-safe to prevent multiple instances from being created.





The following are the guidelines to implement the Singleton Design Pattern in C#.



* Private Parameterless Constructor: We need to create a private and parameterless constructor. This is required because it will restrict the class from being instantiated from outside the class; it will only instantiate from within the class.
* Sealed Class: The Singleton class should be declared sealed to ensure it cannot be inherited.
* Static Variable: We need to create a private static variable that holds the single instance of the class.
* Public Static Method or Property: We also need to create a public static property or method that will return the singleton instance. This method or property first checks whether the singleton class instance has been created. If the singleton instance is created, it returns that instance; otherwise, it will create an instance and then return it. This static method or property provides the global point of access to the singleton instance and ensures that only one instance of the class is created.



4 ways to achieve it

\-eager initialization

\-lazy loading

\-synchronized method

\-double locking



The following are some of the real-time scenarios where using the Singleton Design Pattern can be beneficial:



* Shared Resources Management: For managing shared resources such as database connection pools or configuration data, where only one instance should manage the resource throughout the system.
* Logging: In scenarios where an application-wide logger instance needs to capture logs from various parts of an application, ensuring that the logging mechanism is consistently used.
* Caching: When you need to cache application data so it’s accessible globally and maintained within a single object, ensuring consistency.
* Controlled Access and Operations: When you need to perform operations where exactly one instance is needed to coordinate actions across the system, for example, a logging class or managing access to a value shared across various parts of an application.





The following are the reasons why Singleton classes are often sealed in C#:



* Preventing Inheritance: The Singleton pattern ensures that only one instance of a class exists throughout the application. If inheritance is allowed, a subclass can create additional instances, violating the Singleton Design Pattern. Making the class sealed ensures that no subclass can override methods or properties that could lead to additional instance creation.
* Ensuring Singleton Integrity: By declaring the class sealed, we ensure the integrity of the Singleton instance. If the class was not sealed, a derived class could add new static members, introduce a new access point to the Singleton instance, or even create new instances independently of the original Singleton control mechanism.
* Simplifying Thread Safety: Implementing thread safety in a Singleton can be challenging, especially in multi-threaded environments. Making the class sealed simplifies the synchronization logic, as we only need to consider the Singleton class itself and not any subclasses that might alter the behavior of the Singleton instance.
* Optimization: The compiler can make certain optimizations, knowing that the class cannot be inherited. For example, method calls to members of a sealed class can sometimes be faster because the method dispatching process is simpler and can be more efficiently optimized during JIT (Just-In-Time) compilation.







In a non-thread-safe implementation, if multiple threads access the Singleton instance simultaneously at the point when the instance has not yet been created, each thread could end up creating a new instance. This violates the core principle of the Singleton pattern, which is only to allow one instance of the class.





There are many ways to Implement the Thread-Safe Singleton Design Pattern in C#. They are as follows:



* Thread-Safety Singleton Implementation using Lock.

&#x20;     	slow as only one thread enters

* Implementing Thread-Safety Singleton Design Pattern using Double-Check Locking.
* Using Eager Loading to Implement Thread-Safety Singleton Design Pattern

&#x09;most effective

* Using Lazy<T> Generic Class to Implement Lazy Loading in Singleton Design Pattern.









thread safe singleton using locks



public static Singleton GetInstance()

&#x20;       {

&#x20;           //This is thread-safe

&#x20;           //As long as one thread locks the resource, no other thread can access the resource

&#x20;           //As long as one thread enters into the Critical Section,

&#x20;           //no other threads are allowed to enter the critical section

&#x20;           lock (Instancelock)

&#x20;           { //Critical Section Start

&#x20;               if (Instance == null)

&#x20;               {

&#x20;                   Instance = new Singleton();

&#x20;               }

&#x20;           } //Critical Section End

&#x20;           //Once the thread releases the lock, the other thread allows entering into the critical section

&#x20;           //But only one thread is allowed to enter the critical section

&#x20;           //Return the Singleton Instance

&#x20;           return Instance;

&#x20;       }









double checking locking



The first check is to see if the instance is null before acquiring the lock.

The second check is performed after the lock is acquired to ensure that no other thread created the instance in the meantime.





public static Singleton GetInstance()

&#x20;       {

&#x20;           //This is thread-Safe

&#x20;           if (Instance == null)

&#x20;           {

&#x20;               //As long as one thread locks the resource, no other thread can access the resource

&#x20;               //As long as one thread enters into the Critical Section,

&#x20;               //no other threads are allowed to enter the critical section

&#x20;               lock (Instancelock)

&#x20;               { //Critical Section Start

&#x20;                   if (Instance == null)

&#x20;                   {

&#x20;                       Instance = new Singleton();

&#x20;                   }

&#x20;               } //Critical Section End

&#x20;               //Once the thread releases the lock, the other thread allows entering into the critical section

&#x20;               //But only one thread is allowed to enter the critical section

&#x20;           }

&#x20;

&#x20;           //Return the Singleton Instance

&#x20;           return Instance;

&#x20;       }







lazy loading



no double locks



The following are the key features of the Lazy<T> type:



* Thread Safety: By default, Lazy<T> is thread-safe. It uses locks internally to ensure that the value is initialized only once.
* Lazy Initialization: The instance is not created until it is actually needed, which can improve the application’s startup time and overall memory usage efficiency.
* Simple and Clean: The code is cleaner and easier to maintain, as the complexity of thread management is abstracted away by the Lazy<T> type.





// The private static instance ensures lazy initialization.

&#x20;       private static readonly Lazy<Singleton> instance = new Lazy<Singleton>(() => new Singleton());

&#x20;       // Private constructor to prevent direct instantiation.

&#x20;       private Singleton()

&#x20;       {

&#x20;           // Initialization code here

&#x20;           //Each Time the Constructor is called, increment the Counter value by 1

&#x20;           Counter++;

&#x20;           Console.WriteLine("Counter Value " + Counter.ToString());

&#x20;       }

&#x20;       //The following Static Method is going to return the Singleton Instance

&#x20;       public static Singleton GetInstance()

&#x20;       {

&#x20;           return instance.Value;

&#x20;       }





* Lazy<T> Initialization: The Singleton instance is wrapped in a Lazy<T> object, which is initialized with a delegate (() => new Singleton()) that points to the private constructor of the Singleton. The Singleton instance is created the first time Lazy<T>.Value is accessed.
* Thread Safety: The Lazy<T> class by default ensures thread safety, which means no matter how many threads try to access the Singleton.GetInstance() method simultaneously, the Singleton constructor is executed only once, and all threads will receive the same instance.
* Performance: This method is very efficient because after the instance is created, accessing the Singleton.GetInstance() method does not incur any lock-checking overhead typically associated with other thread-safe approaches.









Early loading

Eager loading creates the Singleton instance when the class is loaded or before it’s needed. This ensures that the instance is available immediately when requested, but it may consume resources even if it’s never used during the application’s lifetime.



The advantage of using Eager Loading in the Singleton Design Pattern is that the CLR (Common Language Runtime) will take care of Object Initialization and Thread Safety in Multithread Environment. That means we will not be required to write any code explicitly for handling the thread safety for a multithreaded environment.



by making the singleton obj static







Lazy loading

lazy loading defers the creation of the Singleton instance until it is needed. This can be beneficial if the initialization of the instance is expensive or if you want to save resources by not creating the instance until it’s necessary.







Use lazy loading when:

* Resource Optimization: If the Singleton instance is heavy and consumes significant resources, lazy loading ensures that these resources are not utilized until absolutely necessary. This is useful if there’s a chance that the instance might not be needed at all during the application’s runtime.
* Start-up Performance: In applications where start-up time is crucial, lazy loading can help reduce the start-up load by deferring the creation of heavy objects to a later point. This is common in desktop applications where initial responsiveness is critical.
* Conditional Initialization: If your application has multiple execution paths and the Singleton is not required for every path, lazy loading can prevent unnecessary initialization, thus saving resources.





Use eager loading when:

* Predictability: Eager loading contributes to the predictable behavior of the application by initializing the Singleton instance during application startup. Since initialization sequences are consistent, this can simplify debugging and behavior analysis.
* Concurrency Simplification: In multi-threaded applications, eager loading can simplify the design by avoiding the need for synchronization mechanisms required for safely lazy loading the instance. Once the instance is created during startup, it can be accessed by multiple threads without additional overhead or complexity.
* Performance Critical Situations: If the Singleton is used in performance-critical parts of the application and must be accessed quickly without delay, having it already created and available (eager loading) ensures that there is no delay in accessing the instance when needed.





Singleton versus Static



similarities between Singleton vs Static Class in C#



* Static Class and Singleton Class can have only one instance available in memory throughout the application.
* They both hold the global state of an application that will be common for all clients.
* Both Static Classes and Singleton Classes can be implemented as Thread-Safe.





differences

The most important point you need to remember is that Static is a language feature, whereas Singleton is a Design Pattern.





* We cannot create an instance of a static class in C#. Yes, one copy of the static class is available in memory, but we cannot create an instance of the static class as a developer. But, as a developer, we can create a single instance of a singleton class, and then we can reuse that singleton instance at many different places in the application.
* When the compiler compiles the static class internally, it treats it as an Abstract and Sealed Class in C#. This is why we neither create an instance nor use the static class as the child class in inheritance. On the other hand, we can create a single instance of the Singleton class, as we can also use the Singleton class as a child class in C#.
* The Singleton Class Constructor is always marked as private. This is why we cannot create an instance outside the singleton class. It provides either a public static property or a public static method whose job is to create the singleton instance only once and then return that singleton instance each and every time we call that public static property/method from outside of the singleton class.
* A Singleton class can be initialized lazily or loaded automatically by CLR (Common Language Runtime) when the program or namespace containing the Singleton class is loaded, i.e., Eager Loading. A static class is generally initialized when it is loaded for the first time, and this may lead to potential classloader issues.
* It is impossible to pass the static class as a method parameter in C #, whereas we can pass the singleton instance as a method parameter in C#.
* In C#, it allows inheritance with the Singleton class. The Singleton class can be created as a Child class only. You cannot create child classes from the Singleton class. These are not possible with a static class. So, the Singleton class is more flexible than the Static Classes in C#.











**Memory Management of Static Class vs Singleton Class in C#:**

When the class is loaded, memory for static classes is allocated once in the high-frequency heap (a special heap for static data). This memory allocation is fixed and exists for the lifetime of the application. Since static classes are not instantiated and their resources are allocated directly in the memory, they live throughout the application life cycle and are only cleaned up when the application domain unloads, or the application exits.



Memory for a singleton instance is allocated on the heap when the instance is created, usually the first time it is requested. This memory remains allocated as long as there is a reference to the instance. The memory for the singleton instance is allocated when the instance is created and normally lasts for the duration of the application. However, unlike static classes, it’s possible to implement patterns like using a weak reference for the singleton instance, which allows for the instance to be garbage collected if there are no more references to it.







Use Singleton when:

* You might need to switch from a single instance to multiple instances in the future without changing the API.
* You require lazy initialization or initialization upon first access.
* The class needs to maintain its state for its lifetime.
* You want the class to implement interfaces or manage inheritance hierarchies.



Use Static Class when:

* You are creating utility functions that do not store any state.
* You want a simple and straightforward mechanism to provide global access without any initialization logic.
* You need constants, static configuration settings, or methods that are logically grouped together but do not need to interact with instance data.







##### 2)Structural design pattern



The Structural Design Patterns simplify the design by identifying a simple way to manage the relationships between entities. They allow developers to obtain new functionalities by composing objects and classes. They focus on how classes inherit from each other and how they are composed from other classes.





1. ###### Adapter design pattern



structural pattern that allows objects with incompatible interfaces to work together. It acts as a bridge between two incompatible interfaces. This pattern is useful when you want to use existing classes, but their interfaces do not match the one you need.

The Adapter Design Pattern acts as a bridge between two incompatible objects. Let’s say the first object is A and the second object is B. Object A wants to consume some of object B’s services. However, these two objects are incompatible and cannot communicate directly.



types:



* Object Adapter Pattern   (adapter has a object of adaptee)
* Class Adapter Pattern    (adapter is child of adaptee)





The Adapter Design Pattern is composed of four components. They are as follows:



* Client: The Client class can only see the ITarget interface, i.e., the class that implements the ITarget interface, i.e., the Adapter (in our example, it is the EmployeeAdapter). Using that Adapter (EmployeeAdapter) object, the client will communicate with the Adaptee, which is incompatible with the client.
* ITarget: This is going to be an interface that needs to be implemented by the Adapter. The client can only see this interface, i.e., the class which implements this interface.
* Adapter: This class makes two incompatible interfaces or systems work together. The Adapter class implements the ITrager interface and provides the implementation for the interface method. This class is also composed of the Adaptee, i.e., it has a reference to the Adaptee object as we are using the Object Adapter Design Pattern. In our example, the EmployeeAdapter class implements the ITarget Interface and provides implementations to the ProcessCompanySalary method of the ITarget Interface. This class also has a reference to the ThirdPartyBillingSystem object.
* Adaptee: This class contains the client’s required functionality but is incompatible with the existing client code. So, it requires some adaptation or transformation before the client can use it. It means the client will call the Adapter, and the Adapter will do the required conversions and then make a call to the Adaptee.





###### 2\. Facade design pattern



Facade (face of building) Design Pattern states that you need to provide a unified interface to a set of interfaces in a subsystem. The Facade Design Pattern defines a higher-level interface that makes the subsystem easier to use.

The Facade (usually a wrapper) class sits on the top of a group of subsystems and allows them to communicate in a unified manner.

// hide system complexity from client

// proxy takes care of only one object not sub systems



* The Facade Class knows which subsystem classes are responsible for a given request, and then it delegates the client requests to appropriate subsystem objects.
* The Subsystem Classes implement their respective functionalities assigned to them, and these Subsystem Classes do not know the Facade class.
* The Client Class uses the Façade Class to access the subsystems.





###### 3\. Decorator design pattern (is a and has a both)



The Decorator Design Pattern in C# allows us to dynamically add new functionalities to an existing object without altering or modifying its structure, and this design pattern acts as a wrapper to the existing class. That means the Decorator Design Pattern dynamically changes the functionality of an object at runtime without impacting the existing functionality of the object. In short, this design pattern adds additional functionalities to the object by wrapping it. A decorator is an object that adds features to another object.





###### 4\. Bridge design pattern



Decouples an abstraction from its implementation so that the two can vary independently

This pattern involves an interface that acts as a bridge between the abstraction class and implementer classes. It is useful in scenarios where an abstraction can have several implementations, and you want to separate the implementation details from the abstraction.

allows both Abstraction and Implementation to be developed independently, and the client code can only access the Abstraction part without being concerned about the Implementation part.

// interface logging process, interface of different logging process (bridge), and then the implementation

// here parent interface and implementation are not tightly coupled



###### 5\. Composite design pattern



Compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions of objects uniformly.

structural pattern that allows us to compose objects into tree structures to represent part-whole hierarchies. This pattern lets clients treat individual objects and compositions of objects uniformly. That means the client can access the individual objects or the composition of objects in a uniform manner. It’s useful for representing hierarchical structures such as file systems, UI components, or organizational structures.



###### 6\. Flyweight design pattern



The Flyweight Design Pattern reduces the number of objects created, decreases memory footprint, and increases performance. It’s especially useful when many objects share some common properties.

That means the Flyweight Design Pattern is used when there is a need to create many objects of almost similar nature. A large number of objects means it consumes a large amount of memory, and the Flyweight Design Pattern provides a solution for reducing the load on memory by sharing objects.



there are two states, i.e., Intrinsic and Extrinsic.

Intrinsic states are things that are Constants and Stored in Memory. On the other hand, Extrinsic states are things that are not constant and need to be calculated on the Fly; hence, they can not be stored in memory.

examples are important

// remove all extrinsic stated from class, class object becomes fly weight, cache and use



###### 7\. Proxy design pattern



Proxy Design Pattern provides a surrogate (act on behalf of another) or placeholder for another object to control access to it.

allows us to create a class that represents the functionality of other classes. The proxy could interface with anything, such as a network connection, a large object in memory, a file, or other resources that are expensive or impossible to duplicate.



* Virtual Proxy: A virtual proxy is a placeholder for “expensive to create” objects. The real object is only created when a client first requests or accesses the object.
* Remote Proxy: A remote proxy provides local representation for an object that resides in a different address space.
* Protection Proxy: A protection proxy controls access to a sensitive master object. The surrogate object checks that the caller has the access permissions required before forwarding the request.



Remote Proxy:
When: You are working with objects in a different address space or want to represent remote objects as if they were local.
Purpose: Handles the communication details with the remote object, abstracting the intricacies of remote communication from the client.



Virtual Proxy:
When: There’s a need to delay the creation and initialization of resource-intensive objects until they’re needed.
Purpose: Acts as a placeholder for the expensive object and initializes it on demand, optimizing performance (lazy initialization).



Protection Proxy: eg: access issue
When: You want to control access to the object based on access rights or add an additional layer of security.
Purpose: Check if the caller has the necessary permissions before granting access to the object.



Cache Proxy:
When: Repeated requests are made for the same object, and you want to improve performance by caching the result.
Purpose: Maintains temporary storage of results from expensive or frequently-used operations, preventing repeated calculations or fetches.



Logging Proxy:
When: You want to record the operations performed on an object for auditing or debugging purposes.
Purpose: Adds logging behavior every time an operation is requested on the object.



Monitoring or Synchronization Proxy:
When: You’re working in a multi-threaded environment and want to ensure that the object is accessed only one thread at a time.
Purpose: Ensures synchronized access to the actual object, preventing race conditions.



Smart Reference Proxy:
When: You need to perform additional actions or housekeeping tasks when accessing an object. This might include counting references to an object, loading or evicting objects from memory, etc.
Purpose: Performs extra actions before or after invoking the operations on the real object.

// adding additional features



Firewall Proxy:
When: You want to protect networked resources from malicious attacks or control data access going in or out.
Purpose: Acts as an intermediary layer to filter incoming or outgoing requests based on pre-defined rules.





##### 3\) Behavioral design pattern



Behavioral Design Patterns deal with the communication or interaction between Classes and Objects.

This ensures that the communication is carried out effectively while keeping the coupling loose. The primary goal of these patterns is to enhance the communication between objects, making it more flexible and efficient.



So, the behavioral design pattern explains how objects interact with each other. It describes how different objects and classes send messages to each other to make things happen and how the steps of a task are divided among different objects.



1. ###### Iterator design pattern



allows sequential access to the elements of an aggregate object (i.e., collection) without exposing its underlying representation. That means using the Iterator Design Pattern, we can access the elements of a collection sequentially without knowing its internal representations. This pattern provides a uniform interface for traversing different data structures.



It involves two primary types:



* Iterator: Defines an interface for accessing and traversing elements.
* Aggregate: An interface for creating an Iterator object.



###### 2\. Observer design pattern



Defines a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically.



This Design Pattern is widely used for implementing distributed event-handling systems where an object needs to notify other objects about its state changes without knowing who these objects are. In the Observer Design Pattern, an object (called a Subject) maintains a list of its dependents (called Observers). It notifies them automatically whenever any state changes by calling one of their methods. The Other names of this pattern are Producer/Consumer and Publish/Subscribe.



###### 3\. Chain of responsibility



Avoid coupling the sender of a request to its receiver by giving more than one receiver object a chance to handle the request. Chain the receiving objects and pass the request along until an object handles it.

eg: ATM, Vending machine, design logger



###### 4\. State design pattern



allows an object to alter its behavior when its internal state changes. In simple words, we can say that the State Design Pattern allows an object to change its behavior depending on its current internal state. The State Design Pattern encapsulates varying behavior for the same object based on its internal state.



Use Cases:

* When an object’s behavior depends on its state, it must be able to change its behavior at runtime based on internal state changes.
* In scenarios where complex conditional logic based on object state is required. The State Pattern can provide a cleaner solution.



They are as follows:



* State: This is going to be an interface that is used by the Context object. This interface defines the behaviors that the child classes will implement in their own way. In our example, it is going to be the IATMState interface.
* ConcreteStateA/B/C: These will be concrete classes that implement the State interface and provide the functionality the Context object will use. Each concrete state class provides behavior that applies to a single state of the Context object. In our example, it is the DebitCardNotInsertedState and DebitCardInsertedState classes.
* Context: This class will hold a concrete state object that provides the behavior according to its current state. This class is going to be used by the client. In our example, it is the ATMMachine class.



Advantages of State Design Pattern:

* Encapsulation of State-Based Behavior: State-specific logic is encapsulated in state classes.
* Easy to Add New States: Introducing new states doesn’t require changing the context or other states.
* Eliminates Conditional Statements: It helps to eliminate conditional statements for behavior changes based on the state.
* Maintainability and Flexibility: The state pattern makes it easier to maintain and extend state-based behavior.
* Dynamic Behavior Change: The context can change its behavior at runtime depending on its state.



eg: wending machine, tv



###### 5\. Template design pattern



defines a sequence of steps of an algorithm and allows the subclasses to override the steps but is not allowed to change the sequence.



Use Cases of Template Method Design Pattern:

* When you have a multi-step algorithm, and you want to ensure that the algorithm’s structure remains unchanged while allowing the steps to be overridden by subclasses.
* In scenarios where code duplication can be reduced by generalizing the framework in an abstract class.



###### 6\. Command design pattern



used to encapsulate a request as an object (i.e., a command) and pass it to an invoker, wherein the invoker does not know how to serve the request but uses the encapsulated command to perform an action.



The Command Design Pattern is a Behavioral Design pattern that turns a request into a stand-alone object that contains all information about the request. This transformation allows you to parameterize methods with different requests, delay or queue a request’s execution, and support undoable operations. It’s useful in scenarios where you need to issue requests without knowing anything about the operation being requested or the receiver of the request.



As per the Command Design Pattern, the Command Object will be passed to the Invoker Object. The Invoker Object does not know how to handle the request. What the Invoker will do is it will call the Execute method of the Command Object. The Execute method of the command object will be called the Receiver Object Method. The Receiver Object Method will perform the necessary action to handle the request.



undo redo very important in command



* Command: This is an interface for executing an operation.
* ConcreteCommand: This class extends the Command interface, implementing the Execute method by invoking the corresponding operation(s) on the Receiver object.
* Receiver: This class knows how to perform the operations associated with carrying out a request. Any class may serve as a Receiver.
* Invoker: This class asks the command to carry out the request.
* Client: The client creates a ConcreteCommand object and sets its receiver.



Use Cases of Command Design Pattern:

* When you need parameterized objects with an operation to be executed.
* When you need to queue operations, schedule their execution or execute them remotely.
* When you need to implement reversible operations (undo/redo).



###### 7\. Visitor design pattern



we use a Visitor object that changes an element object’s executing algorithm. In this way, when the visitor varies, the execution algorithm of the element object can also vary. As per the Visitor Design Pattern, the element object has to accept the visitor object so that the visitor object handles the operation on the element object.



The Visitor Design Pattern should be used when you have distinct and unrelated operations to perform across a structure of objects (element objects). That means the Visitor Design is used to create and perform new operations on a set of objects without changing the object structure or classes.



* Element: This represents an element of an object structure. It provides an Accept method that takes a visitor as an argument.
* ConcreteElement: This is a concrete class that implements the Element interface.
* Visitor: This interface declares a visit operation for each class of ConcreteElement.
* ConcreteVisitor: This is a concrete class that implements the Visitor interface.



###### 8\. Strategy design pattern



define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from clients that use it.



* Strategy (Interface or Abstract Class): This defines an interface common to all supported algorithms. Context uses this interface to call the algorithm defined by a ConcreteStrategy.
* ConcreteStrategy: This implements the algorithm using the Strategy interface.
* Context: This maintains a reference to a Strategy object and may define an interface that lets Strategy access its data.



Use Cases of Strategy Design Pattern:

* When you have many classes that differ only in their behavior, strategies provide a way to configure a class with one of many behaviors.
* When you need to switch algorithms used within an object at runtime dynamically.
* When you have a lot of conditional statements in different places to execute various behaviors of the same algorithm.



###### 9\. Interpreter design pattern



The Interpreter Design Pattern is a Behavioral Design Pattern that defines a grammatical representation of a language and provides an interpreter to deal with this grammar. The main idea is to define a domain language (a small language specific to your application) and interpret expressions in that language. It is useful when interpreting sentences in a language according to a defined grammar.



That means the Interpreter Design Pattern Provides a way to evaluate language grammar or expression. This pattern is used in SQL Parsing, Symbol Processing Engines, etc.



Let’s break down the design pattern:



* AbstractExpression: Declares an abstract Interpret operation common to all nodes in the abstract syntax tree.
* TerminalExpression: Implements the Interpret operation associated with terminal symbols in the grammar. It’s an instance of AbstractExpression.
* NonTerminalExpression: One level of grammar expression. For grammar rules that require multiple instances of AbstractExpression, you’d use the NonTerminalExpression.
* Context: Contains global information and is typically defined outside the grammar.
* Client: Builds (or is provided) the abstract syntax tree representing a particular sentence in the grammar. The tree is then evaluated by invoking the Interpret operation.



###### 10\. Mediator design pattern



define an object that encapsulates how a set of objects interact with each other. Mediator promotes loose coupling by keeping objects from explicitly referring to each other and letting you vary their interaction independently.

The Mediator Design Pattern reduces the communication complexity between multiple objects. This design pattern provides a mediator object, which will be responsible for handling all the communication complexities between different objects.



The Mediator Design Pattern restricts direct communications between the objects and forces them to collaborate only via a mediator object. This pattern is used to centralize complex communications and control between related objects in a system. The Mediator object acts as the communication center for all objects. That means when an object needs to communicate with another object, it does not call the other object directly. Instead, it calls the mediator object, and it is the responsibility of the mediator object to route the message to the destination object.



Use Cases of Mediator Design Pattern:

* When a set of objects communicate in well-defined but complex ways, and you want to centralize this communication to make it more manageable and understandable.
* In a system where multiple classes interact, but the interactions are unstructured or complex.

###### 

eg. airline mngmnt system, chat app

observer (event raised notify to observer) and proxy (lazy loading, logging) are similar but intent is different



###### 11\. Memento design pattern



The Memento Design Pattern is a Behavioral Design Pattern that can restore an object to its previous state. This pattern is useful for scenarios where you need to perform an undo or rollback operation in your application. The Memento pattern captures an object’s internal state so that the object can be restored to this state later. It is especially useful when implementing undo functionality in an application.



So, the Roles and Responsibilities of each component are as follows:



* Originator: The Originator is a class that creates a memento object containing a snapshot of the Originator’s current state. It also restores the Originator to one of its previous states. The Originator class has two methods. One is CreateMemento, and the other one is SetMemento. The CreateMemento Method will Create a snapshot of the current state of the Originator and return that Memento, which we can store in the Caretaker for later use, i.e., for restoring purposes. The SetMemento method accepts the memento object, and this method changes the Internal State of the Originator to one of its Previous States.
* Caretaker: The Caretaker class will hold the Memento objects for later use. This class acts as a store only. It never Checks or Modifies the contents of the Memento object. This class will have two methods, i.e., AddMemento and GetMemento. The AddMomento Method will add the memento, i.e., the internal state of the Originator, into the Caretaker. The GetMemento Method returns one of the Previous Originator Internal States, saved in the Caretaker.
* Memento: The Memento class holds information about the Originator’s saved state. That means it sets the internal state and gets the internal state of the Originator object. This class has one method called GetState, which will return the Internal State of the Originator. This class also has one parameterized constructor, which you can set the internal state of the originator.



Use Cases of Memento Design Pattern:

* When you need to provide an undo mechanism in applications like text editors, graphic editors, or more complex transactional systems.
* In scenarios where you want to capture an object’s state without exposing its implementation details.



Advantages of Memento Design Pattern:

* Undo Mechanism: Provides an undo mechanism by saving the object’s previous state.
* Preserving Encapsulation: It does not violate the originator’s encapsulation, as only the originator can store and retrieve information from the memento.
* Ease of Restoration: The originator can restore its state to a previous point in time.

