|             |                                    |
|-------------|------------------------------------|
| Feature     | GetAllAssociated		                 |
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

allow to retrieve a list a specific list of bound declaration.

---

## Motivation

### Why is this change important for Koin?

important for Koin, no idea, but for me, yes xD.

In my library project [link](https://github.com/by-tezov/tuucho), I allow the user library to add processors in many places / project modules all fed to KoinContext. Some declaration are used in different lists.

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

it works for me, but the issues is i can't resolved the linked scope because they are internal or private.

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

the second one erase or hide the first one and also it instanciate two InstanceFactory when one is only needed.


---

## Implementation Notes (optional)

### Anything relevant for contributors or maintainers to know?

I mainly talk about Library project in my request, but I think in simple application, it could also be useful

---

## Future Considerations (optional)

### How might this evolve or inspire related features?

I think it can be useful in many usecase, not sure I gave enough detail to picture the full behavior. But if it doesn't interrest Koin, would it be possible to remove the internal of some part in Koin core to allow us to add our own behavior ?

like the getAll generic, linkedScope, the rootScopeQualifier, ...


