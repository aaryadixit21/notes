#### **object lifetime**



Transient

A new instance is created every time you request it.



Light-weight

Stateless

Safe

No shared data







Scoped

One instance per “scope.”

In web apps:



One scope = one HTTP request

Same instance reused inside that request







Singleton

Only one object for the entire application.



Shared globally

Lives until application stops

Must be thread-safe





##### **When to use what?**



1\. Transient — Most common

Use when:



Class is stateless //no field or property of other class, utility func

Cheap to create

Independent



Examples:



Handlers

Commands

Validators

Repositories (if stateless)



2\. Scoped — Perfect for web requests

Use when:



You need one instance shared across a request

You need per-request caching

You need per-request database unit of work



Examples:



DbContext

UnitOfWork

Request-level services



3\. Singleton — Use carefully

Use when:



Object must be shared globally

Expensive to create //will stay in the memory till application runs

No request-related data



Examples:



Configuration readers

Caches

Loggers



“Object lifetime is not about choosing Transient vs Singleton — it's about ensuring your entire dependency graph has a consistent, non-conflicting lifetime chain.”

Ensuring that no service depends on another service with a shorter lifetime.





## Service Resolution



the act of creating an object together with all of its dependencies



Example:

If class A needs B, and B needs C, then creating A means:



Create C

Create B with C

Create A with B



👉 Service resolution means assembling an object graph.



Why Service Resolution matters

Because in real applications:



Services depend on other services

Those depend on more services

Which depend on configuration, database connections, loggers…



If you manually resolve everything:



Constructor chains become huge

Code becomes repetitive

Changes ripple everywhere

Hard to test

Hard to manage lifetimes

Hard to remove or swap implementations



This is why we move to IoC Containers



## What does a DI container do during service resolution?

###### It:

1. Reads the registration map
2. Creates the dependency tree
3. Injects constructor parameters automatically
4. Respects lifetime rules (transient / scoped / singleton)
5. Detects misconfigurations (e.g., circular dependencies)



This means you never new up dependencies yourself.

The DI container handles the whole chain.



## code placement



Service resolution happens ONLY in:

###### 

###### Composition Root

Typically the application startup:

Program.cs

Startup.cs

DependencyInjection.cs

Never inside Domain, Application, or Infrastructure services.





#### 1\. Internal Mechanics of Service Resolution



When resolving:

container.Get<IOrderService>();





The container:



1. Looks up the mapping for IOrderService
2. Inspects the constructor of the implementation
3. Recursively resolves each dependency
4. Builds an object graph
5. Applies lifetime rules
6. Caches instances (if scoped/singleton)
7. Returns the fully constructed object



This is called recursive constructor injection resolution.





#### 2\. The Resolution Graph





OrderService

 ├── IOrderRepository

 │     └── AppDbContext

 ├── ILogger

 └── IMessagePublisher



The container builds it bottom-up:



1. Create AppDbContext
2. Create OrderRepository(AppDbContext)
3. Create Logger
4. Create Publisher
5. Create OrderService(Repo, Logger, Publisher)



This is extremely difficult to do manually at scale.



“Service resolution must happen exactly once — at the root — and the entire dependency graph must be consistent and lifetime-safe.”







*A DI container is a specialized form of an IoC container.*

*All DI containers are IoC containers, but not all IoC containers are DI containers.*



*IoC Container = big hotel manager*

*It manages guests, rooms, events, electricity, cleaners — MANY systems.*

*DI Container = room service*

*It just delivers items when you ask.*

*It only handles one specific job: give objects with their dependencies.*





## IoC container



An IoC (Inversion of Control) container is a smart object builder that:



Creates objects

Injects their dependencies

Manages lifetimes (transient/scoped/singleton)

Resolves the entire object graph automatically



Think of it as:

👉 A factory that knows how to build everything in your system.

Real‑world analogy

Imagine a restaurant kitchen:



You order a dish (a service)

The chef (IoC container) knows the recipe

The chef gets ingredients (dependencies)

Prepares them in the right order

Delivers the final dish



You don't care how it happened; you just get the final output.

without container:



var logger = new ConsoleLogger();

var repo = new UserRepository(logger);

var service = new UserService(repo);



with IoC:
var service = container.Get<IUserService>();



##### When to use an IoC container

Always use one when:



Your system has >10 services

You follow Clean Architecture

You follow DDD

You want testability

You want separation of registration vs usage

You want lifetime control

You want automatic resolution





###### IoC container ONLY lives in:

API layer

Program.cs

Startup.cs

or a dedicated Composition Root





###### How containers resolve dependencies internally

IoC containers perform recursive constructor injection:



1. Find implementation type
2. Choose constructor
3. Reflect on parameters
4. Recursively resolve each dependency
5. Cache if scoped/singleton
6. Return fully assembled instance



This involves:



1. Reflection
2. Expression trees (for optimized containers)
3. Caching optimizations
4. Internal resolution graphs
5. Circular dependency detection





###### Performance Considerations

Some containers are fast (Lamar, DryIoc).

Some are slower (Unity).

Some balance usability and speed (Autofac, Ninject).

Ninject is one of the slowest IoC containers because it relies heavily on:



Reflection

Interception

Dynamic proxies



…but it's extremely flexible and easy to use.





###### Advanced Lifetime Safety Rule



Every chain of dependencies must follow:

Singleton ≥ Scoped ≥ Transient



You must never go upward.

Correct chains:



Singleton → Singleton

Scoped → Transient

Transient → Scoped



“An IoC container is a composition engine that builds object graphs from registrations using recursive constructor injection while enforcing lifetime boundaries. It centralizes dependency management, enabling loosely coupled, testable architectures like DDD and Clean Architecture.”





#### Ninject



Think of Ninject as a smart factory (IoC container) that:



* Knows recipes for interfaces → implementations (Bind<IFoo>().To<Foo>())
* Builds object graphs by reading your constructor signatures
* Controls lifetimes/scopes (Transient, Singleton, Request) (default transient)
* Supports conditional/contextual bindings, interception (AOP), factories, open generics, etc.





Ninject is a .NET IoC/DI container. It creates your services and injects dependencies so that your classes never new their own dependencies. You declare bindings (mappings), and Ninject resolves the full object graph for you.





Why use it?



You avoid tight coupling and manual wiring

You get testable code (swap implementations)

You get lifetime control (singleton, request, etc.)

You reduce composition noise in your business code





example



static void Main()

    {

        var kernel = new StandardKernel();

        kernel.Bind<IMessageSender>().To<EmailSender>(); // registration (binding)



        var notifier = kernel.Get<Notifier>(); // auto-resolve Notifier + IMessageSender

        notifier.Notify("aarya@example.com", "Welcome to Ninject!");

    }





Visual/Mental Model



* Bind: “If someone asks for IMessageSender, give EmailSender.”
* Get: “Build Notifier by building all it needs.”



Basic Best Practices



* Put bindings in the Composition Root (e.g., Program/Startup)
* Don’t inject the container into services
* Prefer constructor injection over property injection





###### Core API you’ll use



* var kernel = new StandardKernel(params INinjectModule\[] modules);
* kernel.Bind<TService>().To<TImpl>();
* Lifetimes: .InSingletonScope(), .InTransientScope() (default), .InThreadScope()
* Web: .InRequestScope() (via Ninject.Web.Common)
* Conditional/contextual: .When(...), .WhenInjectedInto<...>(), .Named("..."), .WhenParentNamed("...")
* Providers / factories: .ToProvider<TProvider>(), Ninject.Extensions.Factory





example



infra.cs

using Ninject.Modules;



public class InfrastructureModule : NinjectModule

{

    public override void Load()

    {

        // Repos - per request (web) or scoped equivalent

        Bind<IOrderRepository>().To<EfOrderRepository>(); // default transient; override per host

 

        // DbContext typically per request scope in web; transient in console batch

        Bind<AppDbContext>().ToSelf();

 

        // Cross-cutting

        Bind<ILogger>().To<ConsoleLogger>().InSingletonScope();

        Bind<IMessagePublisher>().To<KafkaPublisher>().InSingletonScope();

    }

}



program.cs

using Ninject;



class Program

{

    static void Main()

    {

        var kernel = new StandardKernel(new InfrastructureModule(), new ApplicationModule());

        var handler = kernel.Get<PlaceOrderHandler>();

        handler.Handle(new PlaceOrderCommand { OrderId = "ORD-1001" });

    }

}





###### Lifetimes (Ninject)



* Transient (default): New instance every time you resolve

 	.InTransientScope() or omitted

* Singleton: One instance per kernel for app’s life

 	.InSingletonScope()

* Request (Web only with Ninject.Web.Common): .InRequestScope()





###### contextual and named binding



// Named variants

Bind<IMessageSender>().To<EmailSender>().Named("email");

Bind<IMessageSender>().To<SmsSender>().Named("sms");



// Injection site chooses name:

public class NotificationService

{

    public NotificationService(\[Named("email")] IMessageSender emailSender,

                               \[Named("sms")] IMessageSender smsSender) { ... }

}



// Contextual:

Bind<IMessageSender>().To<AuditEmailSender>()

    .WhenInjectedInto<AuditNotifier>();





###### ToMethod / Provider (custom construction)



Bind<IDateTime>().ToMethod(ctx => new SystemClock(TimeZoneInfo.Utc)).InSingletonScope();



// Provider pattern

public class KafkaClientProvider : Provider<KafkaClient>

{

    protected override KafkaClient CreateInstance(IContext context)

    {

        var cfg = context.Kernel.Get<IKafkaConfig>();

        return new KafkaClient(cfg.ConnectionString);

    }

}

Bind<KafkaClient>().ToProvider<KafkaClientProvider>().InSingletonScope();





Scenario                                                          Prefer

Concrete class with dependencies all registered in DI             ToSelf()

Need to pass a literal					          ToSelf() + WithConstructorArgument(...)

Need custom construction                                          ToMethod(...)

Need reusable complex creation with clean separation              ToProvider()

