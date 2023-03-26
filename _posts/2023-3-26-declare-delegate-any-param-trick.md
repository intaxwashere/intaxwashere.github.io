---
layout: single
title: "Declaring non-dynamic delegates with N (any) amount of parameters."
excerpt: No more _OneParam or _TwoParams etc. bullshit
header:
  teaser: 
author: Meta
category: Videogames Development
tags:
  - cpp
---

Since dynamic macros are part of reflection system, they rely on different mechanism, but there is no actual reason why we use `_OneParam`, `_TwoParams` etc. on non-dynamic delegates.

If you don't know what are delegates or not sure how to use them properly, I can recommend having a further read from [benui's awesome article here.](https://benui.ca/unreal/delegates-intro/)

These macros below allow you to declare delegate types without manually typing param count for them:

```c
#define DECLARE_DELEGATE_AnyParam(DelegateName, ...) \
    using DelegateName = TDelegate<void(__VA_ARGS__)>;
#define DECLARE_MULTICAST_DELEGATE_AnyParam(DelegateName, ...) \
    using DelegateName = TMulticastDelegate<void(__VA_ARGS__)>

// delegate declarations with return types:
#define DECLARE_DELEGATE_AnyParam_RetVal(DelegateName, RetVal, ...) \
using DelegateName = TDelegate<RetVal(__VA_ARGS__)>;
#define DECLARE_MULTICAST_DELEGATE_AnyParam_RetVal(DelegateName, RetVal, ...) \
using DelegateName = TMulticastDelegate<RetVal(__VA_ARGS__)>
```

Example usage:

```c
// Declare a delegate named FSomeDelegate with given parameters:
DECLARE_DELEGATE_AnyParam(FSomeDelegate, float, int, UObject*, TArray<float>&);
// Declare a multicast delegate named FSomeMulticastDelegate with given parameters:
DECLARE_MULTICAST_DELEGATE_AnyParam(FSomeMulticastDelegate, float, int, UObject*);
void Test()
{
    // execute FSomeDelegate:
    FSomeDelegate Test;
    TArray Array{123.f, 123.f};
    Test.Execute(123.f, 12, nullptr, Array);

    // execute FSomeMulticastDelegate:
    FSomeMulticastDelegate MultiTest;
    MultiTest.Broadcast(123.f, 123, nullptr);
}
```

If you're not a fan of delegates, you can also prefer the given way to declare non-multicast delegates:
```c
using YourDelegateTypename = TDelegate<void(int, float, double, etc, anyothertype)>;
using YourDelegateTypenameWithReturnType = TDelegate<int(int, float, double, etc, anyothertype)>;
```
Here is an example:
```c
// if we have a function signature like this:
virtual int32 AnimeGirlsAreReal(const float X, double& Y, const TArray<FVector>& Z);
// this delegate would be compatible with the function above:
using AnimeGirlsAreRealDelegate = TDelegate<int32(const float, double&, const TArray<FVector>&)>
```
