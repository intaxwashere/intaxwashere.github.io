---
layout: single
title: "Interacting with Blueprint properties from C++."
excerpt: Interact with blueprint variables from C++ with this guide.
header:
  teaser: 
author: Meta
category: Videogames Development
tags:
  - Reflection
---

Hello people! 😊

Today I want to explain a way to interact with Blueprint properties from C++. But before starting, thanks a lot to [Dieter](https://tackytortoise.github.io/) for fixing grammar errors in this article.

If you're trying to do this, you're either:

- Someone with not enough Unreal Engine experience and trying to manipulate your Blueprint class' variables through native
- or someone who has an understanding of recommended workflows and just got curious or working on something low-level enough to require information written in here.

If you're new to Unreal Engine and trying to hack your way around to manipulate Blueprint properties, please be aware what you're trying to achieve is completely wrong and this article doesn't aim to provide what you're looking for *that* purpose. Please prefer going through this course first to learn what is the desired workflow to work with Blueprints and C++ together: [Here](https://www.unrealengine.com/en-US/onlinelearning-courses/converting-blueprints-to-c)

## Reflection

Unreal Engine has a "reflection" system to allow different systems to interact with class amd property metadatas of UObjects. We can say the design of this reflection system is probably inspired by C#.

The most base class of the reflection system is a `FField`, which defines a... field in the reflection system. A field is serializable, can provide references to GC, can be loaded from disk and contains metadata of the field it's reflecting.

What we're more interested in is `FProperty`, which is derived from `FField` and extends its functionalities to *describe* a "variable" in the reflection system. Basically for each variable you have in a UCLASS/USTRUCT, the engine will generate a FProperty behind the scenes that describes what kind of a variable it is. 

A `UStruct` (not `USTRUCT()`!) is the most base class of a *container* for `FProperty`. `UScriptStruct` (`USTRUCT()`s), `UClass` (`UCLASS()`s) and `UUserDefinedStruct`s are derived from `UStruct`. A `UStruct` contains linked list of `FProperty`'s which get linked when the `UStruct` itself get loaded. When a `UStruct` is loaded, it also loads the `FProperty`'s it's referencing/containing if they aren't already loaded. This is also the reason why having hard references causes the engine to load the referenced objects.

`FProperty` contains a few virtuals that help us a lot to interact with them:

- `InitializeValue_InContainer`: Initializes the value in the given memory block. For example, if we have a `FDoubleProperty`, we would need to provide a 4 bytes of memory block, and it would memzero it. Because the engine always initializes primitives to their zero values. But if we had a `FStructProperty`, it would go through its own metadata based on what kind of initialization it should do. For native structs, the pointer to their default constructor is being called. For blueprint structs, their members are loopd over and their own initialize functions get called recursively.
- `CopyValue_Internal`: Copies the value from *source* to *destination*. For primitives this is merely a `memcpy` by default. 
- `DestroyValue_Internal`: Destroys the value in the memory block you provided. Primitives perform a memzero, while structs call destructor.

### What kind of memory block should I provide?

The thing is, the most important part of `FProperty` to function is an integer member variable called `Offset_Internal`. We'll have to go through how pointer arithmetic works in C++ here to understand why it is very important for `FProperty` to provide it to us.

Let's say we have a struct containing three integer values:
```c
struct FThreeIntegers
{
private:
    int32 X;
    int32 Y;
    int32 Z;
} 
```
As you can see since I'm an evil person I hid them behind `private` so you wont be able access them. 

Let's construct this struct in `AEvilActor::BeginPlay`:
```c
virtual void BeginPlay() override
{
    FThreeIntegers MyStruct;
}
```
Right now, C++ created a local struct named `MyStruct`, and it *contiguously* allocated its variables in the memory. Which means, that after the memory address of where is `MyStruct` allocated, it's member variables are also allocated.

So how this "pointer arithmetic" works is: if we would get the address of `MyStruct`, we would actually get the address of `X`. Because structs and classes are merely "arguments" for the compiler on how to construct a bunch of variables and other metadata like vtable etc. in memory. If you would get the address of `X` and add `4` to it, you would get the address of `Y`. Because since integers are 4 bytes and C++ allocated our struct's memory contigously, `&MyStruct + 4` should point to the 2nd variable in our struct.

Each `FProperty`, either with some BP compiler black magic or with UHT, is able to access the relative offset of the member variables in classes and structs. This allows us to access the *current* value of the members in classes without directly using their types. 

A small cool note, since we mentioned structs and classes are merely arguments for the compiler on how to construct data contigously, actually Blueprint structs and classes are the same. They construct a bunch of variables contiguously on top of the memory their native counterparts allocated.

Basically, based on this theory we just discussed, the memory block we should provide to `FProperty` while using its utils should be the owner object *instance* of the FProperty. When we do `NewObject` or spawn an actor, it gets constructed on the heap. And since we already have to work with pointers when it comes to `UObject`s, all we need to do is to pass the living instance of the object to initialize/copy/destroy the properties. 

Don't forget, a FProperty is just a static data that ***describes*** a property in a class or a struct. It doesn't represent the memory of the variable value itself. It knows how to do operations on the memory block you provide, and its virtual functions allows it to perform different behaviors based on the `FProperty` type.

### Receiving or setting values of Blueprint properties

Since we know what of kind of a memory magic goes on behind the scenes, we can finally interact with our fancy reflection properties. 

To receive the `FProperty` itself, you either use 

- `UClass::FindProperty`
- or `TPropertyIterator`, which does exactly what FindProperty does but more useeful in case you need a different control flow over the property linked-list loop. 

After you receive the `FProperty`, get either the `UObject` or `USTRUCT()` instance, and use them like this:
```c
float CopyOfMyFloatInObject;
Property->GetValue_InContainer(MyObjectInstance, &CopyOfMyFloatInObject);

float ValueToSet = 42.42f;
Property->SetValue_InContainer(MyObjectInstnace, &ValueToSet);
```
You will see some sources or communities recommend `ContainerPtrToValue` function, but with the new 5.1 update, the functions above provide functionality to call `BlueprintSetter` and `BlueprintGetter` functions. So in case you don't need the raw value itself, prefer them over `ContainerPtrToValue`. 

Here's a usecase nevertheless:
```c
// get
float CopyOfMyFloatInObject  *(Property->ContainerPtrToValue<float>(MyObjectInstance));

// set
float ValueToSet 42.42f;
float* PtrToActualValue = (Property->ContainerPtrToValue<float>(MyObjectInstance));
*PtrToActualValue = ValueToSet;
```

That was all. Thanks for reading so far!
