|             |                                    |
|-------------|------------------------------------|
| Feature     | GetAllOrdered		                    |
| Submitted   | 2026-01                            |
| Status        | Draft                              |
| Issue       |                                    |
| Project Card  | https://github.com/by-tezov/tuucho |
| Project	  | [Koin]			                          |
| Component     | core                               |
| Version	  | 4.2.0+			                          |

---

## Summary

### What is this proposal about? What problem does it solve?

allow to retrieve a list of bound declaration in the exact same order they were declared. the actual getAll does not keep order.

---

## Motivation

### Why is this change important for Koin?

In my library project [link](https://github.com/by-tezov/tuucho), I allow the user library to create middlewares which is injected in the Isolated Koin context to have control on the library behavior (Network, navigation, other stuff)

Example:

My library have navigation stack, the user can wrap the core navigation inside their own middlewares to guard auth section and/or redirect on the fly, load configuration, catch errors etc.

To keep the code clean, we declare different middlewares

```
CatchErrorMiddleware [
    BeforeNavigateMiddleware [
        // do some stuff
        CoreNavigationMiddleware [
            // core library code
            AfterLoggerMiddleware []
        ]
    ]
   // catch contains all middlewares and is sure to react on unmanaged errors   
]
```

this can work as expected only if the order of middlewares is preserved (same concept as nestJS, or retrofit interceptor).

---

## Proposed Solution

###  Explain the approach simply. Add a short example if helpful.

Example of use : 

```
module {
    factoryOf(::CatcherBeforeNavigateToUrlMiddleware) bindOrdered NavigationMiddleware.ToUrl::class
    
    factory<BeforeNavigateToUrlMiddleware> {
        BeforeNavigateToUrlMiddleware()
    } bindOrdered NavigationMiddleware.ToUrl::class
    
    factoryOf(::LoggerBeforeNavigateToUrlMiddleware) bindOrdered NavigationMiddleware.ToUrl::class
}
-> module is fed to koin

and somewhere else in the Library

factory {
    NavigateToUrlUseCase(
        coroutineScopes = get(),
        useCaseExecutor = get(),
        ...
        middlewareExecutor = get(),
        navigationMiddlewares = getAllOrdered() <- here the retrieval happens
    )
}
```
full example can be seen in sample application shared module [link](https://github.com/by-tezov/tuucho)


And I implemented it like this :

```
@OptIn(KoinInternalApi::class)
@KoinDslMarker
infix fun <S : Any> KoinDefinition<out S>.bindOrdered(
    clazz: KClass<S>
): KoinDefinition<out S> {
    val index = module.mappings.values
        .distinctBy { it.beanDefinition }
        .count { it.beanDefinition.secondaryTypes.contains(clazz) }
    bind(clazz)
    val typeName = clazz.qualifiedName ?: error("class qualified name is null")
    val orderedQualifier = named("$typeName#ordered#$index")
    val mapping = indexKey(clazz, orderedQualifier, factory.beanDefinition.scopeQualifier)
    module.mappings[mapping] = factory
    return this
}

@OptIn(KoinInternalApi::class)
@KoinDslMarker
inline fun <reified T : Any> Koin.getAllOrdered(): List<T> = scopeRegistry.rootScope.getAllOrdered(T::class)

@KoinDslMarker
inline fun <reified T : Any> Scope.getAllOrdered(): List<T> = getAllOrdered(T::class)

@OptIn(KoinInternalApi::class)
@KoinDslMarker
fun <T : Any> Scope.getAllOrdered(
    clazz: KClass<T>
): List<T> = with(getKoin()) {
    val typeName = clazz.qualifiedName ?: error("class qualified name is null")
    val instanceContext = ResolutionContext(logger, scopeRegistry.rootScope, clazz)
    instanceContext.scopeArchetype = scopeRegistry.rootScope.scopeArchetype
    instanceRegistry.instances.entries
        .filter { (key, factory) ->
            key.contains("$typeName#ordered#") &&
                (
                    (factory.beanDefinition.scopeQualifier == instanceContext.scope.scopeQualifier ||
                        factory.beanDefinition.scopeQualifier == instanceContext.scope.scopeArchetype
                    ) &&
                        (factory.beanDefinition.primaryType == clazz || factory.beanDefinition.secondaryTypes.contains(clazz))
                )
        }.distinct()
        .mapNotNull { (key, factory) ->
            val index = key
                .substringAfter("#ordered#", "")
                .substringBefore(':')
                .toIntOrNull() ?: return@mapNotNull null
            index to factory
        }.sortedBy { it.first }
        .mapNotNull {
            @Suppress("UNCHECKED_CAST")
            it.second.get(instanceContext) as? T
        } // TODO linked scope, can't do because it is internal
}
```

it works for me, but the getAllOrdered is ugly and use too much KoinInternalApi.

- code here [link](https://github.com/by-tezov/tuucho/blob/master/tuucho/core-modules/domain/business/src/commonMain/kotlin/com/tezov/tuucho/core/domain/business/_system/koin/BindOrdered.kt)
- unit test here [link](https://github.com/by-tezov/tuucho/blob/master/tuucho/core-modules/domain/business/src/commonTest/kotlin/com/tezov/tuucho/core/domain/business/_system/koin/BindOrderedTest.kt)

---

## Drawbacks & Alternatives

### What are the trade-offs? Were other options considered?

the way I did it works only if bindOrdered are done on the same module. Cross modules overwrite previous module that bind the same interface (can be seen in unit tests), not an issue in my context since library user can use a moduleContext (that's another topic, I will open another KFIP for that)

---

## Implementation Notes (optional)

### Anything relevant for contributors or maintainers to know?

I mainly talk about Library project in my request, but I think in simple application, it could also be useful

---

## Future Considerations (optional)

### How might this evolve or inspire related features?
