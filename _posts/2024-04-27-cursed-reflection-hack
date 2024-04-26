---
layout: single
title: "How to add custom logic into engine classes without modifying source code of UE5"
excerpt: Content warning
header:
  teaser: 
author: Meta
category: Videogames Development
tags:
  - Reflection
  - Cursed
  - ContentWarning
  - cpp
---

Welcome to my another article where I explain another cursed black magic related with Unreal Engine's reflection system!

Today I will show you how to add custom logic into engine UObject types without modifying the source. Of course this comes with limitations:
- You can only override virtual functions.
- Object must have FObjectInitializer ctor implemented manually. UHT always does this for you but use NO_API, so not every class is available for this hack.
- Object must be exported.
- You cant add new members to our HackClass (see above).

```cpp
// declare your class as it is, derive from the object you want to replace its virtual function table:
class HackClass : public APawn
{
    DEFINE_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL(HackClass);

    Hack(const FObjectInitializer& FOI) : APawn(FOI) {} // you must implement FObjectInitializer ctor
    virtual void BeginPlay() 
    {
        UE_LOG(LogTemp, Display, TEXT("We injected our own virtual function table into APawn!!"));
        APawn::BeginPlay();
    }
}

struct ReplaceVtable // only needed for its ctor
{
  ReplaceVTable()
  {
    APawn::StaticClass()->DefaultConstructor() = HackClass::__DefaultConstructor; // replace reflection's ctor
  }
}
// during program startup ctor above will be called and the function that constructs UObjects in a placement new function is replaced so your vtable is replaced too:
inline static ReplaceVTable _; 
```

Dont forget, classes are merely types that humans use to define arguments to compiler to provide how a specific memory layout should be constructed. What we do here is, we're telling compiler that there is a HackClass type that is exactly same as APawn, because **it has exact same sizeof() with APawn**, *but it has a different virtual function table implementation.* This way we pass our own `__DefaultConstructor` that `DEFINE_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL` generates for us, and reflection system constructs our own version of `APawn` definition instead of the original one, and all `Pawn->BeginPlay();` calls *that arent devirtualized* (very unlikely to happen) gets routed into our own functions in the memory.

Enjoyed? No? Ikr, its *so* cursed. 😂
