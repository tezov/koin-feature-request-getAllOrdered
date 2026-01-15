|             |                                   |
|-------------|-----------------------------------|
| Feature     | GetAllAssociated		                |
| Submitted   | 2026-01                           |
| Status        | Draft                             |
| Issue       |                                   |
| Project Card  | https://github.com/by-tezov/tuucho |
| Project	  | [Koin]			                         |
| Component     | core                              |
| Version	  | 4.2.0+			                         |

---

## Summary

### What is this proposal about? What problem does it solve?

allow to retrieve a list of specific items of bound declaration. It allows to retrieve different list when the interface of class is the same.

example:

```

interface Marker
class A: Marker
class B: Marker
class C: Marker

if I want a list with
[A, B, C] -> getAll<Marker>() is enough

But if I want different list
[A,C]
[B,C] -> it is not possible. I would need to add as many interfaces on each class and it become exponential

So what I did it to allow to bound (associate) to anything and the class don't need to implement it.

So it becomes :

interface Marker1
interface Marker2

interface RealInterface
class A : RealInterface
class B : RealInterface
class C : RealInterface

single / factory A associate to Marker1
single / factory A associate to Marker2

single / factory B associate to Marker1

single / factory C associate to Marker2

getAllAssociated<Marker1> return a list of RealInterface with A and B
getAllAssociated<Marker2> return a list of RealInterface with A and C

```

And it allow to add any class to any list associated anywhere cross koin modules, cross project module

---

## Motivation

### Why is this change important for Koin?

In my library project [link](https://github.com/by-tezov/tuucho), I allow the user library to add processors in many places / project modules all fed to Koin registry. Some declarations are used in different lists.

Right now, there are only getAll(), we can't use qualifier.

---

## Proposed Solution

###  Explain the approach simply. Add a short example if helpful.

Example of use :

```
module {
    factoryOf(::ContentsRectifier)
    factoryOf(::SomeRectifierUseInDifferentList)

    // when several need to be added at once 
    associate<MaterialRectifier.Association.Processor> {
        factoryOf(::ComponentsRectifier)
        declaration<ContentsRectifier>() // declaration allow the reuse of existing InstanceFactory
        declaration<SomeRectifierUseInDifferentList>() 
    }

    // when single need to be added
    factoryOf(::IdMatcher) associate IdRectifier.Association.Matcher::class

    // same a added with reuse of InstanceFactory
    factoryOf(::StateRectifier)
    declaration<StateRectifier>() associate StateRectifier.Association.Processor::class
    declaration<SomeRectifierUseInDifferentList>() associate StateRectifier.Association.Processor::class

}
-> Here I use only one module, but it can be done across koin modules
```

And to use it

```
    koin.getAllAssociated(MaterialRectifier.Association.Processor::class)
    //this return list of ComponentsRectifier, ContentsRectifier and SomeRectifierUseInDifferentList
    
    koin.getAllAssociated(IdRectifier.Association.Matcher::class)
    // this return list of IdMatcher
    
    koin.getAllAssociated(StateRectifier.Association.Processor::class)
    // this return list of StateRectifier and SomeRectifierUseInDifferentList

```

full example can be seen in tuucho library core data module and sample application where the library user can add his own processor [link](https://github.com/by-tezov/tuucho)

And I implemented it like this :

```
@KoinDslMarker
infix fun <T : Any> InstanceFactory<T>.associate(
    clazz: KClass<*>
) {
    beanDefinition.secondaryTypes += clazz
}

@OptIn(KoinInternalApi::class)
@KoinDslMarker
infix fun <S : Any> KoinDefinition<out S>.associate(
    clazz: KClass<*>
): KoinDefinition<out S> {
    factory.associate(clazz)
    val mapping =
        indexKey(clazz, factory.beanDefinition.qualifier, factory.beanDefinition.scopeQualifier)
    module.mappings[mapping] = factory
    return this
}

@OptIn(KoinInternalApi::class)
@KoinDslMarker
inline fun <reified T : Any> Module.declaration(
    qualifier: Qualifier? = null
): InstanceFactory<T> {
    val mapping = indexKey(T::class, qualifier, Constant.koinRootScopeQualifier)
    @Suppress("UNCHECKED_CAST")
    return (mappings[mapping] as? InstanceFactory<T>)
        ?: throw DomainException.Default("${T::class.getFullName()} not found in module")
}

@OptIn(KoinInternalApi::class)
@KoinDslMarker
inline fun <reified T : Any> ScopeDSL.declaration(
    qualifier: Qualifier? = null
): InstanceFactory<T> {
    val mapping = indexKey(T::class, qualifier, scopeQualifier)
    @Suppress("UNCHECKED_CAST")
    return (module.mappings[mapping] as? InstanceFactory<T>)
        ?: throw DomainException.Default("${T::class.getFullName()} not found in scope")
}

@OptIn(KoinInternalApi::class)
@KoinDslMarker
inline fun <reified T : Any> Koin.getAllAssociated(
    clazz: KClass<*>
): List<T> {
    val instanceContext = ResolutionContext(logger, scopeRegistry.rootScope, clazz)
    instanceContext.scopeArchetype = scopeRegistry.rootScope.scopeArchetype
    return instanceRegistry.instances.values
        .filter { factory ->
            (factory.beanDefinition.scopeQualifier == instanceContext.scope.scopeQualifier ||
                factory.beanDefinition.scopeQualifier == instanceContext.scope.scopeArchetype
            ) &&
                (factory.beanDefinition.primaryType == clazz || factory.beanDefinition.secondaryTypes.contains(clazz))
        }.distinct()
        .sortedWith(compareBy { it.beanDefinition.toString() })
        .mapNotNull { it.get(instanceContext) as? T } // TODO linked scope, can't do because it is internal
}

@OptIn(KoinInternalApi::class)
@KoinDslMarker
inline fun <reified T : Any> Scope.getAllAssociated(
    clazz: KClass<*>
): List<T> = with(getKoin()) {
    val instanceContext = ResolutionContext(logger, this@getAllAssociated, clazz)
    instanceContext.scopeArchetype = this@getAllAssociated.scopeArchetype
    instanceRegistry.instances.values
        .filter { factory ->
            // TODO linked scope
            (factory.beanDefinition.scopeQualifier == instanceContext.scope.scopeQualifier ||
                factory.beanDefinition.scopeQualifier == instanceContext.scope.scopeArchetype
            ) &&
                (factory.beanDefinition.primaryType == clazz || factory.beanDefinition.secondaryTypes.contains(clazz))
        }.distinct()
        .sortedWith(compareBy { it.beanDefinition.toString() })
        .mapNotNull { it.get(instanceContext) as? T }
}

@KoinDslMarker
inline fun <reified T : Any> Module.associate(
    associateDSL: AssociateModule.() -> Unit
) {
    AssociateModule(T::class, this).associateDSL()
}

@KoinDslMarker
inline fun <reified T : Any> ScopeDSL.associate(
    associateDSL: AssociateScopeDSL.() -> Unit
) {
    AssociateScopeDSL(T::class, this).associateDSL()
}
```

it works for me, but the issues is:
- I can't resolved the linked scope because they are internal or private.
- I need to duplicate Koin core code since direct function are internal

code here [link](https://github.com/by-tezov/tuucho/blob/master/tuucho/core-modules/domain/business/src/commonMain/kotlin/com/tezov/tuucho/core/domain/business/_system/koin/AssociateDSL.kt)

unit test here [link](https://github.com/by-tezov/tuucho/blob/master/tuucho/core-modules/domain/business/src/commonTest/kotlin/com/tezov/tuucho/core/domain/business/_system/koin/AssociateDSLTest.kt)

---

## Drawbacks & Alternatives

### What are the trade-offs? Were other options considered?


I first tried with the qualifier, but 

```
factory(named("mylist1")) { A() } 
factory(named("mylist2")) { A() }
```

the second one erase (or hide the first one) and also it instantiate two InstanceFactory when one is only needed.


---

## Implementation Notes (optional)

### Anything relevant for contributors or maintainers to know?

I mainly talk about Library project in my request, but I think in simple application, it could also be useful.

---

## Future Considerations (optional)

### How might this evolve or inspire related features?

I think it can be useful in many use cases, not sure if I gave enough detail to picture the full behavior. If it doesn't interrest Koin core, would it be possible to remove the internal of some part in Koin core to allow us to add our own behavior ? Maybe by marking them DelicateApi ?

like the getAll generic, linkedScope, the rootScopeQualifier, ...


