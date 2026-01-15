|              |                                    |
|--------------|------------------------------------|
| Feature      | Scattered Scope		                  |
| Submitted    | 2026-01-01                         |
| Status       | Draft                              |
| Issue        |                                    |
| Project Card | https://github.com/by-tezov/tuucho |
| Project	     | [Koin]			                          |
| Component    | core                               |
| Version	     | 4.2.0+			                          |

---

## Summary

### What is this proposal about? What problem does it solve?

In Library context (I don't think it would be useful in Application context). The goal is to allow
Library side or application side to add item inside scope declared inside the library.

Right now, when we use scope,

```
Somewhere
module {
    
    scope<>() {
        ... some declaration.    
    
    }
}

Somewhere else (another Library project module, or in the from the user application)

-> not possible to add more declaration to the scope.
```

---

## Motivation

### Why is this change important for Koin or Koin Annotations?

I really don't know how it would be useful to Koin. This one is not really a feature request needed,
I was able to do it without fighting with InternalApi. It is a share of what I did to let you see it
and decide if it can be useful for you.

---

## Proposed Solution

### Explain the approach simply. Add a short example if helpful.

First I declared what I called a KoinMass. Each project modules will return a list of KoinMass. Then
this list will be added in KoinConfiguration

A KoinMass can be a ModuleContext or a ScopeContext.

```
typealias ModuleDeclaration = Module.() -> Unit
typealias ScopeDeclaration = ScopeDSL.() -> Unit

sealed class KoinMass {
    interface ModuleContext

    interface ScopeContext : Qualifier {
        val group: ModuleContext
        override val value: QualifierValue
            get() = TypeQualifier(this::class).value
    }

    abstract val group: ModuleContext

    data class Module(
        override val group: ModuleContext,
        val declaration: ModuleDeclaration
    ) : KoinMass()

    data class Scope(
        val scopeContext: ScopeContext,
        val declaration: ScopeDeclaration
    ) : KoinMass() {
        override val group get() = scopeContext.group
    }

    companion object {
        @KoinDslMarker
        fun module(
            group: ModuleContext,
            declaration: ModuleDeclaration
        ) = Module(group, declaration)

        @KoinDslMarker
        fun scope(
            scopeContext: ScopeContext,
            declaration: ScopeDeclaration
        ) = Scope(scopeContext, declaration)
    }
}
```

Example:

In one project module, I declare one Module 'Rectifier' with scope 'Material' (In really, I have several scope but to keep this explanation simple, I will use only one scope)
```
object Rectifier : KoinMass.ModuleContext {
    object MaterialScope : KoinMass.ScopeContext {
        override val group get() = Rectifier
    }
}

```

From Library :

```
val coreModule = module(Rectifier) {
    singleOf(::MaterialRectifier)
}

val coreModuleScope = scope(Rectifier.MaterialScope) {
    scoped(::IdRectifier)
    factoryOf(::ComponentsRectifier)
}
```
Some other module project in the Library (or user application)

```
val featModuleScope = scope(Rectifier.MaterialScope) {
    scoped(::SettingComponentRectifier)
    factoryOf(::ColorRectifier)
}
```

And finally, where koin is started :

```
koinMasses list supply somehow = listof(coreModule, coreModuleScope, featModuleScope)

koinApplication {
    allowOverride(override = false)
    modules(koinMasses.groupBy { it.group }.map { (_, groups) ->
        
        val (modules, scopes) = groups.partition { it is KoinMass.Module }
             
        module {
            @Suppress("UNCHECKED_CAST")
            (modules as List<KoinMass.Module>).forEach { module ->
                module.declaration(this)
            }
            
            @Suppress("UNCHECKED_CAST")
            (scopes as List<KoinMass.Scope>)
                .groupBy { it.scopeContext }
                .forEach { (scopeContext, koinScopes) ->
                    scope(scopeContext) {
                        koinScopes.forEach { it.declaration(this) }
                    }
                }
        }
    })
}
```

Advantage, it allow to add stuff in scope from any project modules.

---

## Drawbacks & Alternatives

### What are the trade-offs? Were other options considered?

I guess the drawback would be the learning curve ? And at company, we never used scope manually, so I think it would be only for niche application and adventurous dev...

---

## Implementation Notes (optional)

### Anything relevant for contributors or maintainers to know?

You can see a real use case in my project [link](https://github.com/by-tezov/tuucho). You need to look inside each di folder of each module in the Library (inside data layer for the scope part).

---

## Future Considerations (optional)

### How might this evolve or inspire related features?
