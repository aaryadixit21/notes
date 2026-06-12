\--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

***basics***





1. Entity and DTO

An Entity is a class that represents a table in the database.

A DTO is a simplified object used to transfer data between layers (e.g., API → Client). It is not tied to the database.

Entity = Full database record (like raw data in Excel)

DTO = Filtered export (only needed columns shared with someone)



2\. Reflection

Reflection is a feature that allows a program to inspect and manipulate its own structure at runtime. Libraries like AutoMapper use reflection to map one object to another. this is slow. at runtime we dont use reflection, use cached compiled delegates



3\. expression tree
An expression tree is a data structure that represents code as a tree.



Reflection-based mapping would do this at runtime:

propertyInfo.GetValue(source);

propertyInfo.SetValue(destination, value);



That is slower because reflection has overhead.

Your library instead builds expression trees like:



src => new Destination

{

&#x20;   Name = src.Name,

&#x20;   Age = src.Age

}



Then compiles them into delegates.

expression trees are used in TypeMapSealer to build factories, setters, getters, direct-copy delegates, full map functions, and nested map functions.



I used expression trees because they let me generate mapping code dynamically at configuration time and compile it into delegates. This avoids reflection on every map call and gives performance close to handwritten mapping while still supporting runtime configuration.





4\. Paths



The Three Execution Paths

1\. CachedFastMapFunc

2\. MapNoContext

3\. MapInternal



1\. CachedFastMapFunc

This is the fastest path.

Used when:



Mapping is simple.

Properties are mostly simple types.

No circular reference.

No MaxDepth.

No complex type converter.



For simple flat objects, I generate a single compiled expression-tree delegate that creates the destination object and assigns all properties directly. So runtime mapping becomes almost like handwritten mapping code.





2\. MapNoContext

This is used for nested objects or collections when your configuration-time analysis proves there is no circular reference.



Order -> OrderDto

Order.Customer -> CustomerDto

Order.Items -> List<OrderItemDto>



Here, you need nested mapping, but you may not need a full ResolutionContext.

So your library avoids allocating extra context objects.



MapNoContext handles nested and collection mapping without allocating a full mapping context, as long as the object graph is proven cycle-free. This improves performance and reduces memory allocation.





3\. MapInternal

This is the full safe path.

Used when:



Circular reference is possible.

MaxDepth is used.

Existing destination object is passed.

Type converter needs context.

Mapping is too complex for fast path



This path uses ResolutionContext.



MapInternal is the most general path. It supports circular reference handling, max depth, type converters, existing destination mapping, and all complex scenarios. The trade-off is that it allocates more memory because it needs context and identity tracking.







5\. Sealing

A TypeMap is the internal metadata model for one source-destination pair. It captures what should be mapped, how each member should be mapped, and after sealing, it also stores compiled execution delegates for fast runtime mapping.

Before sealing, the map is mutable. Users are still configuring it.



After sealing, the map becomes finalized.

The response says TypeMap.Seal() delegates to TypeMapSealer.Seal, which builds factories, compiles source expressions, builds property mappings, caches delegates, and tries to build the fast mapping function.



Simple explanation:

Sealing means converting user configuration into an optimized, read-only execution plan.

I used a two-phase model. First, configuration APIs collect user intent. Then sealing freezes the TypeMap and compiles expression-tree delegates. After sealing, the mapper can safely share the configuration across threads because the mapping metadata is effectively immutable.





6\. Why ConcurrentDictionary and \_lastMapFunc?



\_mapFuncCache stores mapping delegates per mapper instance, while \_lastMapFunc is a single hot-path cache for the most recently used type pair.

If an application maps the same type repeatedly, like thousands of UserEntity objects to UserDto, checking \_lastMapFunc is faster than doing a dictionary lookup every time.



I used ConcurrentDictionary because the mapper is expected to be shared across threads. It gives safe concurrent reads and writes. I added \_lastMapFunc as a micro-optimization because many workloads repeatedly map the same source-destination pair in a loop. A volatile single-entry cache avoids even the dictionary lookup on the hot path



7\. Circular reference handling



If the mapper sees the same source object again while mapping the same graph, it reuses the already-created destination object instead of mapping forever.



Circular references require per-call state. I store already mapped source instances in ResolutionContext using identity-based keys. Before mapping a source object, I check whether it already exists in the context cache. If yes, I return the existing mapped destination. This prevents infinite recursion and preserves object identity.



8\. reverse map



Custom forward MapFrom or Ignore rules are not automatically inverted. The reverse map is convention-based unless explicitly configured.

My ReverseMap creates a new reverse TypeMap and registers it in the same profile. It supports convention-based reverse mapping and allows additional reverse-side configuration. I intentionally did not try to automatically invert all custom expressions because not every custom mapping is reversible.



9\. Testing



Unit tests prove your own implementation works. Parity tests prove your behavior matches AutoMapper for the features you support.

I used xUnit and FluentAssertions. For parity tests, I configured both AutoMapper and my CustomAutoMapper with equivalent configurations, mapped the same input, and compared the outputs structurally using BeEquivalentTo. This made AutoMapper itself an executable specification for the features I implemented.

\--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------



***complete workflow***





User configuration

&#x20;   ↓

Profile

&#x20;   ↓

CreateMap

&#x20;   ↓

TypeMap

&#x20;   ↓

Seal TypeMap

&#x20;   ↓

Compile expression delegates

&#x20;   ↓

Resolve nested maps and cycles

&#x20;   ↓

Create IMapper

&#x20;   ↓

Runtime Map call

&#x20;   ↓

\_lastMapFunc cache

&#x20;   ↓

ConcurrentDictionary cache

&#x20;   ↓

FastPath / NoContext / MapInternal

&#x20;   ↓

Destination object





Configuration Phase → Optimization Phase → Runtime Phase

Left side → User defines mapping

Middle → Your library prepares execution plan

Right side → Actual mapping happens very fast





I built CustomAutoMapper, a .NET 8 object-mapping library inspired by AutoMapper 6.2.2. It supports core features like convention-based mapping, ForMember, MapFrom, Ignore, UseValue, Condition, AfterMap, ConvertUsing, nested mapping, collections, ReverseMap, MaxDepth, and circular reference handling. Internally, each source-destination pair is represented as a TypeMap. During configuration, I seal these TypeMaps and compile expression trees into delegates, so runtime mapping avoids reflection. At runtime, the mapper chooses between three execution paths: a single compiled fast delegate for flat maps, a no-context path for cycle-free nested graphs, and a full context-aware path for circular references and complex cases. I validated correctness with 600+ xUnit tests, including parity tests against AutoMapper 6.2.2, and benchmarked it with BenchmarkDotNet and MemoryDiagnoser





The mapping flow has three main phases.

First is configuration. The user defines mappings using profiles and CreateMap, which internally creates TypeMap objects.

Second is the optimization phase. During configuration, each TypeMap is sealed, where I compile expression trees into delegates, resolve nested mappings, and detect circular references. This converts the mapping into an optimized execution plan.

Third is runtime mapping. When Map is called, the mapper first checks a hot-cache called \_lastMapFunc, then a ConcurrentDictionary cache. Based on the mapping complexity, it chooses one of three execution paths:



a fast compiled delegate for simple mappings,

a no-context path for nested but cycle-free mappings,

or a full context-aware path for circular references and complex graphs.



This design allows the mapper to be both efficient and flexible by doing all expensive work upfront and keeping runtime mapping extremely fast.



\--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------



***Interview questions***





* ###### What problem does object mapping solve?

Object mapping solves the problem of converting one object shape into another object shape, usually across application boundaries. This becomes error-prone as the number of models and properties grows. A mapping library centralizes these rules, reduces boilerplate, and keeps domain models decoupled from API or persistence models.



* ###### What is a TypeMap?

A TypeMap is the internal representation of one mapping relationship.

It stores the source type, destination type, member configurations, custom mapping expressions, ignored members, conditions, converters, AfterMap actions, and after sealing, cached compiled delegates used for runtime execution



CreateMap<UserEntity, UserDto>()    typemap created:  typeof(UserEntity), typeof(UserDto)

During configuration, this TypeMap is mutable. During sealing, it becomes an optimized execution plan with compiled factories and property-mapping delegates.



* ###### What is sealing?

After sealing, the configuration should not change. At this point, the library compiles expression trees, builds destination object factories, prepares property mappings, caches AfterMap actions, and tries to build a fast mapping delegate if the map is eligible.



This gives two benefits:

Runtime mapping becomes fast because most expensive work is done upfront.

The configuration becomes effectively immutable, which makes it safer to share across threads.



* ###### Why are compiled delegates faster than reflection?

A compiled delegate is different. Once the expression tree is compiled, it becomes executable code that can be invoked directly, almost like a normal method call. So instead of discovering properties and invoking them reflectively on every map call, my library pays that cost once during configuration and then reuses the compiled delegate at runtime.



* ###### How do you handle nested mapping?

Nested mapping happens when a destination property itself needs another map.

eg:

Order -> OrderDto

Order.Customer -> CustomerDto     two maps will be created



During the post-seal phase, my configuration scans property mappings and resolves nested TypeMaps ahead of time. If a destination property is complex and there is a registered map for the source property type to destination property type, that nested map is stored on the PropertyMapping.



At runtime, depending on complexity, nested mapping can be handled by:



Inlining a child fast function if possible.

Using MapNoContext if the graph is cycle-free.

Using MapInternal if circular references or max depth handling are needed.



* ###### How do you handle collections?

For collection mapping, the mapper checks whether the source and destination properties are collection-like types. It excludes string because string implements IEnumerable<char> but should not be treated as a collection. The collection mapper extracts the source and destination element types. If the elements are simple or directly assignable, it copies them. If the element types require mapping, it uses the registered element TypeMap.

Collection mapping is handled through cached factories and element mapping, with correctness support for mapped elements





* ###### How do you handle circular references?

My library handles this using ResolutionContext. When mapping with the full context-aware path, before mapping a source object, I check whether that source object has already been mapped to the requested destination type. If yes, I return the cached destination object. If not, I create the destination object, cache it, and then continue mapping its members.

The cache key is based on object identity, not logical equality, because two different objects may be equal by value but should still be treated as different graph nodes. The implementation uses RuntimeHelpers.GetHashCode and ReferenceEquals for this.





* ###### Resolution Context

Simple maps do not need this context, so my mapper avoids allocating it on the fast path. But when the map involves circular references, MaxDepth, existing destination objects, or full converter behavior, ResolutionContext becomes necessary.  ResolutionContext is the state carrier for complex object graph mapping, especially circular reference detection and depth tracking.



* ###### Why did you use ConcurrentDictionary?

I used ConcurrentDictionary because the mapper is designed to be shared across multiple threads after configuration.

The mapping configuration is effectively immutable after sealing, but the mapper still has runtime caches, especially the map function cache: (source runtime type, destination type) -> compiled mapping delegate

Multiple threads may map different object types at the same time and may populate this cache concurrently. A normal Dictionary would not be safe for concurrent writes. A lock around a normal dictionary would work but would add unnecessary contention. ConcurrentDictionary gives thread-safe access with efficient reads and controlled writes.

The project also uses ConcurrentDictionary for process-wide caches such as collection list factories, typed add delegates, and simple-type detection.



Since the mapper is read-heavy and write-light, ConcurrentDictionary fits the “configure once, map many times” model very well.



* ###### What is \_lastMapFunc and why is it volatile?



\_lastMapFunc is a single-entry hot-path cache.

Most applications map the same type pair repeatedly. For example, mapping a list of UserEntity objects to UserDto objects means the mapper repeatedly uses the same source-destination pair.

Instead of doing a ConcurrentDictionary.TryGetValue on every call, \_lastMapFunc stores the most recently used mapping delegate. If the next call has the same source and destination types, the mapper can invoke that delegate immediately.

It is marked volatile to make the reference read/write safe and visible across threads without using locks. The race condition is benign: if two threads overwrite \_lastMapFunc, the worst case is a cache miss and fallback to the dictionary. Correctness is not affected because the delegate is always derived from the type pair.





* ###### Benchmark



I used BenchmarkDotNet not just to claim speed, but to understand where my design wins and where it still needs improvement.

I used BenchmarkDotNet with MemoryDiagnoser.

BenchmarkDotNet measured runtime performance and memory allocations across different mapping scenarios, such as simple mapping, MapFrom, nested mapping, collections, converters, conditions, ignored members, AfterMap, ReverseMap, large object mapping, batch mapping, and configuration setup.

Each benchmark had AutoMapper as the baseline and CustomAutoMapper as the comparison. MemoryDiagnoser reported allocated bytes per operation and GC-related metrics such as Gen0 collections.



* ###### strongest part of project

The strongest part is the architecture. I separated configuration-time work from runtime work. At configuration time, I build and seal TypeMaps, compile expression trees, resolve nested maps, and analyze cycles. At runtime, I use cached delegates and choose between fast-path, no-context, and context-aware execution. That gave me a good balance between correctness, performance, and maintainability.



\--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------







1\. Project Overview



1.1 What this library does

CustomAutoMapper is a convention-based, opt-in object-to-object mapper for .NET 8. The user declares maps via a fluent API (MapperConfiguration + MapperProfile.CreateMap<TSource, TDestination>()), the library introspects the types using reflection, compiles a per-type execution plan into delegates via System.Linq.Expressions, and exposes an IMapper with three overloads of Map<…> for runtime conversion.



1.2 Real-world problem it solves

•	Removes the repetitive, error-prone hand-written copy code at boundary layers (Entity ↔ DTO, API request ↔ domain model, ViewModel ↔ command).

•	Centralizes mapping rules so changes to data shape happen in one place.

•	Decouples persistence/transport types from domain types.



1.3 Why object mapping libraries are useful

•	Reduce LOC and bug surface in CRUD-heavy layers.

•	Provide a uniform place to put cross-cutting concerns (conditions, AfterMap hooks, ignores, constants).

•	Enable test parity: "given these inputs, this DTO" can be asserted at the mapper level.



1.4 Similarities to AutoMapper

•	Same fluent surface: IMapperConfigurationExpression.CreateMap<S,D>().ForMember(...).Ignore() / .MapFrom(...) / .UseValue(...) / .Condition(...) / .ResolveUsing(...) / .AfterMap(...) / .ConvertUsing(...) / .As<T>() / .MaxDepth(n) / .ReverseMap().

•	Same profile model: MapperProfile (abstract, \_typeMaps collection) mirrors AutoMapper's Profile.

•	Same member-list validation modes: MemberList.Source | Destination | None in MemberList.cs.

•	Same last-wins semantics in MapperProfile.CreateMap (lines 17–31 of MapperProfile.cs) and global last-profile-wins in MapperConfiguration.BuildTypeMaps (line 251).

•	Same ResolutionContext contract: Items, Depth, Parent, CreateNestedContext (ResolutionContext.cs, lines 58–106).



1.5 Differences

•	Much smaller surface: no IValueResolver<TSource,TDestination,TMember> (only Func<object, object?>), no ConstructUsing, no Include/IncludeBase for inheritance, no ProjectTo for IQueryable, no open generics, no PreserveReferences toggle, no AllowNullCollections (instead, collections always become empty), no DisableConstructorMapping, no AddMaps(assembly) (only AddProfiles(assembly)).

•	Adds three explicit execution paths (CachedFastMapFunc, MapNoContext, MapInternal) chosen at config time + a \_lastMapFunc volatile last-hit cache (Mapper.cs, lines 17–32).

•	LambdaTypeConverter.Convert ignores the supplied destination and context (LambdaTypeConverter.cs, lines 15–20) — this is a deliberate simplification, not full AutoMapper parity.



1.6 Replicated AutoMapper 6.2.2 behavior (verified in code/comments)

•	Last-profile-wins on duplicates (MapperConfiguration.cs line 250 comment; MapperProfile.cs line 17 comment).

•	Inline CreateMap profiles are inserted before added profiles so explicit profiles "win" (MapperConfigurationExpression.cs lines 66–68).

•	CreateMapper() does not validate (MapperConfiguration.cs line 228).

•	ConvertUsing causes member-level validation to be skipped (ConfigurationValidator.cs lines 25–27).

•	MaxDepth(0) is ignored (does not block the root) — verified by the parity test MaxDepth0\_SelfReferencing\_NoRecursionAtAll (Mapper\_DepthAndCircularRef\_Tests.cs lines 80–92).

\---





2\. Architecture Deep Dive



2.1 Major modules / classes / interfaces

Layer			Type																	Responsibility

Public API (config)	IMapperConfigurationExpression, IMappingExpression<S,D>, IMemberConfigurationExpression<S,D>, IConfigurationProvider, ITypeConverter	User-facing fluent surface and contracts.

Config implementations	MapperConfiguration, MapperConfigurationExpression, MappingExpression<S,D>, MemberConfigurationExpression<S,D>				Capture user intent into TypeMap + MemberConfiguration graphs.

Profiles		MapperProfile (abstract), InlineProfile (internal, sealed)										Group of related CreateMap calls.

Domain (metadata)	TypeMap (sealed), MemberConfiguration (sealed), PropertyMapping (internal sealed), MemberList (enum)					"What" to map.

Sealing/compilation	TypeMapSealer (static), ParameterReplacer (private nested)										Turns metadata into compiled delegates.

Validation		ConfigurationValidator (static), SourceMemberAccessVisitor (ExpressionVisitor)								Implements AssertConfigurationIsValid.

Runtime			IMapper, Mapper (sealed), ResolutionContext (sealed), CollectionMapper (static), EnumConverter (static), LambdaTypeConverter<S,D> 	Actually performs the mapping.

Utilities		ReflectionHelper (static)											Centralized reflection (simple-type cache, collection element extraction, expression member name).



2.2 End-to-end mapping flow



1. Configure: User constructs new MapperConfiguration(cfg => cfg.AddProfile(new MyProfile())).
2. Capture: Each CreateMap<S,D>() builds a TypeMap and a MappingExpression<S,D>. Fluent calls mutate TypeMap and per-member MemberConfigurations. Implicit members are discovered immediately in TypeMap.DiscoverImplicitMembers() using a case-insensitive lookup.
3. Build maps: MapperConfiguration.BuildTypeMaps(profiles) flattens all profiles into \_typeMaps : Dictionary<(Type,Type), TypeMap>.
4. Seal: For each TypeMap, TypeMap.Seal() delegates to TypeMapSealer.Seal, which:
* builds CachedFactory (() => (object)new TDest()),
* eagerly compiles every SourceExpression to CompiledSourceExpression,
* builds an array of PropertyMapping with lazy delegates for setter/getter/direct-copy,
* tries to build CachedFastMapFunc if all destination properties are simple types (no MaxDepth, no TypeConverter).

5\.	Post-seal pass 1 (PostSealResolveNestedMappings): for each TypeMap, resolve NestedTypeMap, CollectionElementTypeMap, CollectionListFactory\*, CollectionTypedAdd, and record (allResolvable, hasCollection) per map.

6\.	Post-seal eligibility pass: walk every map again; if all children resolvable and no cycle (HasCircularReference DFS), set EligibleForNoContextPath = true.

7\.	Post-seal pass 3 (PostSealBuildNestedFastMapFuncs): fixed-point loop that calls TypeMapSealer.BuildNestedMapFunc for each eligible map (leaves succeed first; parents succeed in subsequent iterations).

8\.	CreateMapper: new Mapper(this) is returned. No validation runs.

9\.	Runtime Map<TDestination>(source):

•	Hit \_lastMapFunc volatile field — if same (source.GetType(), typeof(TDestination)), invoke directly.

•	Otherwise \_mapFuncCache.TryGetValue (per-mapper ConcurrentDictionary).

•	On miss: ResolveMapFunc returns the best delegate based on CachedFastMapFunc → TypeConverter → EligibleForNoContextPath → MapInternal fallback. Cached.



2.3 How TypeMaps are created, sealed, cached, executed

•	Created in MapperProfile.CreateMap (replaces existing for same (S,D)).

•	Sealed in MapperConfiguration constructor after BuildTypeMaps (single pass), then enriched in the two post-seal passes.

•	Cached in \_typeMaps (configuration scope) and indirectly via \_mapFuncCache (mapper instance scope).

•	Executed through one of three paths (see §5).



2.4 Config-time vs runtime

•	Config-time = inside MapperConfiguration ctor. Everything that allocates Lambda/Compile happens here. Costs \~4.7 ms / \~356 KB for 4 profiles in your benchmarks.

•	Runtime = every mapper.Map<…>(src) call. Should be 17–60 ns and \~24–216 B per object for the configured benchmark scenarios.



2.5 Validation

•	AssertConfigurationIsValid() → ConfigurationValidator.Validate(typeMaps).

•	Per type map: if it has a TypeConverter/TypeConverterType, skip (line 26–27).

•	Otherwise, walk MemberConfigurations (currently a no-op — see §13.7 weakness), then run member-list rules:

•	MemberList.Destination → every writable public destination property must have an entry in \_memberConfigurations.

•	MemberList.Source → every readable public source property must be either name-matched by a destination property or explicitly referenced inside a MapFrom expression. The latter uses SourceMemberAccessVisitor to walk the expression tree.

•	MemberList.None → no check.



2.6 Public API design

•	Fluent and chainable (IMappingExpression<S,D> returns this from every method).

•	Generic class constraints on CreateMap<TSource, TDestination> where TSource : class where TDestination : class (forces ref types — see §13 weakness for value-type support).

•	Separation between expression interfaces (for configuration) and TypeMap/MemberConfiguration (the captured metadata).

\---





3\. Core Design Decisions



Decision	Where	Rationale

Use System.Linq.Expressions + .Compile()	TypeMapSealer.BuildFactory/BuildSetter/BuildSourceGetter/BuildDirectCopy/BuildFullMapFunc/BuildNestedMapFunc	Reflection on every call costs hundreds of ns and allocates; a JIT-compiled delegate is \~5–20 ns per call and avoids MethodInfo.Invoke boxing.

Compiled delegates over Activator.CreateInstance	CachedFactory (TypeMapSealer.BuildFactory)	Activator.CreateInstance(Type) does reflection + caches per call site; a compiled Expression.New lambda is \~3× faster and fully inlinable.

sealed TypeMap / MemberConfiguration / Mapper	All three classes	(a) Prevents downstream override of perf-critical types, (b) lets the JIT inline virtual calls (sealed types can have non-virtual dispatch), (c) clarifies intent: "this is the final implementation".

ConcurrentDictionary for \_mapFuncCache	Mapper.cs line 17	First call writes; subsequent calls only read. Lock-free reads + striped locks on write match the "configure once / map many" lifecycle. A normal Dictionary would require a global lock or risk torn reads.

\_lastMapFunc volatile single-entry cache	Mapper.cs lines 23–32	Most apps map the same type pair in a hot loop. A volatile read + two reference equality checks is faster than a ConcurrentDictionary.TryGetValue, which still computes a hash and acquires a bucket.

Fast-path vs context-aware paths	ResolveMapFunc, MapInternal, MapNoContext	Most real maps don't need ResolutionContext. Allocating one per call (\~150 B + dict alloc) costs more than the actual mapping. The 3-path strategy lets you "pay only for what you use".

Circular reference handling requires a context	ResolutionContext.\_instanceCache with InstanceCacheKey(source identity, dest type)	Detection must remember the specific source instance already in-flight; this is per-call mutable state and cannot be captured in a config-time delegate.

Avoid context allocation when no cycle is possible	PostSealResolveNestedMappings.HasCircularReference DFS sets EligibleForNoContextPath	Static structural analysis at seal time proves no cycle exists, so runtime cache is unnecessary.

Trade-off vs pure reflection	All Cached\* fields	Reflection is simple but slow and allocation-heavy. Cost: more code, larger compile-time graph, higher memory at seal time.

Trade-off vs source generators	(Not used)	SGs would be even faster (zero startup compilation, AOT-friendly) but require build-time tooling, no dynamic types, harder to debug. You chose runtime expressions for AutoMapper-feel and dynamic support.

Trade-off vs AutoMapper	(vs 6.2.2)	You traded breadth of features for lower allocations on the hot path and simpler code. AutoMapper does more (open generics, projection, custom resolvers with context, inheritance) but each call allocates an IMapper-scoped context.

\---





4\. Feature-by-Feature



For each feature: what / where / how / edges / likely Qs / answers.



4.1 Basic object-to-object mapping

•	Where: TypeMap.DiscoverImplicitMembers, fast path in TypeMapSealer.BuildFullMapFunc (Expression.MemberInit branch, lines 209–229).

•	How: Implicit discovery uses case-insensitive source name match; explicit pass selects writable public destination props. If every dest prop has a same-named, same-typed source prop and no per-member configuration, the sealer emits dest = new TDest { A = src.A, B = src.B, … } as a single compiled lambda.

•	Edge cases: Source missing a prop → that mapping is dropped (no source getter cached). Different prop types → falls back to slower BuildFullMapFunc block path with Expression.Convert (may throw at compile if not convertible — silently swallowed by the catch at line 318).

•	Likely Q: "How are matching properties found?"

Answer: TypeMap.DiscoverImplicitMembers builds an OrdinalIgnoreCase dictionary of source props and intersects it with writable public destination props.



4.2 Property matching

•	Convention-based, case-insensitive on names, exact type match required for the ultra-fast MemberInit path. For mismatched types, BuildFullMapFunc emits Expression.Convert.

•	No flattening (SourceFooBar → Source.Foo.Bar) — that is not implemented. Only direct name match + explicit MapFrom.



4.3 Constructor mapping

•	Not implemented. BuildFactory always emits Expression.New(destType) (parameterless). If the destination has no parameterless ctor, Compile would still work, but Activator.CreateInstance fallback in CreateDestination would throw. No ConstructUsing API exists.



4.4 Nested object mapping

•	Where: MapperConfiguration.PostSealResolveNestedMappings (resolves NestedTypeMap), TypeMapSealer.BuildNestedMapFunc (inlines child fast func), Mapper.MapNoContext (no-context path), Mapper.MapInternal (lines 477–499 nested path with CreateNestedContext).

•	How: For each non-simple destination prop, look up (srcPropType, destPropType) in \_typeMaps. If found, store on PropertyMapping.NestedTypeMap and emit inlined Expression.Invoke(childFastFunc, src.Prop) in the parent's compiled lambda when possible.

•	Edge cases: Source nested object is null → emit if (src.Prop != null) dest.Prop = childFunc(src.Prop). Missing child type map → Mapper.MapInternal creates a default instance (Activator.CreateInstance(destinationPropertyType)) and assigns it.

4.5 Collection mapping

•	Where: CollectionMapper.cs (entire), Mapper.MapInternal lines 420–454, Mapper.MapNoContext lines 173–220, TypeMapSealer.BuildNestedMapFunc lines 390–429 (MapCollectionWithIdentity inlined).

•	How: Source must be IEnumerable; destination must be IList-shaped or an array. List factories and Add delegates are compiled per-element-type and cached in three ConcurrentDictionarys. Capacity-aware factory is used when source is ICollection. For arrays, the list is copied into a fresh Array.CreateInstance(...).

•	Edges: string is excluded explicitly (lines 70, 105). Null source collection becomes an empty destination collection via TryCreateEmptyCollection. Identity tracking within one source list reuses earlier results (MapCollectionWithIdentity does an O(i) scan back — clever but quadratic for very large lists; see §13).

4.6 Array/List/IEnumerable handling

•	IsCollectionType returns true for arrays and IEnumerable (except string).

•	GetCollectionElementType reads Type.GetElementType() for arrays or GenericArguments\[0] for generics.

•	Returns an Array for T\[] destination, otherwise a List<T>.

•	Not handled: Dictionary<TK,TV>, ImmutableArray/ImmutableList, IReadOnlyList, HashSet, Queue/Stack — any of these as the destination element type will route through the list factory and may fail at the final cast.

4.7 ReverseMap

•	Where: MappingExpression.ReverseMap() lines 103–109.

•	How: Allocates a fresh TypeMap(typeof(TDestination), typeof(TSource), \_typeMap.MemberList), attaches it as ReverseTypeMap, and adds it to the same profile's \_typeMaps list (\_siblingTypeMaps?.Add). Then returns a new MappingExpression over the reverse map — so users can chain configuration onto the reverse.

•	Edges: Custom forward MapFrom/Ignore is not inverted automatically; the reverse map is convention-only unless the user configures it. Verified by MappingExpression\_ReverseMap\_ParityTests.ReverseMap\_WithForMember\_OnForwardMap\_ShouldMatchAutoMapper.

4.8 Custom converters

•	Two flavors:

•	Lambda: ConvertUsing(Func<TSource, TDestination> converter) → wrapped in LambdaTypeConverter<TSource,TDestination> (sealed). \_typeMap.TypeConverter is set.

•	Type-based: ConvertUsing<TConverter>() where TConverter : ITypeConverter, new() → stores typeof(TConverter) in \_typeMap.TypeConverterType. A fresh instance is created every MapInternal call (Mapper.cs line 318) — matches AutoMapper 6.x behavior comment.

•	The compiled fast path is skipped when either converter is set (TypeMapSealer.Seal lines 37–48).

4.9 Conditional mapping

•	Condition(Func<TSource,bool>) stores Func<object,bool> on MemberConfiguration.Condition.

•	At runtime: MapNoContext and MapInternal check if (config.Condition != null \&\& !config.Condition(source)) continue;.

•	At compile time: BuildFullMapFunc wraps assignment with Expression.IfThen(conditionCall, assignExpr) (line 294) so even fast-path mappings honor it.

•	Edge: Receives only the source, not the destination (limitation vs AutoMapper's ConditionContext).

4.10 Null handling

•	Source object null → ArgumentNullException (Mapper.Map lines 51, 74, 85).

•	Source member null:

•	Collection destination → empty collection via CollectionMapper.TryCreateEmptyCollection.

•	Non-collection: if AllowNull (default true) → assign null; else continue (preserve existing value).

•	Source nested object null → fast path emits null check; slow path doesn't enter nested branch (value == null path runs first).

4.11 Circular reference handling

•	Detection: ResolutionContext.\_instanceCache keyed by InstanceCacheKey(RuntimeHelpers.GetHashCode(source), destType) with ReferenceEquals on equality.

•	Cycle break: Before constructing the destination, MapInternal calls TryGetMappedInstance and returns the cached object if found.

•	Hardening: GlobalMaxDepth = 500 const int in Mapper.cs line 11 — when exceeded, MapInternal throws to prevent stack overflow.

•	Important: Cycle detection is never invoked on the fast path. The fast path is only ever assigned when HasCircularReference proved no cycle.

4.12 Configuration validation

•	AssertConfigurationIsValid() is opt-in. Run at startup if you want fail-fast.

•	Member-level validation rules: empty (last-wins is allowed). MemberList: Destination and Source rules described in §2.5.

•	Gap: No "no map registered for nested type" check, no "MapFrom expression references invalid member" beyond what SourceMemberAccessVisitor collects.

4.13 Mapping context (ResolutionContext)

•	Constructed per MapInternal call when needed (Mapper.cs line 291).

•	Child contexts share \_instanceCache, \_typeDepths, \_items by reference (lazy where possible).

•	Exposes Items, Depth, Parent, Mapper, SourceType, DestinationType, Configuration.

•	Limitation: The runtime never passes the context to user-supplied Func<object,object?> resolvers/conditions — only ITypeConverter.Convert sees it. So you cannot read context.Items\["tenantId"] from inside a ResolveUsing lambda. (See §13.)

4.14 Caching

•	Five caching layers exist:

1\.	MapperConfiguration.\_typeMaps (per config) — config-time metadata.

2\.	MapperConfiguration.\_overrideIndex (per config) — secondary index for As<T>.

3\.	Mapper.\_mapFuncCache (ConcurrentDictionary, per mapper) — (srcRuntimeType,destType) → Func<object,object>.

4\.	Mapper.\_lastMapFunc (volatile, per mapper) — single hot-path entry.

5\.	ReflectionHelper.\_simpleTypeCache and CollectionMapper.\_listFactoryCache/\_listFactoryWithCapacityCache/\_typedAddCache (ConcurrentDictionary, process-wide statics).

•	No invalidation API: caches live as long as their owner.

4.15 Expression tree generation

•	Heavy lifting in TypeMapSealer. Key patterns:

•	BuildFactory: Expression.Lambda<Func<object>>(Expression.Convert(Expression.New(destType), typeof(object))).

•	BuildSetter: typed cast → Expression.Property → Expression.Assign → Expression.Convert of value.

•	ParameterReplacer (private ExpressionVisitor at line 490): rewrites the user's MapFrom lambda parameter to use the typed local in BuildFullMapFunc/BuildNestedMapFunc.

•	Fallback: try/catch around BuildFullMapFunc and BuildNestedMapFunc returns null on any failure → mapper drops to the next-slower path.

4.16 Compiled delegate execution

•	All compiled delegates have shape Func<object>, Action<object,object?>, Func<object,object?>, or Func<object,object>. They take object (not generics) so a single delegate type can be cached uniformly per type pair without generic re-instantiation.

•	Boxing risk: BuildDirectCopy only fires when sourceProp.PropertyType == destProp.PropertyType — and uses typed property access on both sides, so primitives don't box.

4.17 AutoMapper parity tests

•	Strategy in ParityTestBase.AssertParity (lines 9–64):

•	Build identical AutoMapper + Custom configs.

•	Run both maps on the same input.

•	customResult.Should().BeEquivalentTo(autoResult) (FluentAssertions, structural compare).

•	Cross-check autoMap.Ignored / CustomExpression / CustomResolver against customTypeMap.IsMemberExplicitlyConfigured and MemberConfiguration.IsIgnored.

•	Verify TypeConverterType presence parity, AfterMapActions presence parity, and runtime destination type parity.

•	For validation parity (AssertConfigurationParity): catches both sides, normalizes exceptions into semantic buckets ("duplicate\_mapping", "memberlist\_violation", etc.) so AutoMapper's exception types don't have to match — only the meaning matches.

4.18 BenchmarkDotNet benchmarks

•	See §5.

\---

5\. Performance Deep Dive

5.1 Why it is faster (on the configured benchmarks)

1\.	Single compiled lambda for flat objects: Func<object,object> from Expression.MemberInit — one delegate call, one new, then assignments inlined by the JIT. AutoMapper 6.2.2 still walks per-property maps and allocates a ResolutionContext per call.

2\.	No context allocation on the fast path and the no-context path. Your SimpleMap allocates only the destination object itself (\~72 B for SimpleDestination).

3\.	Per-mapper delegate cache with volatile last-hit cache removes the ConcurrentDictionary lookup from the steady-state hot loop.

4\.	Pre-resolved nested/collection type maps at seal time (PostSealResolveNestedMappings) → zero FindTypeMap lookups per call when on no-context path.

5\.	Capacity-aware list construction + cached Add delegate avoids List<T> resizes and interface dispatch.

5.2 What causes lower allocation

•	No ResolutionContext on hot path → no dictionary, no items dict, no InstanceCacheKey boxing.

•	MemberInit path produces only one allocation: the destination instance itself.

•	LambdaTypeConverter allocations are amortized: the closure is built once at config time.

•	Setup-time win (CM 356 KB vs AM 13.5 MB for the same 4 profiles, ratio 0.03) comes from compiling only the delegates you need and not building AutoMapper's full execution plan + planner trees.

5.3 Hot path (steady state)



Mapper.Map<TDestination>(source)

&#x20;→ \_lastMapFunc != null \&\& types match (volatile read, two ref compares)

&#x20;  → cast and invoke compiled CachedFastMapFunc

&#x20;    → new TDest { ... }   // single allocation

Anything outside this path is colder.

5.4 Delegate caching impact

•	The first call per (srcType, destType) resolves the delegate via ResolveMapFunc (one FindTypeMap + branching). Then cached.

•	Without caching, every call would re-traverse FindTypeMap and re-decide between the three paths → 2–5× slowdown for trivial maps.

5.5 Fast-path vs context-aware

Path	When chosen	Allocations	Supports

CachedFastMapFunc	Flat (all simple types), no MaxDepth, no TypeConverter	Dest only (+ closures for resolvers if configured)	MapFrom, Resolver, Condition, UseValue, Ignore, AfterMap

MapNoContext	Has nested/collection but no cycle and no MaxDepth	Dest + child dests + capacity-sized List + lazy identity dict per collection	Same as above plus nested/collection mapping

MapInternal	Anything else (cycle, MaxDepth, TypeConverter, destination overload, depth>50)	Dest + ResolutionContext + caches + per-level nested contexts	Everything

5.6 When it may be slower

•	AutoMapper wins on Collection (your own results: AM 194 ns vs CM 262 ns, ratio 1.35). The reason: AutoMapper 6.2.2 emits a fully-inlined per-element loop with direct Add; your MapCollectionWithIdentity does an O(i) reference scan for identity each element and uses generic helper method invocation via reflection (MakeGenericMethod once at seal, but the call still goes through Func<object,object>).

•	ConfigSetup: AM 3.94 ms vs CM 4.73 ms (ratio 1.20). The two extra post-seal passes + fixed-point loop are the cost — won back massively in memory (\~38× less allocation) but slower wall-clock for setup.

•	Anything not covered by the fast path (a real ITypeConverter with Activator.CreateInstance per call, or anything with MaxDepth).

5.7 BenchmarkDotNet usage

•	\[MemoryDiagnoser], \[Orderer(SummaryOrderPolicy.FastestToSlowest)], \[GroupBenchmarksBy(BenchmarkLogicalGroupRule.ByCategory)], \[CategoriesColumn], \[MinColumn, MaxColumn, MedianColumn] (MapperBenchmarks.cs lines 10–14).

•	\[GlobalSetup] builds both mappers and pre-fills \_reverseMapDest to ensure both sides receive equivalent inputs.

•	14 categories, two methods per category (AM baseline, CM under test). \[Benchmark(Baseline = true)] on AM yields Ratio and Alloc Ratio.

•	Program.cs simply calls BenchmarkRunner.Run<MapperBenchmarks>().

5.8 What MemoryDiagnoser measures

•	Allocated: bytes per operation (managed heap only, from BCL's GC.GetAllocatedBytesForCurrentThread).

•	Gen0/Gen1/Gen2: number of collections per 1k operations (so 0.0115 Gen0 means \~11.5 Gen0 collections per 1M ops).

•	Does NOT measure: stack allocations, unmanaged memory, finalizer pressure, working set.

5.9 Interpreting the columns

Column	Meaning	Defense point

Mean	Average per op	Stable signal.

Error	99.9% CI half-width	Should be a small fraction of mean.

StdDev	Run-to-run noise	<5% of mean = clean.

Min/Max/Median	Distribution shape	Median ≈ Mean implies no skew.

Ratio	This / Baseline	Single most-cited number — anchor every claim to it.

Gen0	Young-gen collections	Lower = less GC pressure.

Allocated	Bytes per op	Drives Gen0.

5.10 How to defend results in interview

•	Cite the exact host: "BenchmarkDotNet 0.15.8, .NET 8.0.26 on Intel Core Ultra 7 268V, single configuration, in-process JIT, no debugger." (Top of your \*-report-github.md.)

•	Cite the ratio: "0.65–0.91 on the steady-state mapping benchmarks; 1.20 on config setup, 1.35 on collections — both with explicit explanations."

•	Be honest: "I'm slower on Collection because my identity tracking does an O(i) scan and my collection codegen is less specialized than AutoMapper's."

•	Note that \[MemoryDiagnoser] was on for every run → no missing data.

5.11 Pitfalls / unfair comparisons (be honest)

•	Both libraries are single-threaded, single-process — no contention testing.

•	Categories are isolated; no benchmark covers "build a mapper, map once, throw it away" cold start. AutoMapper amortizes its planner across calls; your library does the same.

•	Some benchmarks (AfterMap, Condition) use the fast path with a tiny per-property action — that's where you win biggest. A real-world map with Func<TSource, ResolutionContext, TDestination, TMember> resolvers would shift the picture.

•	LargeObject has only 30 properties of two types. Real DTOs with 100+ properties + Nullable<T> mixed in would re-balance.

5.12 Suggested rigor improvements

•	Add \[Params(1, 10, 100, 1000)] to the Collection category so element count is varied.

•	Add a benchmark that uses \[MarkdownExporter, RPlotExporter] and runs in isolated processes with \[SimpleJob(launchCount: 3, warmupCount: 5, iterationCount: 10)] to detect cross-run jitter.

•	Add a cold-start benchmark that measures new MapperConfiguration(...).CreateMapper().Map<…>(...) end-to-end (currently ConfigSetup excludes the first map call).

•	Add a multi-threaded scenario (Parallel.For calling Map) to validate ConcurrentDictionary + \_lastMapFunc under contention.

•	Track Tier1Jit impact: use \[DisassemblyDiagnoser] to confirm the fast path doesn't deopt.

\---

6\. Correctness and Testing

6.1 Test structure

•	One xUnit project: tests/Mapping.Tests (folders mirror src).

•	Two file-naming patterns: \*\_Tests.cs (unit) and \*\_ParityTests.cs (parity against AutoMapper 6.2.2).

•	Diagnostic probe: \_DiagnosticTests/AutoMapper622\_BehaviorProbe.cs — used to lock in quirks like MaxDepth(0) being ignored.

6.2 Parity vs unit tests

•	Unit tests: validate the library's own semantics without comparing to AutoMapper. They are the source of truth for your design intent (e.g., "AllowNull defaults to true").

•	Parity tests: extend ParityTestBase. They simultaneously configure both libraries and assert: output equivalence, IsExplicit parity, IsIgnored parity, presence of TypeConverterType and AfterMapActions, and runtime destination type parity. This guarantees you can drop your library in as an AutoMapper 6.2.2 replacement for the scenarios you cover.

6.3 Why parity is valuable

•	Provides an executable spec: you don't have to read AutoMapper docs, the tests pin the behavior.

•	Catches drift when you refactor your library — if AutoMapper still does X, you still must do X.

•	Doubles as a bug-finder for AutoMapper edge cases (your MaxDepth(0) test asserts the AutoMapper quirk).

6.4 Important edge cases covered (file: test pattern)

•	Mapper\_DepthAndCircularRef\_Tests: direct self-loop, mutual A↔B, distinct objects not sharing cache, MaxDepth interaction with cycles, sibling branches independently truncated.

•	Mapper\_NestedMapping\_Tests: existing-destination override, multi-level nesting, default child when no map.

•	Mapper\_CollectionMapping\_Tests: list-to-list, list-to-array, nested collections, large lists.

•	Mapper\_AllowNull\_Tests: explicit AllowNull semantics.

•	Mapper\_Condition\_Tests: condition false skips assignment.

•	Mapper\_AfterMap\_Tests: post-mapping action ordering.

•	Mapper\_AsOverride\_Tests: As<TDerived>() and the override index.

•	Mapper\_ConvertUsing\_Tests and \_ClassBased\_Tests: lambda and ITypeConverter flavors.

•	Mapper\_IgnoreAndConstant\_Tests: Ignore + UseValue interactions (last-wins).

•	Mapper\_ExplicitMapping\_Tests: ForMember/ForAllOtherMembers/ForAllMembers precedence.

•	Configuration/\*: profile registration, duplicate handling, validation rules, ReverseMap parity.

6.5 Possibly missing edges (you should mention these proactively)

•	Open generic mappings (CreateMap(typeof(Foo<>), typeof(Bar<>))) — not implemented, not tested.

•	Value types as TSource / TDestination — blocked by the where TSource : class constraint on CreateMap.

•	Nullable<T> ↔ underlying — partially handled by IsSimpleType cache but no explicit tests around Nullable conversion to non-nullable when source is null.

•	Thread safety under concurrent Map calls (no parallel test verifying \_lastMapFunc + \_mapFuncCache correctness under stress).

•	Mapping into existing collection — current behavior overwrites; not tested.

•	Behavior when the destination has both a writable prop and matching backing field with different name.

•	Anonymous types as source.

•	Inheritance: source is a derived class, map registered for base — verified by Map<TDestination> using source.GetType(), but no explicit base-class polymorphism tests around Include.

6.6 Additional tests interviewers expect

•	Concurrency stress (Parallel.For).

•	Memory pressure: assert allocation count matches the benchmark via GC.GetAllocatedBytesForCurrentThread.

•	Property-based tests with FsCheck/CsCheck to fuzz arbitrary type graphs.

•	Serialization round-trip (source → dest → source via ReverseMap) parity.

6.7 xUnit usage

•	\[Fact] and \[Theory] with \[InlineData] / \[MemberData].

•	FluentAssertions for readable assertions (Should().Be(...), BeEquivalentTo(...)).

•	Moq is referenced but used sparingly (mostly the real mapper is tested).

•	Test base class (ParityTestBase) factors out the AutoMapper setup + cross-assert plumbing.

6.8 How to talk about "624+ tests"

"I have 614 declared \[Fact]/\[Theory] attributes, which expand to 624+ runtime cases because several theories carry multiple \[InlineData] rows. About two-thirds are parity tests against AutoMapper 6.2.2 — they build identical configurations against both libraries, run the same input, and assert structural equivalence plus configuration parity (Ignore/MapFrom/TypeConverter/AfterMap presence). The remaining third are unit tests that pin behavior unique to my library, like the three-path execution dispatch and MaxDepth interaction with the identity cache."

\---

7\. .NET and C# Concepts Used (with where to point in the code)

Concept	Evidence	Talking point

Generics (open + closed)	IMappingExpression<TSource, TDestination>, LambdaTypeConverter<TSource, TDestination>, generic Map<TDestination>	Constraint where TSource : class where TDestination : class enforces reference-type semantics.

Reflection	ReflectionHelper, Type.GetProperty(...BindingFlags.Instance \\| BindingFlags.Public) in BuildSourceGetter, BuildSetter, BuildDirectCopy	Used only at seal time, never on the hot path.

Expression trees	System.Linq.Expressions throughout TypeMapSealer.cs (Expression.New, Expression.Property, Expression.Assign, Expression.MemberInit, Expression.IfThen, Expression.Block, Expression.Invoke, custom ExpressionVisitor subclass ParameterReplacer)	The library compiles expression trees into delegates exactly once per TypeMap.

Delegates	Func<object>, Func<object,object>, Action<object,object?>, Action<object,object>, Func<object,bool>	Boxed-object delegates avoid generic-instantiation explosion in the cache.

Compiled lambda expressions	.Compile() calls in BuildFactory, BuildSetter, BuildSourceGetter, BuildDirectCopy, BuildFullMapFunc, BuildNestedMapFunc, CollectionMapper.GetListFactory\*, GetTypedAdd	Tier-1 JITted code — \~2–5× faster than PropertyInfo.GetValue/SetValue.

Dictionaries	Dictionary<string, MemberConfiguration> (StringComparer.Ordinal), Dictionary<(Type,Type), TypeMap>, Dictionary<InstanceCacheKey, object>	Choice of Ordinal vs OrdinalIgnoreCase is deliberate: storage uses Ordinal (already-canonicalized names) but discovery uses OrdinalIgnoreCase.

ConcurrentDictionary	Mapper.\_mapFuncCache, CollectionMapper.\_listFactoryCache/\_listFactoryWithCapacityCache/\_typedAddCache, ReflectionHelper.\_simpleTypeCache	Lock-free reads, striped writes, ideal for "configure once / read many" caches.

Thread safety	Mapper is read-only after construction except for \_mapFuncCache (concurrent) and \_lastMapFunc (volatile, idempotent write)	"Safe to share one IMapper across threads" is your value proposition.

Immutability	TypeMap is mutated only during config/seal; all Cached\* properties are private set and only assigned by Seal()	After CreateMapper(), the configuration graph is effectively immutable from the user's perspective.

Object graph traversal	Mapper.MapInternal recursion through CreateNestedContext, HasCircularReference DFS in MapperConfiguration	Both runtime and config-time use graph traversal.

Recursion	MapInternal, MapNoContext, HasCircularReference	Bounded by GlobalMaxDepth = 500 and MapNoContext's depth > 50 fallback.

Reference equality	ReferenceEquals in InstanceCacheKey.Equals, ReferenceEqualityComparer.Instance in MapNoContext identity dict, RuntimeHelpers.GetHashCode for identity hashing	Required so two equal-by-value but distinct instances do not collide in the cycle cache.

LINQ	Used at config time in MapperConfiguration.cs, ReflectionHelper.cs, ConfigurationValidator.cs (e.g., Where, Select, ToDictionary, ToHashSet, First, Single)	Avoided on the hot path.

Nullable reference types	<Nullable>enable</Nullable> in csproj; object?, Func<object, object?>?, MemberConfiguration? throughout	Public API is null-aware.

Value types vs reference types	InstanceCacheKey is a readonly struct (zero-allocation key for the dictionary). TypeMap/Mapper are classes (shared state).	The cache key design is the textbook "struct key to avoid GC pressure".

Boxing/unboxing	Expression.Convert(value, typeof(object)) boxes value types when constructing the boxed Func<object,object> delegates. BuildDirectCopy avoids this by using typed source/dest with same prop type.	You pay one box for property reads on slow paths; fast path avoids it via MemberInit.

Covariance / contravariance	Not heavily used; IReadOnlyDictionary<string, MemberConfiguration> exposed as the TypeMap.MemberConfigurations public surface, leveraging variance on the read-only interface.	Not a major theme.

Dependency injection	Not part of the library. The library only depends on the BCL. AutoMapper has ServiceCollectionExtensions; you do not.	Could be added in a separate Extensions project (folder exists at src\\Extensions\\, currently empty).

\---

8\. Data Structures and Algorithms

8.1 Data structures

Structure	Used for	Why

Dictionary<(Type,Type), TypeMap>	\_typeMaps, \_overrideIndex	O(1) lookup keyed by type pair.

Dictionary<string, MemberConfiguration>	TypeMap.\_memberConfigurations	Per-destination-property O(1) lookup, StringComparer.Ordinal.

Dictionary<InstanceCacheKey, object>	ResolutionContext.\_instanceCache	Identity-based O(1) cycle detection.

Dictionary<(Type,Type), int>	ResolutionContext.\_typeDepths	Per-type-pair recursion counters for MaxDepth.

ConcurrentDictionary<(Type,Type), Func<object,object>>	Mapper.\_mapFuncCache	Lock-free reads for delegate dispatch.

ConcurrentDictionary<Type, Func<IList>> etc.	CollectionMapper factories	Process-wide cache keyed by element type.

volatile MapFuncCacheEntry?	Mapper.\_lastMapFunc	Single-slot inline cache.

PropertyMapping\[]	TypeMap.CachedPropertyMappings	Array (not List<T>) — direct indexing in the hot for-loop, no enumerator alloc.

Action<object,object>\[]	TypeMap.CachedAfterMapActions	Same rationale; indexed for-loop.

Lazy<T>	PropertyMapping.\_lazy\*Delegates	Defer compilation cost until first slow-path use.

HashSet<TypeMap> (DFS visited)	HasCircularReference	O(1) cycle detection.

HashSet<PropertyInfo>	SourceMemberAccessVisitor	Deduplicate properties referenced in MapFrom expression.

8.2 Time / space complexity

Operation	Time	Space

MapperConfiguration constructor	O(M·P) compilation + O(M) for two post-seal passes + O(M·E) DFS for cycle detection, where M = #TypeMaps, P = avg #props, E = avg #nested edges	O(M·P) cached delegates and arrays

FindTypeMap(srcType, destType)	O(1) expected	O(M) keys

Mapper.Map<TDestination>(source) cold	O(P) on first call (resolves + compiles already cached delegates)	O(1) per cache entry added

Steady-state Map (fast path)	O(1) — one delegate invocation	O(1) — single destination allocation

Steady-state Map (no-context)	O(P) where P = #props in graph touched	O(P) destinations + per-collection list

Steady-state Map (full context)	O(P + E) plus O(N) on identity dict where N = #unique source refs in graph	O(N) identity cache

Nested map	O(depth · P)	O(depth) stack + O(N) identity cache

Collection map	O(K · cost\_per\_element); identity scan within MapCollectionWithIdentity is O(K²) worst case	O(K) for the destination list

Cycle detection at seal	O(M · E) DFS	O(M) visited set

8.3 Recursion depth concerns

•	Mapper.MapInternal recurses for nested objects and per-element via CollectionMapper.TryMapCollection's callback. Bounded by:

•	GlobalMaxDepth = 500 (hard).

•	typeMap.MaxDepth (configurable per map).

•	MapNoContext falls back to MapInternal past depth > 50 (line 131) — a soft floor.

8.4 Graph traversal aspects

•	Config-time DFS: HasCircularReference (line 148) walks NestedTypeMap/CollectionElementTypeMap edges to decide eligibility. Uses a visited set, removes-on-backtrack to avoid false positives across siblings.

•	Runtime: \_instanceCache plays the same role but at object-identity level instead of type-level.

8.5 Caching strategy and invalidation

•	No invalidation API. Caches are immutable for the lifetime of the owning object (MapperConfiguration for \_typeMaps, Mapper for \_mapFuncCache, process for CollectionMapper/ReflectionHelper).

•	Mitigation: discard the IMapper/MapperConfiguration when configuration changes. The static ConcurrentDictionarys on CollectionMapper and ReflectionHelper are pure functions of Type so they never need eviction.

\---

9\. Low-Level Code Walkthrough — The 10 Methods Interviewers Will Ask About

9.1 TypeMap.Seal() (TypeMap.cs lines 118–127)

•	Input: none (uses this).

•	Output: writes CachedFactory, CachedPropertyMappings, CachedAfterMapActions, CachedTypeConverterInstance, EligibleForFastPath, CachedFastMapFunc.

•	Side effects: triggers expression compilation via TypeMapSealer.

•	Throws: nothing directly (sealer swallows compile failures and returns nulls).

•	Complexity: O(P) work, P = #destination properties; can call .Compile() up to 5 times (factory + setter + getter + direct copy + dest getter), most via Lazy<T>.

•	Why: separates "rules captured" from "rules compiled" — a clean two-phase config model.

9.2 TypeMapSealer.Seal(TypeMap) (lines 18–59)

•	Input: the configured TypeMap.

•	Output: a SealResult record.

•	Side effects: mutates MemberConfiguration.CompiledSourceExpression for each member via CompileSourceExpression.

•	Complexity: O(P) per map; one allocation for the result, one array of PropertyMapping.

•	Why: extracting compilation out of TypeMap keeps it testable and small.

9.3 TypeMapSealer.BuildFullMapFunc(...) (lines 205–323)

•	Input: source/dest types, the PropertyMapping\[], optional AfterMap actions.

•	Output: a Func<object,object> or null on failure.

•	Side effects: none, pure compilation.

•	Complexity: O(P) expression nodes; .Compile() cost dominates (\~milliseconds for tiny lambdas).

•	Why: two strategies — MemberInit for the trivial all-direct-copy case (zero conditional code), else a Block with per-member IfThen for conditions, Convert for type mismatches, and inlined Invoke for resolvers.

•	Risk: the try/catch at line 318 silently swallows compile errors — interviewers will ask "what if expression compilation fails?" — your answer is "fall back to the per-property loop in MapInternal/MapNoContext".

9.4 MapperConfiguration.PostSealResolveNestedMappings() (lines 52–146)

•	Input: implicit (\_typeMaps).

•	Output: enriches PropertyMapping.NestedTypeMap / CollectionElementTypeMap / CollectionListFactory\* / CollectionTypedAdd and sets TypeMap.EligibleForNoContextPath.

•	Side effects: caches source-type property dictionaries to avoid re-reflection within the same map.

•	Complexity: O(M·P) for resolution + O(M·E) DFS.

•	Why: enables the entire no-context fast path; this is the heart of your low-allocation story.

9.5 MapperConfiguration.HasCircularReference(...) (lines 148–169)

•	Input: starting TypeMap, mutable HashSet<TypeMap> visited.

•	Output: bool.

•	Side effects: mutates visited.

•	Complexity: O(M+E) per call.

•	Why: prevents marking a recursive graph as fast-pathable — that would either stack-overflow or produce wrong output.

9.6 Mapper.Map<TDestination>(object source) (lines 49–58) + MapSlow (60–70)

•	Input: any object.

•	Output: typed destination.

•	Side effects: updates \_lastMapFunc; populates \_mapFuncCache on miss.

•	Throws: ArgumentNullException for null source; InvalidOperationException if no map registered.

•	Complexity: O(1) hot path; O(1) amortized for cache miss.

•	Why: minimizes call-site overhead — one volatile load + two ref-equals.

9.7 Mapper.ResolveMapFunc(...) (lines 94–119)

•	Input: source/dest types.

•	Output: a Func<object,object> chosen from four candidates (fast func, TypeConverter, no-context delegate, full MapInternal delegate).

•	Complexity: O(1) plus one FindTypeMap.

•	Why: collapses the dispatch decision into a delegate so subsequent calls pay nothing for it.

9.8 Mapper.MapInternal(...) (lines 247–517)

•	Input: source, destination type, optional destination, optional context, current depth.

•	Output: mapped destination.

•	Side effects: allocates ResolutionContext if missing; mutates context cache + depth tracker; runs AfterMap actions.

•	Throws: InvalidOperationException if no map or depth > GlobalMaxDepth.

•	Complexity: O(P + nested), depth-bounded.

•	Why: catch-all path. Handles every feature including MaxDepth, ITypeConverter, existing-destination overload, nested collections under a circular graph, and AfterMap.

9.9 Mapper.MapNoContext(...) (lines 129–245)

•	Input: source, type map, dest type, depth.

•	Output: mapped destination.

•	Side effects: lazy identity dict per collection (key: source ref, value: mapped dest).

•	Complexity: O(P) per call + O(K) per collection.

•	Why: combines "no context allocation" with full nested/collection semantics. Bails to MapInternal past depth 50 to avoid pathological cases without context.

9.10 ResolutionContext.CacheMappedInstance + TryGetMappedInstance + InstanceCacheKey (lines 23–46, 124–140)

•	Input: source object reference, destination type, mapped destination.

•	Output: side-effect store / boolean lookup.

•	Side effects: writes to \_instanceCache.

•	Complexity: O(1).

•	Why: RuntimeHelpers.GetHashCode is identity-based (object.GetHashCode would be overridden on user types and break detection); paired with ReferenceEquals for collision resolution.

Two honorable mentions: CollectionMapper.MapCollectionWithIdentity (the O(i²) scan) and MappingExpression.ReverseMap (sibling list mutation).

\---

10\. Interview Questions (≥150)

A. Beginner (Q1–Q25)

1\.	What is object mapping?

Answer: Copying data from one object shape to another, typically across layers (entity↔DTO). Eliminates manual copy code.

Avoid: confusing with serialization.

Follow-up: when wouldn't you use a mapper?

2\.	Why not just hand-write dto.Name = entity.Name?

Answer: Works for tiny models; doesn't scale across 200-property objects, doesn't centralize convention rules, doesn't support runtime polymorphism. Mappers reduce repetition and provide a single point of change.

3\.	What does CreateMap<S,D>() do in your library?

Answer: Adds a TypeMap(S, D, MemberList.Destination) to the current MapperProfile, discovers convention-based members in the ctor (DiscoverImplicitMembers), and returns an IMappingExpression<S,D> for further fluent configuration.

4\.	What does MapperProfile do?

Answer: It is a base class that holds a list of TypeMaps. Users derive from it and call CreateMap in the ctor. The MapperConfiguration flattens all profile maps at build time.

5\.	What's the difference between MapperConfiguration and IMapper?

Answer: MapperConfiguration is the immutable, sealed metadata graph (built once, expensive). IMapper is the cheap runtime entry point that holds a reference to the configuration and a delegate cache. You build the config once and create many mappers (typically one per app).

6\.	What does IConfigurationProvider expose?

Answer: AssertConfigurationIsValid() and CreateMapper(). Mirrors AutoMapper's contract.

7\.	What does Ignore() do?

Answer: Sets MemberConfiguration.IsIgnored = true and MarkExplicit(). Both fast-path codegen and the per-property loop short-circuit on it.

8\.	What does UseValue(x) do?

Answer: Stores a constant ConstantValue and sets HasConstantValue = true. Last-wins — it clears prior IsIgnored, SourceExpression, Resolver (MemberConfigurationExpression.cs lines 29–39).

9\.	What does MapFrom(s => s.X + s.Y) do?

Answer: Stores the user's LambdaExpression; the sealer compiles it once into CompiledSourceExpression and (in the fast path) inlines the body into the compiled map function via ParameterReplacer.

10\.	What does ForMember actually do internally?

Answer: Extracts the destination member name from the expression via ReflectionHelper.GetMemberName, gets-or-creates the MemberConfiguration, marks it MarkConfigured(), builds a MemberConfigurationExpression, and invokes the user's memberOptions action against it.

11\.	What is ConvertUsing?

Answer: Wholesale override — the entire destination is produced by the supplied converter. Two forms: lambda (wrapped in LambdaTypeConverter) and ITypeConverter type-based (instantiated per Map call).

12\.	What is AfterMap?

Answer: A post-mapping side-effect action (source, destination) => …. Cached as Action<object,object>\[] and invoked at the end of every path.

13\.	What is As<T>()?

Answer: Tells the mapper to instantiate TDerivedDestination instead of TDestination. Implemented by setting TypeMap.DestinationOverrideType and populating a secondary \_overrideIndex after sealing.

14\.	What is ReverseMap?

Answer: Creates a new TypeMap(D,S), attaches it as ReverseTypeMap, adds it to the same profile's sibling list, returns a fresh MappingExpression<D,S> so the reverse can be configured further.

15\.	What is the role of MemberList?

Answer: Controls validation. Destination requires every dest prop be mapped, Source requires every source prop be consumed (by convention or MapFrom), None skips checks.

16\.	What is ResolutionContext?

Answer: Per-mapping-call metadata: source/dest types, current depth, parent context, user Items, identity cache for cycles, and lazy per-type depth tracker.

17\.	What is IMapper?

Answer: The runtime contract with three Map overloads. The single implementation is Mapper (sealed).

18\.	What is Mapper?

Answer: Sealed; holds the MapperConfiguration, a ConcurrentDictionary delegate cache, and a volatile single-slot cache. Threadsafe by design.

19\.	What does MapperConfiguration.AssertConfigurationIsValid do?

Answer: Delegates to ConfigurationValidator.Validate(\_typeMaps.Values) which enforces MemberList rules and structural sanity.

20\.	What is IsExplicit vs IsConfigured?

Answer: IsExplicit = touched by a member-creating API like ForMember/MapFrom/Ignore/UseValue. IsConfigured = touched by any config API. AutoMapper distinguishes "user said something specific" from "user said anything at all".

21\.	What is AllowNull?

Answer: When true (default), a null source value overwrites the destination property; when false, the null is skipped (existing destination value preserved).

22\.	How are collection elements mapped?

Answer: Source IEnumerable → destination List<T> via cached factory; per-element delegate is the child CachedFastMapFunc (or recursive MapInternal). Arrays get a final Array.CreateInstance + copy.

23\.	How do you avoid Activator.CreateInstance overhead?

Answer: TypeMap.CachedFactory is a compiled Func<object> from Expression.New(destType) — JITted, no reflection per call.

24\.	What is the default MemberList?

Answer: MemberList.Destination (in MapperProfile.CreateMap's parameter default).

25\.	Where is the \[Fact] count for your tests?

Answer: 614 declared, 624+ expanded at runtime (theories with multiple \[InlineData]).

B. Intermediate (Q26–Q60)

26\.	Walk me through what happens during new MapperConfiguration(cfg => …).

Answer: (i) cfg is built via MapperConfigurationExpression, \_profiles populated; (ii) BuildTypeMaps flattens to \_typeMaps; (iii) each map's Seal() runs (compilation); (iv) PostSealResolveNestedMappings resolves nested/collection metadata and decides EligibleForNoContextPath; (v) PostSealBuildNestedFastMapFuncs runs a fixed-point loop building CachedFastMapFunc for eligible maps.

27\.	Why two post-seal passes?

Answer: Pass 1 must finish for every map so that HasCircularReference can walk the full graph. Pass 2 builds the compiled nested funcs bottom-up — children must succeed before parents can inline them.

28\.	Why a fixed-point loop in PostSealBuildNestedFastMapFuncs?

Answer: A parent map can only be compiled when its children's CachedFastMapFunc is non-null. Iterating until no further progress is the simplest topological fix.

29\.	Why is TypeMap sealed?

Answer: Performance (sealed types can have non-virtual dispatch), correctness (downstream can't override and break the seal contract), API clarity ("this is the only TypeMap").

30\.	Why ConcurrentDictionary over Dictionary with lock?

Answer: After warm-up, reads are lock-free in ConcurrentDictionary; Dictionary+lock would serialize every read. The cache is read >>> write.

31\.	What does \_lastMapFunc buy you?

Answer: Skips even the ConcurrentDictionary.TryGetValue cost (hash + bucket access) when the previous call was for the same (srcType, destType). Hot loops mapping one type at a time see \~10–15 ns savings — measurable at single-digit ns scale.

32\.	Why volatile?

Answer: To guarantee freshness of the reference write/read across threads without locks. Worst case: two threads race and one overwrites the other's entry — no correctness issue because the entry is derived only from the type pair.

33\.	Why is InstanceCacheKey a readonly struct?

Answer: No heap allocation per key, no boxing in the dictionary. Equality uses identity hash + ref equals — cheap.

34\.	Why RuntimeHelpers.GetHashCode?

Answer: It returns the identity hash of the object, ignoring any user-overridden GetHashCode. Critical: two equal-by-value but distinct instances must hash differently so they aren't treated as the same node in the cycle cache.

35\.	Why is MapNoContext separate from MapInternal?

Answer: To avoid allocating ResolutionContext when static analysis proved no cycle could exist. Most real maps fall here, and the per-call savings compound.

36\.	Why fall back to MapInternal past depth 50 in MapNoContext?

Answer: Defense in depth. Even without a known cycle, very deep graphs benefit from the identity cache to avoid duplicate work. 50 is a heuristic.

37\.	What's the difference between CachedFastMapFunc and the CachedPropertyMappings loop?

Answer: CachedFastMapFunc is a single compiled delegate that does everything (create + assign + after-map). The loop is the per-property fallback when the fast func is null.

38\.	How are MapFrom expressions handled in the fast path?

Answer: ParameterReplacer (private ExpressionVisitor in TypeMapSealer) rewrites the user's parameter to use the typed local in the parent lambda, then the body is inlined into the parent expression block via Expression.Convert if types differ.

39\.	What's the difference between Resolver and MapFrom?

Answer: Resolver (ResolveUsing) is a Func<object,object?> — runtime delegate, not inlinable beyond Expression.Invoke(Constant). MapFrom is an Expression<Func<…>> that the sealer can splice into the parent lambda for inlined code.

40\.	When does the fast path silently fall back?

Answer: TypeMapSealer.BuildFullMapFunc and BuildNestedMapFunc wrap their body in try/catch and return null on any compile error — Mapper.ResolveMapFunc then routes to the next-slower path. Risk: silent bugs.

41\.	Why a MemberInit ultra-fast path?

Answer: Equivalent to new Dest { A = s.A, B = s.B } — the cheapest C# can express. No statement body, fewest expression nodes, JIT inlines aggressively.

42\.	What happens on duplicate CreateMap<S,D>() calls in the same profile?

Answer: MapperProfile.CreateMap replaces the existing TypeMap in \_typeMaps (last-wins) — verified in MapperProfile.cs lines 17–31. Matches AutoMapper 6.x.

43\.	What about duplicate maps across profiles?

Answer: MapperConfiguration.BuildTypeMaps writes \_typeMaps\[key] = typeMap; — also last-wins. Profiles added later override earlier ones.

44\.	Where do inline cfg.CreateMap<...>() calls live?

Answer: In an InlineProfile (internal sealed), inserted at index 0 of \_profiles so explicitly added profiles win the last-wins race.

45\.	How does As<T>() interact with FindTypeMap?

Answer: Sealing populates \_overrideIndex\[(SourceType, DestinationOverrideType)] = typeMap. FindTypeMap checks \_typeMaps first, then \_overrideIndex — O(1) both.

46\.	What if both fast func and TypeConverter are set?

Answer: Can't happen — TypeMapSealer.Seal skips fast func construction when TypeConverter or TypeConverterType is non-null (lines 37–48).

47\.	What does Activator.CreateInstance in Mapper.cs line 318 cost?

Answer: \~20–60 ns per call + heap allocation for the converter. This is per Map call and matches AutoMapper 6.x behavior (comment line 33). Improvement opportunity: cache the converter type's compiled factory.

48\.	What is SourceMemberAccessVisitor used for?

Answer: Inside ConfigurationValidator.ValidateSourceMembers. Walks MapFrom expression trees, collects every PropertyInfo accessed on the source type, so the validator can mark them "covered" for MemberList.Source validation.

49\.	What is ParameterReplacer?

Answer: private sealed class ParameterReplacer : ExpressionVisitor in TypeMapSealer.cs (lines 489–496). Used to rebind the user's MapFrom lambda parameter to the typed local in the parent compiled lambda.

50\.	What is the EligibleForFastPath field for?

Answer: A static (config-time) "this map could be entirely flat" flag. Currently used as input to the fast-func compile decision; CachedFastMapFunc is the authoritative runtime check.

51\.	What's the difference between EligibleForFastPath and EligibleForNoContextPath?

Answer: Fast = all-simple props, can be inlined into one compiled lambda. NoContext = may have nested objects/collections, but proven cycle-free at config time so no ResolutionContext needed at runtime.

52\.	How is the destination created when no factory is available?

Answer: Mapper.CreateDestination (line 34) tries typeMap.CachedFactory?.Invoke(), then Activator.CreateInstance(actualType), then throws. The factory is always built unless compilation failed.

53\.	How does the existing-destination overload work?

Answer: Map<TSource,TDestination>(TSource, TDestination) routes straight to MapInternal(source, typeof(TDestination), destination). CreateDestination skips the factory and returns the provided instance.

54\.	Why doesn't the fast func support existing destinations?

Answer: CachedFastMapFunc is Func<object,object> — it always allocates and returns a fresh destination. The check at Mapper.cs line 261 (destination == null \&\& context == null) gates this.

55\.	What is the role of MarkExplicit() vs MarkConfigured()?

Answer: MarkExplicit sets both IsConfigured and IsExplicit; MarkConfigured sets only IsConfigured. ForMember sets explicit; Condition/AllowNull set configured only. Used by parity tests to mirror AutoMapper's Ignored | CustomExpression | CustomResolver semantics.

56\.	What's the role of Lazy<T> on PropertyMapping?

Answer: Defers compilation of the setter/getter/direct-copy delegates until first slow-path access. The fast path doesn't touch them. Lazy keeps cold-start cheap.

57\.	What happens when a source property is missing on the source type?

Answer: BuildSourceGetter returns null; the slow path's else continue; skips assignment; the fast path's MemberInit branch sets allGood = false and falls into the block-expression branch.

58\.	How does the library handle enums?

Answer: EnumConverter.TryConvert. First tries Enum.Parse(destType, sourceName); if that throws (name mismatch), tries Convert.ToInt32 + Enum.ToObject for numeric fallback. Only invoked in the slow path (Mapper line 472).

59\.	How does the library handle Nullable<T>?

Answer: ReflectionHelper.IsSimpleType peels Nullable.GetUnderlyingType and recurses. So int? is treated as simple. Conversion when types differ relies on Expression.Convert.

60\.	What logging is in the library?

Answer: None. No ILogger, no diagnostics output, no source generators. This is a documented gap.

C. Advanced (Q61–Q95)

61\.	Why does the cycle detection use both an identity hash and a reference equality check?

Answer: RuntimeHelpers.GetHashCode is identity-based but can still collide (32-bit space). ReferenceEquals ensures correctness on hash collisions — without it, two objects with the same identity hash but different references would be treated as the same node.

62\.	Why does the library compile Func<object, object> instead of Func<TSource, TDestination>?

Answer: One delegate type can be cached uniformly per (srcType, destType) key. Generic-typed delegates would require per-type generic instantiations and cache shape. Cost: one box on entry (for value-type sources) — but the fast-path MemberInit avoids per-property boxes.

63\.	What's wrong with MaxDepth(0) being ignored?

Answer: Nothing — it's deliberate AutoMapper 6.2.2 parity. Verified by MaxDepth0\_SelfReferencing\_NoRecursionAtAll (line 80 in Mapper\_DepthAndCircularRef\_Tests.cs) and the diagnostic probe.

64\.	What invariant does try/finally around DecrementTypeDepth enforce?

Answer: Sibling branches see the same starting depth. Without decrement, a deep left subtree would prematurely truncate the right subtree at the same MaxDepth.

65\.	What's the worst-case complexity of MapCollectionWithIdentity?

Answer: O(K²) — for each element, scan all prior elements for reference equality. Acceptable for small lists (your benchmarks: 10 items), pathological for large lists. Alternative: lazy Dictionary<object,object> with ReferenceEqualityComparer (which you use in MapNoContext line 190). Trade-off is the inverse: dict has setup cost; scan has scaling cost. The compiled collection path picked the scan to avoid the dict alloc.

66\.	What happens if Compile() succeeds but the runtime call throws?

Answer: Exception propagates. The try/catch is only around compilation, not invocation. A NullReferenceException inside a user's MapFrom lambda will bubble up to the caller of Map.

67\.	What's the locking story on \_mapFuncCache?

Answer: ConcurrentDictionary uses striped locks on write; lock-free reads. The write at line 66 (\_mapFuncCache\[key] = mapFunc;) is non-atomic w.r.t. concurrent writers — but ResolveMapFunc is idempotent (same inputs → same delegate), so a duplicate write is harmless. No GetOrAdd because the factory is non-trivial; you don't want it called twice unnecessarily — but in current code it can be, so GetOrAdd is a possible micro-improvement.

68\.	Why didn't you use GetOrAdd for the delegate cache?

Answer: With GetOrAdd, both racers' factories would still run (it's "add value factory"); only one result wins. The pattern in MapSlow is the standard "TryGet then double-check then add" pattern — which doesn't help correctness here but is slightly cheaper for the common cache-hit case (single TryGet, no factory wrapping). I could refactor to GetOrAdd for clarity at the cost of identical runtime perf.

69\.	How does ParameterReplacer work?

Answer: Extends ExpressionVisitor. Overrides VisitParameter; when it encounters the old parameter it returns the new expression, else delegates to base. Walks every node of the body and rewrites parameter references.

70\.	Why Expression.Block for the comprehensive fast func?

Answer: It supports statement sequences with locals. MemberInit cannot host conditional assignment, so anything with Condition, Resolver, or differing types must be a Block with IfThen wrappers.

71\.	What is ReferenceEqualityComparer.Instance?

Answer: A BCL singleton that uses object.ReferenceEquals and RuntimeHelpers.GetHashCode for hashing. Available in .NET 5+. Used in MapNoContext to track in-collection identity without allocating per-key state.

72\.	Why is \_typeDepths lazily allocated on ResolutionContext?

Answer: Most maps don't configure MaxDepth, so the dictionary would be a wasted \~120 B allocation. Lazy initialization (??=) keeps the common case at zero allocation cost.

73\.	Why is \_items lazily allocated?

Answer: Same reason — most calls never touch Items. The first read triggers ??= new Dictionary<string, object>().

74\.	What thread-safety guarantees does the library make?

Answer: (a) After construction, MapperConfiguration is read-only and safe to share. (b) Mapper is safe for concurrent Map calls; the only mutable state is \_mapFuncCache (ConcurrentDictionary) and \_lastMapFunc (volatile reference write — atomic on 64-bit). (c) ResolutionContext is per-call and not thread-safe — never share one across threads. (d) Caches on CollectionMapper and ReflectionHelper are process-wide ConcurrentDictionary.

75\.	What's the closure cost of Condition and Resolver?

Answer: They become heap-allocated closures captured by Expression.Constant(delegate). One allocation per delegate at config time; zero at call time. The constant is baked into the compiled lambda.

76\.	Why is \_overrideIndex needed?

Answer: As<TDerived>() overrides destination type. Without a secondary index, FindTypeMap(src, overrideType) would need an O(n) scan over \_typeMaps.Values checking DestinationOverrideType. The secondary dict makes nested lookups O(1).

77\.	Why is MapperConfiguration sealed?

Answer: Same reasons as TypeMap — sealing the perf-critical container avoids virtual call costs and prevents accidental override of FindTypeMap/CreateMapper.

78\.	How does BuildNestedMapFunc handle collections of mapped elements?

Answer: Emits Expression.Call(typeof(CollectionMapper).GetMethod("MapCollectionWithIdentity").MakeGenericMethod(srcElemType, destElemType), srcPropAccess, elemFastFunc) — a fully typed call into the helper. Wraps in IfThenElse(srcProp != null, assign mapped, assign new List<TDest>()).

79\.	What is the role of MakeGenericMethod?

Answer: Creates a closed generic method MapCollectionWithIdentity<TSrc,TDest> at seal time. The cost (reflection) is paid once; the resulting MethodInfo is baked into the compiled lambda via Expression.Call.

80\.	What happens if you call Map from inside AfterMap?

Answer: Re-enters the same IMapper. If the inner mapping is for a different type pair, the existing ResolutionContext is not propagated (the public IMapper.Map overloads don't accept one), so cycle detection is reset. This is a documented limitation — the resolver/condition/after-map signatures don't expose context.

81\.	What happens if the source type at runtime differs from the configured TSource?

Answer: Mapper uses source.GetType() not typeof(TSource) to find the map — so a derived class is mapped via its concrete type's registered map if one exists; otherwise the configured base map applies (only because the cache key uses runtime type). If no map exists, InvalidOperationException.

82\.	How would you add support for IQueryable.ProjectTo?

Answer: Would require generating an Expression<Func<TSource,TDestination>> (not just Func) so EF can translate it. The sealer already builds the expression tree — exposing a BuildProjectionExpression(TypeMap) that returns the un-Compiled tree would enable LINQ-to-Entities translation. Today this is not implemented.

83\.	How does Expression.MemberInit differ from a Block-based assignment?

Answer: MemberInit is a single expression returning the constructed object — easier for the JIT to optimize and allocates only the destination. A Block introduces local variables and statement sequencing — extra MSIL but supports IfThen and complex control flow.

84\.	Why does MapNoContext still need an identity cache for collections?

Answer: Because the same source element instance may appear multiple times in one collection. Without identity tracking, you'd return distinct destination instances for the same source — wrong for cases like "list of products where two slots reference the same product".

85\.	Why does the library not use Span<T> / Memory<T>?

Answer: The hot path is mostly about delegate dispatch and property access on heap objects, not contiguous memory. Span adds little for property copies. Could be useful inside collection enumeration if you specialized for T\[] — currently you don't.

86\.	What concurrency hazard could \_lastMapFunc introduce?

Answer: A torn read on 32-bit if MapFuncCacheEntry were a struct. It's a class (reference), so writes are atomic on all platforms. volatile prevents reordering. Worst case: two threads with different type pairs alternately overwrite — they both still hit the dictionary cache, so no correctness issue.

87\.	What's the consequence of caching Type in static dictionaries?

Answer: For long-lived processes with finite types, this is fine. For plugin scenarios that load/unload AssemblyLoadContexts, the static caches would prevent the contexts from unloading (Types pin assemblies). Mitigation: use a ConditionalWeakTable.

88\.	What happens with \[Theory] and \[InlineData]?

Answer: xUnit treats each \[InlineData] as a separate test case at runtime. Your 614 declared attributes expand to 624+ runtime tests.

89\.	Why does CompileSourceExpression mutate MemberConfiguration even though it's public?

Answer: CompiledSourceExpression is internal set — mutated only by the sealer. It's an internal cache, not a config knob. Public API is unchanged.

90\.	Why does MapperConfiguration pre-size dictionaries?

Answer: Lines 26–31: count the total maps first to size the dictionary, avoiding the resize-and-rehash sequence as maps are added. Small win; reflects perf discipline.

91\.	What is the cost of Expression.Compile()?

Answer: \~1–5 ms per delegate including IL emit + JIT, plus heap allocations for the compiled DynamicMethod. That's why the seal pass batches them and lazy-init defers them.

92\.	What's the difference between "InterpretedExpression" and "Compiled"?

Answer: Expression.Compile(preferInterpretation: true) skips JIT in favor of an interpreter (used on AOT/iOS). You compile normally — fastest steady-state, slowest cold-start. AOT would require switching.

93\.	Could you replace Lazy<T> with Volatile.Read/Write?

Answer: Yes; Lazy<T> allocates \~80 B per instance and has a thread-safety mode parameter. Volatile.Read + double-checked init is leaner but more error-prone. For \~4 lazies per property × dozens of properties, this could shave KB at config time.

94\.	What happens to ReverseTypeMap field?

Answer: It's set but never read by the runtime (you grep won't find a consumer). The reverse map's runtime behavior comes from being added to the profile's \_typeMaps list, not from the ReverseTypeMap link. The field exists for diagnostics/inspection.

95\.	What's the impact of declaring private readonly struct?

Answer: readonly prevents accidental defensive-copy elision and signals immutability. The struct goes on stack/dictionary slot; never boxed (because the dictionary is Dictionary<InstanceCacheKey, …>, not Dictionary<object, …>).

D. Expert / System-Design (Q96–Q120)

96\.	Design an object mapper from scratch — what would you do differently than AutoMapper?

Answer: Three-path execution like yours; pure expression-tree codegen; explicit "I am threadsafe after build" guarantee; first-class projection support (Expression<Func<S,D>> for IQueryable); first-class DI; benchmark-gated CI.

97\.	How would you add open-generic support (CreateMap(typeof(Wrapper<>), typeof(WrapperDto<>)))?

Answer: Add an OpenTypeMap collection. On first Map<Wrapper<int>>, scan open maps to find a match, close it with the concrete type args via MakeGenericType, run the same seal pipeline, cache. Need to evict on AssemblyLoadContext unload.

98\.	How would you support inheritance via Include?

Answer: At seal time, for CreateMap<Base, BaseDto>().Include<Derived, DerivedDto>(), ensure the derived map inherits the base's MemberConfigurations (copy-on-write merge). At runtime, runtime-type dispatch (you already use source.GetType()) means derived calls hit the derived map naturally.

99\.	How would you add IValueResolver<S,D,M>?

Answer: Add an interface taking source + destination + member name + context. The MemberConfiguration would hold a Type not a delegate; the sealer would Activator.CreateInstance or use a compiled factory. Plumbing concern: resolver may need DI — add an IServiceProvider-aware ServiceCtor.

100\.	How would you make this AOT-friendly?

Answer: Replace Expression.Compile with a Roslyn source generator that emits static partial class GeneratedMaps { public static TDest Map\_S\_to\_D(S src) { … } }. Mapper resolves to the static method via a registry. AOT-clean, zero startup compilation.

101\.	How would you publish this as a NuGet package?

Answer: <PackageId>, <Version>, <Authors>, <Description>, <PackageProjectUrl>, <RepositoryUrl>, <PackageLicenseExpression>MIT</PackageLicenseExpression>, multi-target net6.0;net8.0;netstandard2.0 (where possible), strong-named optional, deterministic build, SourceLink, SymbolPackages. Set up CI to publish on tag push.

102\.	How would you version it?

Answer: SemVer 2.0. Breaking API changes ↑major; new fluent methods ↑minor; perf/bug fixes ↑patch. The internal sealed types insulate the public API.

103\.	How would you ensure backward compatibility?

Answer: Public surface is small (MapperConfiguration, IMapper, IMapperConfigurationExpression, IMappingExpression, IMemberConfigurationExpression, IConfigurationProvider, MapperProfile, ITypeConverter, LambdaTypeConverter, MemberList). Lock these via a public API analyzer (Microsoft.CodeAnalysis.PublicApiAnalyzers).

104\.	How would you handle DI?

Answer: Add Microsoft.Extensions.DependencyInjection extensions in src/Extensions. Register MapperConfiguration as singleton, IMapper as singleton (since it's threadsafe). Optionally a ServiceCtor so resolvers can be DI-instantiated.

105\.	How would you handle observability?

Answer: Add ActivitySource("CustomAutoMapper") around MapInternal; emit Activity with tags for source/dest type and depth. Light enough to leave on in prod.

106\.	How would you handle failure modes?

Answer: Today: throws InvalidOperationException and ArgumentNullException. Add: a MappingException type with SourceType, DestinationType, MemberName, inner exception. Better diagnostics.

107\.	Could you replace Expression with System.Reflection.Emit?

Answer: Yes, would shave a small fraction off compile time and give you full IL control (e.g., emit primitive stelem.ref for arrays directly). Trade-off: much more code, harder to read, no expression rewriter. Not worth it unless you're chasing the last 5%.

108\.	How would you design the cache eviction policy if maps could be dynamically added?

Answer: Today MapperConfiguration is immutable; if you wanted dynamic maps, switch to a ConcurrentDictionary for \_typeMaps, add a version counter, and have Mapper check version on hot path (compare-and-swap \_mapFuncCache on bump). Or: copy-on-write — build a new MapperConfiguration on each change.

109\.	How would you instrument GC pressure under load?

Answer: dotnet-counters monitor System.Runtime for gen-0-gc-count; dotnet-gcdump for heap snapshots; BenchmarkDotNet's \[MemoryDiagnoser]; ETW + PerfView for allocation site stacks.

110\.	How would you design a generation-safe identity cache?

Answer: Today the cache uses Dictionary<InstanceCacheKey, object>. For cross-Gen2 retention you might use ConditionalWeakTable<object, object> so completed maps don't pin sources. Trade-off: weak references add \~40 B per entry.

111\.	What would change if the destination needed dependency-injected services?

Answer: Add a ServiceCtor callback on MapperConfiguration. Replace Activator.CreateInstance in CreateDestination and Mapper.cs line 318 with serviceCtor(actualType) ?? Activator.CreateInstance(actualType).

112\.	What would a hostile input look like?

Answer: A graph deeper than 500 → triggers GlobalMaxDepth throw — graceful. A property that throws on getter → exception propagates from the compiled delegate. A MapFrom lambda that does Environment.FailFast → process dies; no defense possible.

113\.	How does this compare to Mapster?

Answer: Mapster also uses expression trees; supports source generators in v8; has open-generic and custom-attribute support; richer DI. Your library is narrower but matches AutoMapper 6.x semantics more closely.

114\.	How does this compare to MapStruct (Java)?

Answer: MapStruct is source-generated at compile time — zero runtime reflection or expression compilation. AOT-friendly by design. Closer to where a v2 of your library could go.

115\.	What's the trade-off between expressions and source generators?

Answer: Expressions: dynamic types OK, debugger-friendly, runtime cost. SGs: AOT-friendly, zero cold start, build-time errors, fixed set of types, harder local iteration.

116\.	How would you support Tuple or record destinations with positional ctors?

Answer: Detect via type.GetConstructors() for non-default ctors; for records, RecordParameterAttribute / positional parameters. Build a BuildFactoryWithArgs that compiles Expression.New(ctor, srcProps...). Currently not supported.

117\.	What if I want to map IDictionary<string, object> to a POCO?

Answer: Currently the dictionary is treated as IEnumerable and fails. You'd need a special path that, per destination prop, calls dict.TryGetValue(propName, out var v) and assigns.

118\.	How would you support immutable destination types?

Answer: Same as records — ctor-based mapping. Compiler emits Expression.New(ctor, args) instead of New() + assignments. Requires knowing parameter→source mapping (by name or by MapFrom).

119\.	How would you add config-time error reporting that lists every problem?

Answer: Today the validator throws on first failure. Switch to "collect all" mode: accumulate List<ValidationError> and throw an AggregateException (or custom ConfigurationValidationException) at the end. Easy refactor inside ConfigurationValidator.

120\.	How would you support cancellation?

Answer: Map is synchronous and short. For batch APIs (Map<TDest\[]>(IEnumerable<TSrc>)) you'd accept CancellationToken and check it per element. Not currently exposed.

E. Behavioral / project-explanation (Q121–Q130)

121\.	Why did you build this?

Answer: To deeply understand AutoMapper internals and to learn expression-tree compilation, performance benchmarking, and parity testing — the kind of work that comes up at scale in production frameworks.

122\.	Why AutoMapper 6.2.2 specifically?

Answer: It's the last version with the classic static-API + ResolveUsing shape, before AutoMapper moved to instance-only mappers in v9+. Its semantics are stable, well-documented, and easy to pin parity against. Many enterprise codebases are still on 6.x.

123\.	What was the hardest part?

Answer: Pinning down AutoMapper 6.2.2's quirky behaviors (like MaxDepth(0) being ignored, last-wins on Ignore/UseValue) and replicating them — the diagnostic probe file exists exactly for that. Designing the three-path execution model without over-engineering was the second hardest.

124\.	What would you do differently if you started over?

Answer: (i) Drop the src segment from namespaces. (ii) Pass ResolutionContext to resolvers/conditions. (iii) Add a source-generator option from day one. (iv) Add an open-generic API. (v) Use Microsoft.Extensions.Logging for diagnostics.

125\.	What did you learn?

Answer: Expression trees as a code-generation target; how RuntimeHelpers.GetHashCode enables identity-keyed dictionaries; the cost-per-allocation calculus when you're working at 20–60 ns per call; how parity testing is much more efficient than reading source docs.

126\.	What surprised you?

Answer: That AutoMapper 6.2.2 ignores MaxDepth(0). That Expression.MemberInit is materially faster than equivalent Block codegen even when both should JIT identically. That \_lastMapFunc made a measurable difference at sub-30 ns budgets.

127\.	How did you validate correctness?

Answer: 614 declared xUnit tests, \~400 of them parity tests via ParityTestBase against AutoMapper 6.2.2. Every meaningful method has both a unit test (intent) and a parity test (compatibility).

128\.	How did you validate performance?

Answer: BenchmarkDotNet with \[MemoryDiagnoser] covering 14 scenarios. Results checked into BenchmarkDotNet.Artifacts/results. Numbers cited from the github-md report.

129\.	How big is it?

Answer: Library code (\~20 files) + benchmarks (4 files) + tests (\~25 files). The full implementation is small enough to walk through in 20 minutes.

130\.	How did you stay disciplined?

Answer: Parity-test-first for each AutoMapper feature; rejected any optimization that lacked a measurable BenchmarkDotNet delta; kept internal sealed defaults to minimize public surface.

F. Debugging / troubleshooting (Q131–Q140)

131\.	Map throws "No mapping configured" — where do you look first?

Answer: Make sure the profile was added (cfg.AddProfile(...)). Confirm the runtime source type matches the registered TSource (recall: Mapper uses source.GetType()). Check for typos in the profile registration order — last-wins.

132\.	The fast path doesn't seem to be hit — how do you verify?

Answer: Inspect typeMap.CachedFastMapFunc != null. If null: a property type is not "simple" (per IsSimpleType), or MaxDepth/TypeConverter is set, or compilation silently failed. Temporary diagnostic: comment out the catch in BuildFullMapFunc.

133\.	AfterMap runs twice in tests — why?

Answer: You likely added the same profile twice, or the inline CreateMap and an explicit AddProfile both registered the same map and both have AfterMap. Inspect typeMap.CachedAfterMapActions.Length.

134\.	Stack overflow on a circular graph despite MaxDepth?

Answer: Check whether the map fell into the fast path / no-context path. EligibleForNoContextPath should be false for cyclic graphs (the DFS detects it). If true and you still cycle, it's a missing NestedTypeMap edge — possibly because the source/destination property name mismatched and srcProps.TryGetValue returned false in PostSealResolveNestedMappings.

135\.	Mapped collection is empty when source has items — why?

Answer: Likely the destination property type isn't recognized as a collection by ReflectionHelper.IsCollectionType (e.g., a custom collection that doesn't implement IEnumerable), or the element type wasn't found via GetCollectionElementType. Add a unit test that asserts the source/dest element types.

136\.	Two threads mapping the same type — first call slow, then fast — is that expected?

Answer: Yes. First call resolves the delegate via ResolveMapFunc and writes the cache. Subsequent calls hit either \_lastMapFunc or \_mapFuncCache. Both reads are lock-free.

137\.	AssertConfigurationIsValid doesn't catch a missing nested map — why?

Answer: It doesn't traverse nested type maps. Only checks declared members. Recommend extending ConfigurationValidator with a "walk nested types" pass.

138\.	MapFrom(s => s.Inner.Value) returns 0 — why?

Answer: Probably s.Inner is null. Today the compiled source expression doesn't null-check intermediate accesses; an NRE would propagate. AutoMapper has "null-substitute" semantics; you do not.

139\.	My Condition is ignored — why?

Answer: Did you call Condition on a member without subsequently setting MapFrom/Ignore/UseValue? Condition only gates assignment, not source resolution. If the source getter resolves to null and AllowNull = true, the condition will still allow the null write.

140\.	Benchmark numbers vary between runs — how do you stabilize?

Answer: Disable CPU boost, fix CPU affinity, run in Release w/ Tiered JIT, use \[SimpleJob(launchCount: 3, warmupCount: 5, iterationCount: 10)], run in a quiet machine, prefer median over mean for noisy benchmarks.

G. Performance \& benchmarking (Q141–Q160)

141\.	What does Ratio = 0.65 mean?

Answer: Your implementation took 65% of the baseline's mean — 35% faster.

142\.	What does Alloc Ratio = 0.43 mean?

Answer: Your implementation allocated 43% as many bytes per op — 57% fewer allocations.

143\.	What does Gen0 = 0.0051 mean?

Answer: 5.1 Gen0 collections per 1000 operations — i.e., one Gen0 collection roughly every 200 ops on the configured workload.

144\.	Why is CM\_Collection slower than AM\_Collection?

Answer: AutoMapper 6.2.2 emits a fully inlined per-element copy loop; you call MapCollectionWithIdentity through a MethodInfo baked into the expression. Plus the O(i²) identity scan adds work. Improvement: inline a typed per-element loop and use a dict for K>some\_threshold.

145\.	Why is CM\_ConfigSetup slower but 38× less memory?

Answer: Two post-seal passes + fixed-point loop cost wall-clock time; but you skip building AutoMapper's full planner trees, which are huge. The trade-off is a deliberate "save allocations, spend some CPU".

146\.	What's the most expensive thing in Mapper.Map?

Answer: source.GetType() (one virtual call) + delegate invocation. With \_lastMapFunc hit, the steady-state cost is dominated by the destination allocation — which you can't avoid.

147\.	How do you measure cache effectiveness?

Answer: Add a counter on \_mapFuncCache misses; ratio of misses to calls. In your benchmark workloads (one type pair per benchmark), expect 1 miss + N hits.

148\.	How would you validate the fast path is actually used?

Answer: A targeted unit test: configure a flat map, build the mapper, reflect on TypeMap.CachedFastMapFunc and assert non-null. (Internals-visible-to your test project, which you have.)

149\.	Why are your simple-map allocations 72 B, not 48 B?

Answer: 72 B = 24 B object header + 7 props × \~variable bytes for primitives, padded to 8-byte alignment on 64-bit CLR. SimpleDestination has 2 ref props (string ×2) at 8 B each, plus int+int+decimal+bool+DateTime (4+4+16+1+8 = 33 B → \~40 B padded), so 24 + 40 + 8 = 72 ✓.

150\.	What's the impact of MapNoContext's depth > 50 fallback?

Answer: For graphs >50 deep, you incur the ResolutionContext allocation. Realistically rare — most domain graphs are <10 deep.

151\.	How would you make the cache warmup faster?

Answer: Add a Mapper.PreCompile<TSource, TDestination>() API that calls ResolveMapFunc and populates \_mapFuncCache ahead of time. Could be invoked from startup code.

152\.	What's the latency profile of one Map call on the fast path?

Answer: \~22 ns mean on Intel Core Ultra 7 268V, \~72 B allocated. Within an order of magnitude of new TDest().

153\.	What's the latency of a configuration build?

Answer: \~4.7 ms for 4 profiles in your benchmark — dominated by .Compile() calls. For 100 profiles, expect \~100 ms.

154\.	How could you reduce setup time?

Answer: Use Expression.Compile(preferInterpretation: true) for cold paths (interpreted, no JIT), only JIT-compile when the map is actually used the first time. Switch to source generators for ultimate speed.

155\.	What does \[MinColumn, MaxColumn, MedianColumn] add?

Answer: Reports min/max/median in addition to mean — easier to spot bimodal distributions caused by JIT tiering or GC pauses.

156\.	What does \[Orderer(SummaryOrderPolicy.FastestToSlowest)] do?

Answer: Sorts results in the output so the fastest method appears first — useful when you're highlighting "we're faster".

157\.	What does \[GroupBenchmarksBy(BenchmarkLogicalGroupRule.ByCategory)] do?

Answer: Groups results by \[BenchmarkCategory] attribute so AM/CM pairs appear together with the Ratio computed within group.

158\.	How do you handle benchmark noise from antivirus / background tasks?

Answer: Run with elevated priority, close Slack/Teams, disable Defender real-time scan on the bin folder. BDN's Error column should be <2% of Mean for the run to be trustworthy.

159\.	What's the difference between \[GlobalSetup] and \[IterationSetup]?

Answer: \[GlobalSetup] runs once per benchmark instance (before any iteration). \[IterationSetup] runs before every iteration — adds noise if the work is expensive, but necessary for non-idempotent benchmarks.

160\.	Why no \[Params] in your benchmarks?

Answer: Each scenario is a single configuration; you cover variety via different methods, not parameter sweeps. Improvement: add \[Params(1, 10, 100, 1000)] to Collection and Batch to characterize scaling.

H. Testing (Q161–Q170)

161\.	Why use FluentAssertions over xUnit's built-in Assert?

Answer: Better failure messages, fluent chaining, structural equivalence (BeEquivalentTo) which is critical for parity tests.

162\.	Why BeEquivalentTo not Equals?

Answer: Two distinct DTO instances with equal field values are not Equals (no override), but they are structurally equivalent. The parity tests need value comparison, not reference comparison.

163\.	What is \_DiagnosticTests/AutoMapper622\_BehaviorProbe.cs for?

Answer: To probe and pin AutoMapper 6.2.2 behaviors that you then mirror — e.g., MaxDepth(0) ignored. Acts as an executable spec.

164\.	Why is ParityTestBase an abstract class?

Answer: It provides shared helpers (AssertParity, AssertConfigurationParity, AssertReverseMapParity) usable by every \*\_ParityTests class without forcing inheritance of an unrelated state.

165\.	How would you add a property-based test?

Answer: Pull in FsCheck.Xunit. Generate random source instances; assert Map(Map(src, dst), src') ≡ Map(src', dst) for idempotence. Generate type graphs (within limits) and assert cycle detection.

166\.	What test gaps exist?

Answer: Concurrency tests (no Parallel.For over a shared mapper); fuzz tests for MapCollectionWithIdentity at K=10k; AOT smoke test (build with PublishAot=true).

167\.	How do you keep parity tests fast?

Answer: Each parity test builds two mappers and runs one map — sub-millisecond. The full suite of 600+ runs in seconds.

168\.	What's the failure mode if AutoMapper changes its behavior in a patch?

Answer: Parity tests would fail. Lock AutoMapper version (you do: Version="6.2.2").

169\.	How do you handle internal types in tests?

Answer: \[InternalsVisibleTo("CustomAutoMapper.tests.Mapping.Tests")] (or whatever your test asm is called). Without it, tests couldn't reach internal sealed types.

170\.	What's the test pyramid here?

Answer: Heavy on unit/parity (600+), zero integration, zero E2E (the library has no I/O). Add: a "real-world" integration with EF Core projections once ProjectTo is implemented.

I. .NET/C# internal (Q171–Q180)

171\.	What is the difference between Expression<T> and Func<T>?

Answer: Expression<T> is a tree of AST nodes describing the lambda; can be analyzed, rewritten, translated (LINQ-to-SQL). Func<T> is the runnable delegate. .Compile() converts the former to the latter.

172\.	What is LambdaExpression.Compile() doing under the hood?

Answer: Emits MSIL via DynamicMethod + ILGenerator, then asks the JIT to compile it to machine code on first invocation (with tiered JIT in .NET 7+: first tier 0, then tier 1).

173\.	What's the cost difference between MethodInfo.Invoke and a compiled delegate?

Answer: MethodInfo.Invoke is \~200–500 ns + boxing of args; a compiled delegate is \~3–10 ns. \~50× faster.

174\.	Why is volatile enough for \_lastMapFunc?

Answer: Reference assignments are atomic on all CLR platforms. volatile prevents reordering of the read with surrounding code. Race condition is benign: worst case, two threads briefly disagree and re-read from \_mapFuncCache.

175\.	What is ReferenceEqualityComparer.Instance?

Answer: BCL singleton in .NET 5+; IEqualityComparer<object> that uses object.ReferenceEquals for equality and RuntimeHelpers.GetHashCode for hashing.

176\.	What is RuntimeHelpers.GetHashCode?

Answer: Returns the identity hash code of an object, ignoring overrides. Used when you must hash by reference rather than logical equality.

177\.	What is Lazy<T>'s thread-safety mode?

Answer: Default is LazyThreadSafetyMode.ExecutionAndPublication — full lock around the factory. Other modes: PublicationOnly (allow concurrent runs, publish first result) or None (no thread safety).

178\.	What is MemberInit?

Answer: An expression node representing new T { Prop = value, … } — internally a NewExpression plus a list of MemberBindings. Compiles to a single object construction + field initializers.

179\.	What is Activator.CreateInstance cost?

Answer: For default ctor, \~50–100 ns + reflection cache lookup. For non-default ctors, more. A compiled Func<object> from Expression.New is \~3 ns.

180\.	What is BindingFlags.Instance | BindingFlags.Public?

Answer: Filters property reflection to instance (non-static) public members. Used consistently in your library to discover mappable properties.

J. Design trade-offs (Q181–Q200)

181\.	Why use expression trees instead of pre-built compiled functions?

Answer: Allows the library to operate on user types it has no compile-time knowledge of. SGs would require build-time integration.

182\.	Why a single Func<object,object> cache instead of typed delegates?

Answer: One cache shape regardless of types; avoids generic explosion in ConcurrentDictionary instantiations. Cost: one boxing on entry for value-type sources (which your where TSource : class constraint prevents anyway).

183\.	Why per-mapper cache instead of static?

Answer: Two mappers can have different configurations for the same (S,D) pair (different MaxDepth, different converters). Per-mapper cache prevents cross-configuration contamination.

184\.	Why a single MapperConfiguration ctor with everything?

Answer: Mirrors AutoMapper. Single immutable graph is easier to reason about than multi-step builders.

185\.	Why doesn't Resolver/Condition take a ResolutionContext?

Answer: API simplicity — Func<object,object?> and Func<object,bool> are the simplest signatures. Trade-off: users can't access Items/Depth from resolvers. Real limitation.

186\.	Why internal sealed for so many types?

Answer: Minimize public surface; reduce future versioning burden; let the JIT devirtualize. The library is intentionally narrow.

187\.	Why a base class MapperProfile instead of an interface?

Answer: Profiles store mutable \_typeMaps state and provide a CreateMap method with default value — both natural for a base class.

188\.	Why does MapperProfile.CreateMap default to MemberList.Destination?

Answer: AutoMapper parity. Safer default than None because it catches "you forgot to map a property" at validation.

189\.	Why is MemberList an enum and not flags?

Answer: The three modes are mutually exclusive; flags would invite invalid combinations (Source | Destination is not meaningful).

190\.	Why does the library not support Profile.Init pattern?

Answer: Simplicity. AutoMapper's Profile.Configure() override exists for inheritance; you require config in the derived ctor. Less flexibility, less code.

191\.	Why an IConfigurationProvider interface if there's only one implementation?

Answer: Mirrors AutoMapper API for drop-in compatibility; allows test doubles; future-proofs for an in-memory fake if needed.

192\.	Why is Mapper.cs 519 lines instead of split into smaller classes?

Answer: Map<…>, MapSlow, MapInternal, MapNoContext share a lot of nuance (depth tracking, identity cache, type converter dispatch); splitting would create artificial seams and inhibit inlining.

193\.	Why is ConfigurationValidator static?

Answer: Pure function over a TypeMap collection; no state to carry. Static = no allocation.

194\.	Why is CollectionMapper static?

Answer: Same reason; cache state is process-wide.

195\.	Why no async API?

Answer: Object mapping is CPU-bound. Wrapping in Task.Run would add scheduling overhead and obscure the synchronous nature. AutoMapper doesn't have async either.

196\.	Why LambdaTypeConverter ignores the destination parameter?

Answer: A simplification — the Func<TSource, TDestination> lambda doesn't accept a destination, so the wrapper has nothing to do with it. Real limitation: doesn't support "merge into existing destination" via ConvertUsing.

197\.	Why does CreateMap require class constraints?

Answer: Several internal paths box source-as-object. With value types you'd box on every call and lose performance. Limitation: structs need a different API.

198\.	Why is the source identifier in InstanceCacheKey a field, not a property?

Answer: Fields in a readonly struct are slightly cheaper to read than auto-properties (no getter call, even though it's inlined). Micro-perf consistent with the rest of the file.

199\.	Why is \_typeMaps a Dictionary not a ConcurrentDictionary?

Answer: Written only once during the ctor (single-threaded); read concurrently after. A plain Dictionary has no contention overhead for concurrent reads on an immutable instance.

200\.	Why Func<object,object> and not delegate void Map(object src, object dst)?

Answer: The fast path creates the destination; returning it lets the caller assign without a closure. The two-arg form would force always-pass-a-destination semantics.

\---

11\. 60-Minute Mock Interview Plan

Schedule (interview structure)

Phase	Time	Focus

1\. Project pitch	0–5 min	What it is, why you built it, headline metrics

2\. Architecture	5–20 min	Whiteboard the three execution paths; explain Seal+post-seal pipeline

3\. Code/design deep dive	20–35 min	Pick Mapper.MapInternal and TypeMapSealer.BuildFullMapFunc; explain line-by-line

4\. Performance \& benchmarks	35–45 min	Cite ratios; defend pitfalls; explain MemoryDiagnoser columns

5\. Testing \& edge cases	45–55 min	Parity vs unit; AutoMapper 6.2.2 quirks; missing edges

6\. Behavioral	55–60 min	Why this project; what you learned

Recommended interactive flow

I can run an actual mock with you — one question at a time. Tell me: "start the mock" and I'll ask Q1, wait for your answer, evaluate, give feedback, present a stronger version, then ask Q2. I recommend the 20-question sequence below, drawn from the question bank:

1\.	(Behavioral) Walk me through what this project is in 60 seconds.

2\.	(Architecture) Diagram the data flow from new MapperConfiguration(...) to mapper.Map<Dto>(entity).

3\.	(Architecture) Why three execution paths?

4\.	(Internals) Walk me through TypeMapSealer.Seal.

5\.	(Internals) Why is \_lastMapFunc volatile?

6\.	(Internals) How does cycle detection work?

7\.	(Internals) Why RuntimeHelpers.GetHashCode?

8\.	(Internals) Walk through what PostSealResolveNestedMappings does and why pass 2 is separate.

9\.	(Internals) When does CachedFastMapFunc get built? When does it not?

10\.	(Performance) Defend your Collection benchmark being slower than AutoMapper.

11\.	(Performance) Why is CM\_ConfigSetup 1.2× slower but 38× less memory?

12\.	(Performance) What's the worst-case complexity of MapCollectionWithIdentity and why did you accept it?

13\.	(Testing) Difference between parity and unit tests in this codebase?

14\.	(Testing) Three edge cases you tested. Three you didn't.

15\.	(Trade-offs) Why expression trees instead of source generators?

16\.	(Trade-offs) What would you change if you started over?

17\.	(Design) How would you add open-generic support?

18\.	(Design) How would you add AOT support?

19\.	(Defensive) An interviewer claims your numbers are biased. Defend them.

20\.	(Wrap) What's one weakness and how would you fix it?

\---

12\. Resume Defense

Bullet 1 — "Built a high-performance object-mapping library in .NET 8 replicating core features of AutoMapper 6.2.2 — ReverseMap, nested mapping, collections, converters, conditional mapping, circular reference handling."

Simple terms: I built my own AutoMapper. It supports the seven most-used features from AutoMapper 6.2.2 and behaves the same way for them.

Technical depth: The library mirrors AutoMapper 6.2.2's public surface (MapperProfile, MapperConfiguration, IMappingExpression<S,D>, IMemberConfigurationExpression<S,D>, ITypeConverter). Internally it uses compiled System.Linq.Expressions lambdas for property access, sealed types throughout, and an identity-based ResolutionContext for cycle detection. AutoMapper-specific quirks like MaxDepth(0) being ignored and last-wins on duplicate CreateMap are pinned by parity tests.

30-sec: "I built an object-mapping library in .NET 8 that drops in for AutoMapper 6.2.2 across the common scenarios: convention-based mapping, MapFrom, nested objects, collections, ReverseMap, custom converters, conditional mapping, and circular reference handling. It's about 20 source files, all behavior pinned by parity tests against the real AutoMapper."

2-min: Same as above, plus: "The public surface — MapperConfiguration, MapperProfile, CreateMap, ForMember, Ignore, MapFrom, UseValue, Condition, ResolveUsing, AfterMap, ConvertUsing, As<T>, MaxDepth, ReverseMap — matches AutoMapper 6.2.2 method-for-method. Validation supports MemberList.Destination and MemberList.Source. Cycle detection uses an identity cache keyed on RuntimeHelpers.GetHashCode + ReferenceEquals so two distinct objects never collide. Circular graphs are bounded by a GlobalMaxDepth = 500 const for DoS protection. I deliberately did not implement open-generics, Include/IncludeBase for inheritance, ProjectTo, or IValueResolver<S,D,M> — those were out of scope."

Challenges:

•	"Did you actually achieve parity?" — "For every implemented feature, yes. \~400 parity tests via ParityTestBase.AssertParity assert structural output equivalence plus configuration parity (Ignored, CustomExpression, CustomResolver, TypeConverterType, AfterMapActions). Where AutoMapper has features I don't (open generics, projection), I clearly mark them as not implemented."

•	"How is 6.2.2 still relevant?" — "It's the version with the static API and ResolveUsing shape still common in production enterprise code. Pinning to a specific version makes parity testable and stable."

Bullet 2 — "Engineered a low-latency execution pipeline using sealed TypeMaps and compiled expression-tree delegates, with ConcurrentDictionary caching and fast-path vs context-aware mapping for complex graphs."

Simple terms: I made it fast. Configuration time costs more, runtime cost is minimal. Threadsafe by design.

Technical depth:

•	TypeMap is sealed; its execution plan (CachedFactory, CachedPropertyMappings, CachedFastMapFunc, CachedAfterMapActions) is built once in TypeMapSealer.Seal and is read-only afterwards.

•	Three execution paths picked at config time and dispatched at runtime: (i) CachedFastMapFunc — single compiled lambda that does new + assign + after-map (used for flat all-simple-types maps); (ii) MapNoContext — full graph traversal but no ResolutionContext allocation, picked when static cycle analysis proves no recursion possible; (iii) MapInternal — slowest, supports MaxDepth, ITypeConverter, existing destinations, cycle detection.

•	Caching layers: MapperConfiguration.\_typeMaps (build-time), Mapper.\_mapFuncCache (ConcurrentDictionary<(Type,Type), Func<object,object>> per mapper), Mapper.\_lastMapFunc (volatile single-slot inline cache), CollectionMapper.\_listFactoryCache + \_typedAddCache + \_listFactoryWithCapacityCache (process-wide).

30-sec: "Each TypeMap is sealed at config time — we compile expression trees once and cache the delegates. At runtime, the mapper picks one of three execution paths based on what the map actually needs: a single compiled lambda for flat maps, a no-context path for cycle-free graphs, or full context-tracking for everything else. The hot loop is a volatile last-hit cache plus a ConcurrentDictionary — both lock-free for reads."

2-min: Add: "The hot path is one volatile reference read and two reference equality checks before dispatching to the cached Func<object,object>. The fast func uses Expression.MemberInit when possible — the leanest expression-tree shape, equivalent to new Dest { A = s.A, B = s.B }. For more complex flat maps with conditions or resolvers, the sealer builds a Block expression with IfThen wrappers. Nested type maps are resolved in a post-seal pass so the runtime never calls FindTypeMap. Collection support uses cached compiled list factories and typed Add delegates per element type."

Challenges:

•	"What's wrong with three paths?" — "Maintenance burden. Each feature has to be considered against three code locations. Mitigated by 600+ tests covering all three paths."

•	"Why not just one path?" — "Allocating ResolutionContext per call costs more than the rest of the work for simple maps. Three paths are the cost of pay-only-for-what-you-use."

•	"Could you prove the paths actually do what you claim?" — "Yes — TypeMap.CachedFastMapFunc != null and EligibleForNoContextPath == true can be asserted in unit tests with InternalsVisibleTo. I do this."

Bullet 3 — "Validated correctness with 624+ automated tests using xUnit — 400+ parity vs AutoMapper plus 200+ unit tests."

Simple terms: I prove it works the same as AutoMapper, and I prove my own logic is correct, with hundreds of tests.

Technical depth: 614 declared \[Fact]/\[Theory] attributes (verified via PowerShell grep). After theory expansion, 624+ runtime cases. Parity tests live in \*\_ParityTests.cs files and extend ParityTestBase. The base provides AssertParity<S,D>(autoCfg, customProfileFactory, source) which configures both libraries, runs both maps on the same input, and asserts output BeEquivalentTo plus configuration parity (Ignored/MapFrom/Resolver/TypeConverterType/AfterMapActions). Diagnostic probes in \_DiagnosticTests/AutoMapper622\_BehaviorProbe.cs lock in AutoMapper quirks.

30-sec: "Over 600 xUnit tests. About two-thirds are parity tests against AutoMapper 6.2.2 — they configure both libraries identically, map the same input, and assert structural equivalence and configuration parity. The remainder are unit tests for things only my library cares about, like the three-path dispatch logic and the MaxDepth interaction with the cycle cache."

2-min: Add: "I use FluentAssertions for BeEquivalentTo because DTOs don't override Equals. Parity tests are organized by feature: Mapper\_DepthAndCircularRef\_Tests, Mapper\_NestedMapping\_Tests, Mapper\_CollectionMapping\_Tests, MappingExpression\_ReverseMap\_ParityTests, etc. When AutoMapper and my library both throw on bad config, I normalize the exception messages to semantic buckets — duplicate\_mapping, memberlist\_violation — so the exact exception types don't have to match, only the meaning."

Challenges:

•	"How do you handle parity failures?" — "Investigate root cause; if AutoMapper is doing something undocumented, the diagnostic probe captures it and I either mirror it or document why I don't (e.g., MaxDepth(0) ignored)."

•	"600+ tests is a lot — slow?" — "All run in seconds. No I/O, no DB, just CPU. Parallel xUnit by default."

Bullet 4 — "Benchmarked using BenchmarkDotNet with MemoryDiagnoser; delivered approximately 1.2x–3.2x faster mapping and 13%–57% fewer allocations, plus 2.2x faster config setup with around 99% lower allocated memory."

Important — be honest about your own results. The github-md report shows your fastest runtime ratios at 0.63–0.78 (1.27–1.59× faster) and alloc ratios at 0.43–0.96 (4–57% fewer allocations). The 3.2× claim and 2.2× config-setup-speed claim don't match the numbers in the checked-in \*-report-github.md. I recommend updating the bullet:

Revised honest bullet:

"Benchmarked using BenchmarkDotNet with MemoryDiagnoser; delivered 1.1×–1.6× faster runtime mapping with up to 57% fewer allocations on 12 of 14 benchmarks. Config setup uses \~38× less memory at \~1.2× the build time."

Simple terms: I measured every meaningful scenario with industry-standard tools and the numbers favor my implementation on the runtime and memory dimensions that matter.

Technical depth: 14 categories in MapperBenchmarks.cs, each with an AM baseline + CM contender, decorated with \[MemoryDiagnoser] and grouped by \[BenchmarkCategory]. Hardware: Intel Core Ultra 7 268V, 8 cores, .NET 8.0.26 host. Per-scenario results checked into BenchmarkDotNet.Artifacts/results/CustomAutoMapper.Benchmarks.MapperBenchmarks-report-github.md.

Key numbers from the actual report:

•	SimpleMap: ratio 0.72 (1.39× faster), alloc ratio 0.69 (31% less)

•	MapFrom: 0.79 / 0.76

•	Nested: 0.77 / 0.78

•	Collection: 1.35 / 0.96 (slower, but slightly fewer allocations)

•	ConvertUsing: 0.65 / 0.43

•	Condition: 0.70 / 0.50

•	IgnoreConstant: 0.63 / 0.69

•	ResolveUsing: 0.89 / 0.69

•	AfterMap: 0.78 / 0.78

•	AsOverride: 0.65 / 0.50

•	LargeObject: 0.91 / 0.87

•	Batch1000: 0.73 / 0.71

•	ReverseMap: 0.69 / 0.69

•	ConfigSetup: 1.20 / 0.03 (1.2× slower, but 38× less memory)

Challenges:

•	"Your bullet says 3.2× but I see 1.59× max." — "You're right; the bullet was overstated. The honest max is 1.59× faster on ConvertUsing. I'll correct it."

•	"Your Collection is slower." — "Yes; my implementation routes through a generic helper via MethodInfo and uses an O(K²) identity scan within the compiled path. For benchmark-sized collections (10 items) the absolute delta is \~70 ns. Improvement is to inline a typed loop and switch to a dict for K above a threshold."

•	"Your config setup is 1.2× slower." — "Yes; I deliberately trade build time for runtime memory — two post-seal passes plus a fixed-point loop. The 38× memory win is what enables the runtime wins."

30-sec (revised): "I benchmarked 14 scenarios with BenchmarkDotNet + MemoryDiagnoser. On 12 of 14, I'm 1.1×–1.6× faster than AutoMapper 6.2.2 with 13–57% fewer allocations. Config setup is 1.2× slower but uses 38× less memory — that's a deliberate trade. The two losses are well-understood: Collection is slower because my codegen routes through a generic helper, and config setup is slower because I do two extra passes for cycle detection and nested fast-func construction."

2-min: Add: "Results checked into the repo at BenchmarkDotNet.Artifacts/results/\*.md. Test bench is Intel Core Ultra 7 268V, .NET 8.0.26, single in-process job. I use \[Orderer(FastestToSlowest)] and \[CategoriesColumn] to make the comparison readable. I haven't yet added \[Params] for collection-size sweeps or \[SimpleJob(launchCount: 3)] for multi-process noise reduction — those are improvements I'd add for a production claim."

\---

13\. Weaknesses and Limitations (brutally honest)

13.1 Unsupported AutoMapper features

•	Open generics (no CreateMap(typeof(X<>), typeof(Y<>))).

•	Include / IncludeBase for type hierarchies.

•	ProjectTo for IQueryable / EF Core.

•	IValueResolver<TSource, TDestination, TMember> (context-aware resolvers).

•	ConstructUsing for ctor-arg mapping.

•	DisableConstructorMapping.

•	PreserveReferences toggle (you implicitly preserve via context).

•	AllowNullCollections toggle (you always make empty).

•	Custom naming conventions (only exact-case-insensitive match).

•	Property flattening (e.g., SourceFooBar → Source.Foo.Bar).

•	Profile-level AddMaps(assembly) for type-attribute-based discovery.

•	Configuration callback after seal (no ICustomResolver-style hook).

Interviewer attack: "Why doesn't your library do X?"

Response: "Out of scope for a learning project. Each feature would be a 1–3 day add; the architecture supports it — for X I'd do Y."

13.2 Edge cases not handled / not tested

•	Value types as TSource (blocked by where TSource : class).

•	Dictionary<K,V> as destination collection.

•	Nullable<T> ↔ non-nullable with explicit conversion.

•	Source property throws on getter (would propagate unhandled).

•	Null intermediate access inside MapFrom(s => s.A.B.C) — no null-substitute support.

•	Mapping into a populated existing collection (current behavior: overwrite).

•	IReadOnlyList/ImmutableArray/HashSet as destination element types.

Improvement: Add tests; add a NullSubstitute config option per member.

13.3 Design limitations

•	Func<object,object?> resolver signature can't accept ResolutionContext. Users can't read Items from resolvers.

•	LambdaTypeConverter.Convert ignores the destination and context.

•	Namespace CustomAutoMapper.src.Core.\* includes a folder name (src) — non-idiomatic.

•	GlobalMaxDepth is a hardcoded const int 500; the commented-out configurable property in IMapperConfigurationExpression (lines 13–19) was never finished.

•	ReverseTypeMap field on TypeMap is set but never read.

•	ConfigurationValidator.ValidateMemberConfigurations is essentially a no-op — the only real checks are MemberList.

Improvement: Add a context-aware overload set; finish the GlobalMaxDepth knob; remove the dead ReverseTypeMap field.

13.4 Performance limitations

•	Collection benchmark is 1.35× slower than AutoMapper.

•	Activator.CreateInstance allocation per call for ConvertUsing<TConverter> type-based converters (mirrors AutoMapper but unnecessary if cached).

•	MapCollectionWithIdentity is O(K²) within an element loop — pathological for large lists.

•	No Span/Memory use anywhere — collection copies could shave on arrays.

•	The try/catch in BuildFullMapFunc / BuildNestedMapFunc silently masks compilation bugs.

Improvement: Cache the converter type's factory; inline a typed per-element collection loop; add a threshold-based switch from O(K²) scan to O(1) dict in the helper; log compilation failures.

13.5 API limitations

•	IMapper.Map has three overloads; no overload that accepts an external ResolutionContext for advanced scenarios.

•	MapperProfile requires a parameterless ctor (used by AddProfiles(Assembly)); no DI-aware activation.

•	No fluent WithName/Override for naming conventions.

Improvement: Add Map<…>(source, configureContext) overload; add a ServiceCtor.

13.6 Thread-safety concerns

•	Mapper is threadsafe by design — but ResolutionContext is not, and no analyzer prevents misuse.

•	The \_lastMapFunc race is benign but undocumented.

•	Static caches on CollectionMapper/ReflectionHelper pin types — problematic for plugin scenarios with AssemblyLoadContext unloading.

Improvement: XML doc the thread-safety guarantees; add ConditionalWeakTable for plugin scenarios.

13.7 Maintainability concerns

•	Mapper.MapInternal is 270 lines — large method, hard to test pieces in isolation.

•	Namespace src.Core.\* is awkward.

•	No XML doc on every public member.

•	No public-API analyzer guarding against accidental breaking changes.

Improvement: Split MapInternal into private helpers; rename namespace; add Microsoft.CodeAnalysis.PublicApiAnalyzers.

13.8 Testing gaps

•	Zero concurrency tests despite multi-threading claims.

•	Zero allocation-budget assertions (you measure but don't enforce in CI).

•	Zero AOT smoke test.

•	Zero fuzz/property-based tests.

•	Validation tests don't cover nested-type-missing scenarios.

Improvement: Add \[Fact] public async Task Mapper\_Is\_Threadsafe() with Parallel.For(0, 100\_000, \_ => mapper.Map<…>(src)); add \[Fact] public void SimpleMap\_Allocates\_Exactly\_72\_Bytes() using GC.GetAllocatedBytesForCurrentThread.

13.9 Benchmarking gaps

•	No \[Params] to characterize scaling.

•	Single launch / single iteration count — noisier than multi-launch.

•	No cold-start measurement (config + first Map).

•	No multi-threaded benchmark.

•	The bullet's "3.2× faster" and "2.2× faster config setup" don't match the checked-in numbers (0.63–0.78 / 1.20).

Improvement: Tighten the bullet to "up to 1.6×" and "38× less memory at 1.2× the time"; add multi-launch and multi-thread.

\---

14\. System Design Angle

14.1 Requirements

Functional:

•	Map any reference type A to any reference type B given a configured map.

•	Support fluent member-level rules: Ignore, MapFrom, UseValue, Condition, ResolveUsing, AllowNull.

•	Support type-level rules: ConvertUsing (lambda + class), AfterMap, As<T>, MaxDepth, ReverseMap.

•	Profile-based and inline configuration.

•	Configuration validation (MemberList.Destination / Source).

•	Map into existing destinations.

•	Nested object and collection mapping.

•	Circular-reference safety.

Non-functional:

•	Lock-free reads on the hot path.

•	Threadsafe IMapper shared across requests.

•	Predictable allocation per call (or known fallback path).

•	No code generation at build time (runtime-only).

•	Small dependency footprint (BCL only).

14.2 API design

•	Configuration ≠ Runtime: MapperConfiguration (build) → IMapper (use). Same separation as AutoMapper.

•	Fluent expressions returning this for chaining.

•	Interfaces (IConfigurationProvider, IMapper, IMapperConfigurationExpression, IMappingExpression, IMemberConfigurationExpression, ITypeConverter) exposed; impls (MapperConfiguration, Mapper, MapperConfigurationExpression, etc.) sealed.

14.3 Extensibility

•	Currently extensible only via ITypeConverter. No pluggable naming convention, no pluggable resolver factory, no profile-level hooks.

14.4 Thread safety

•	Mapper and MapperConfiguration safe to share across threads after construction.

•	ResolutionContext per-call, not threadsafe.

•	All static caches use ConcurrentDictionary.

14.5 Performance \& memory

•	Three execution paths (see §3, §5).

•	Volatile single-slot inline cache for hottest scenario.

•	Compiled delegates, capacity-aware list factories, identity hash struct keys.

14.6 Observability

•	None today. Add ActivitySource("CustomAutoMapper") around MapInternal and EventSource counters for MapsExecuted, CacheMisses.

14.7 Failure handling

•	ArgumentNullException (null source/destination).

•	InvalidOperationException (no map; depth > GlobalMaxDepth).

•	Unhandled exceptions inside user delegates propagate.

•	Silent fallback on compilation failure — risk of hidden slowness.

14.8 Versioning \& backward compatibility

•	Public API is small (\~15 types) — easy to lock with a public-API analyzer.

•	SemVer; lock AutoMapper dependency at 6.2.2 in benchmarks; library itself has no runtime deps beyond BCL.

14.9 Package publishing

•	Single CustomAutoMapper.csproj library + a separate hypothetical CustomAutoMapper.Extensions.Microsoft.DependencyInjection for DI.

•	Multi-target netstandard2.1;net8.0.

•	SourceLink, deterministic builds, debug snupkg.

•	License: choose MIT.

14.10 NuGet considerations

•	PackageId, Authors, Description, Tags=automapper;mapper;object-mapping;performance, RepositoryUrl, PackageProjectUrl, PackageLicenseExpression=MIT, PackageReadmeFile=README.md, PackageReleaseNotes.

•	Sign optional; include xml docs (<GenerateDocumentationFile>true</GenerateDocumentationFile>).

•	dotnet pack, push via CI on tag.

\---

15\. Final Interview Preparation Kit

15.1 60-second pitch

"I built CustomAutoMapper — an AutoMapper-style object mapping library in .NET 8 that mirrors the core fluent API of AutoMapper 6.2.2 (CreateMap, ForMember, MapFrom, ResolveUsing, Ignore, UseValue, Condition, AfterMap, ConvertUsing, As, MaxDepth, ReverseMap). At config time I compile expression-tree lambdas into delegates and cache them per type pair. At runtime, I dispatch to one of three execution paths depending on what each map needs: a single compiled lambda for flat types, a no-context path for cycle-free graphs, and a full context-tracking path for circular or MaxDepth scenarios. Verified by 614 xUnit tests, mostly parity against AutoMapper. BenchmarkDotNet shows 1.1×–1.6× faster runtime on 12 of 14 scenarios with up to 57% fewer allocations, and config setup uses 38× less memory."

15.2 3-minute deep project explanation

1\.	Problem (15s): Hand-written DTO mapping doesn't scale; AutoMapper is the standard but heavyweight. Wanted to learn what makes it slow/fast.

2\.	What it does (30s): Drop-in for AutoMapper 6.2.2 features I listed; same fluent surface, same last-wins semantics, same MemberList validation.

3\.	Architecture (60s): MapperConfiguration builds and seals TypeMaps; sealing compiles Func<object> factory + Action<object,object?> setters/getters + a Func<object,object> fast-map func when possible. Two post-seal passes resolve nested type maps and prove cycle-freedom for the no-context fast path. At runtime, Mapper.Map hits a volatile last-hit cache → ConcurrentDictionary → resolved delegate.

4\.	Why three paths (30s): A single path either allocates a ResolutionContext always (slow) or can't support cycles (wrong). Three paths let you pay only for the features each map needs.

5\.	Validation (30s): 614 xUnit declarations, \~400 parity tests via a ParityTestBase that runs both libraries on the same input and asserts structural + configuration equivalence.

6\.	Performance (15s): On the configured benchmarks, 1.1×–1.6× faster mapping on 12 of 14 categories, 38× less config-time memory.

15.3 5-minute architecture explanation

Walk this diagram on a whiteboard:



User code

&#x20;  │

&#x20;  ▼

new MapperConfiguration(cfg => {

&#x20;  cfg.AddProfile(new SomeProfile());

&#x20;  // SomeProfile.CreateMap<Src,Dst>().ForMember(...).ReverseMap();

})

&#x20;  │

&#x20;  ▼  ────────────  CONFIG TIME  ────────────

MapperConfigurationExpression collects MapperProfile instances

&#x20;                               │

&#x20;                               ▼

&#x20;  BuildTypeMaps  ──►  \_typeMaps: Dictionary<(Type,Type), TypeMap>

&#x20;                               │

&#x20;                               ▼

&#x20;  foreach typeMap:  typeMap.Seal()  ──►  TypeMapSealer.Seal

&#x20;                      • BuildFactory (compiled Func<object>)

&#x20;                      • BuildPropertyMappings (lazy delegates)

&#x20;                      • Try BuildFullMapFunc (compiled Func<object,object>)

&#x20;                               │

&#x20;                               ▼

&#x20;  PostSealResolveNestedMappings

&#x20;      • resolves PropertyMapping.NestedTypeMap / CollectionElementTypeMap

&#x20;      • marks EligibleForNoContextPath when HasCircularReference == false

&#x20;                               │

&#x20;                               ▼

&#x20;  PostSealBuildNestedFastMapFuncs (fixed-point loop)

&#x20;      • builds CachedFastMapFunc for nested-but-cycle-free maps

&#x20;                               │

&#x20;                               ▼

&#x20;  CreateMapper() → new Mapper(this)

&#x20;                               │

&#x20;  ─────────────  RUNTIME  ─────────────

&#x20;                               │

&#x20;  mapper.Map<TDestination>(source)

&#x20;                               │

&#x20;                               ▼

&#x20;  \_lastMapFunc (volatile single-slot) ──HIT──► invoke compiled delegate

&#x20;       │ MISS

&#x20;       ▼

&#x20;  \_mapFuncCache.TryGetValue(...) ──HIT──► invoke

&#x20;       │ MISS

&#x20;       ▼

&#x20;  ResolveMapFunc picks one of:

&#x20;       1. CachedFastMapFunc   (compiled, fastest)

&#x20;       2. typeMap.TypeConverter (lambda)

&#x20;       3. MapNoContext        (no ResolutionContext alloc)

&#x20;       4. MapInternal         (full path: MaxDepth, cycles, type converter, existing dest)

&#x20;       │

&#x20;       ▼  cached in \_mapFuncCache for next call

&#x20;  invoke → return TDestination





Talking points to hit:

•	Why sealing exists (build-once contract).

•	Why the post-seal passes are needed (cycle detection requires the full graph).

•	Why three paths exist (allocation cost is dominated by ResolutionContext).

•	Why \_lastMapFunc is volatile (lock-free hot path).

