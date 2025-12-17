# FutureCall System (UE Plugin)

[Chinese](https://github.com/slmt123/FutureCallSystem/blob/main/README_CN.md)

**FutureCall** is an Asynchronous Dependency Resolution System designed for Unreal Engine.

It temporally decouples **Data Providers** from **Logic Consumers** through a centralized "Ticket" mechanism, aiming to reduce issues related to initialization order and frequent null pointer checks.

---

## 1. Overview

The core goal of FutureCall is to make the process of "waiting for resources to be ready before executing logic" declarative, typed, and centrally managed, avoiding hard-coded initialization orders or repeatedly checking `IsValid()` in `Tick`.

A Consumer submits a Ticket (FutureCall), declaring several Providers it depends on (identified by Class + Tag). When all dependencies are satisfied, the system automatically injects the corresponding objects into the callback and executes it.

---

## 2. Core Features

* **Compile-time Type Safety**: C++ template metaprogramming ensures that `TFutureKey<T>` matches the callback parameter types, providing clear error prompts at compile time.
* **Full Callback Support**: Supports Lambdas, Member Functions, Delegates, Multicast Delegates, Dynamic Delegates, and Dynamic Multicast Delegates.
* **GC & Lifecycle Safety**: Tracks weak references for Providers and Consumers. If a Consumer becomes invalid, the Ticket fails. If a non-critical Provider becomes invalid, the Ticket returns to the waiting queue. If a critical Provider becomes invalid, the Ticket fails and triggers a global error event.
* **Loop-safe Delay**: A delay call solution (`DelayCall`) that replaces native Delay. It automatically binds to the Caller's lifecycle—if the Caller is destroyed, the delayed callback is automatically cancelled.
* **Thread Friendly**: `RegisterProvider` supports being called from worker threads; the system internally marshals it to the GameThread for safe execution.

---

## 3. Quick Start

### 3.1 Register Provider

Register yourself after the object is initialized (e.g., in `BeginPlay` or `PostInitializeComponents`):

```cpp
UFutureCallSubsystem* Subsystem = GetGameInstance()->GetSubsystem<UFutureCallSubsystem>();

// Default Tag = NAME_None
Subsystem->RegisterProvider(this); 

// Specify Tag ("Player1")
Subsystem->RegisterProvider(this, "Player1"); 
```

### 3.2 Make a FutureCall (Basic Usage)

**Lambda Style**:
```cpp
UFutureCallSubsystem* FC = GetGameInstance()->GetSubsystem<UFutureCallSubsystem>();

FC->FutureCall(this, 
    // Callback: Parameters must be UObject derived classes
    [](AMyHeroCharacter* Hero){
        UE_LOG(LogTemp, Log, TEXT("Hero ready: %s"), *Hero->GetName());
    }, 
    // Dependency declaration: Type must match the parameter
    TFutureKey<AMyHeroCharacter>(NAME_None)
);
```

**Member Function Style**:
```cpp
// Declaration: void OnHeroReady(AMyHeroCharacter* Hero);
FC->FutureCall(this, 
    &ThisClass::OnHeroReady, 
    TFutureKey<AMyHeroCharacter>(NAME_None)
);
```

**Multi-Dependency Injection**:
```cpp
MyGameMode->FutureCall(this,
    [](AMyHeroCharacter* Hero, AMyPlayerController* PC)
    {
        Hero->SetController(PC);
    },
    // Note: The order of Keys must strictly match the callback parameter order
    TFutureKey<AMyHeroCharacter>("Player1"),
    TFutureKey<AMyPlayerController>(NAME_None)
);
```

---

## 4. Advanced Usage

### 4.1 Support for All Delegate Types
The system supports various Unreal C++ delegates, suitable for scenarios requiring Blueprint interaction or complex bindings.

* **Delegate**, **Multicast Delegate**
* **Dynamic Delegate**, **Dynamic Multicast Delegate**

```cpp
// Example: Using Delegate
FMyDelegate MyDelegate;
MyDelegate.BindUObject(this, &ThisClass::OnCallback);

Subsystem->FutureCall(this, 
    MyDelegate, 
    TFutureKey<AProviderActor>("ServiceA")
);

// Example: Using Dynamic Delegate
FMyDynamicDelegate MyDynDelegate;
MyDynDelegate.BindDynamic(this, &ThisClass::OnDynamicCallback);

Subsystem->FutureCall(this, 
    MyDynDelegate, 
    TFutureKey<AProviderActor>("ServiceA")
);
```

### 4.2 Critical Dependencies & Fault Tolerance
When constructing a `TFutureKey`, you can specify whether the dependency is "Critical".

* **Critical Dependency (`true`)**: If the Provider is destroyed while waiting (Mid-flight Loss), the Ticket fails immediately and triggers a global error.
* **Optional Dependency (`false`, Default)**: If the Provider becomes invalid, the system tolerates the error (depending on logic, it may hang/wait), and is not treated as a fatal error.

```cpp
Subsystem->FutureCall(this,
    [](AProviderActor* A, AProviderActor* B) { ... },
    // Critical dependency: Error if lost mid-flight
    TFutureKey<AProviderActor>("ServiceA", true), 
    // Non-critical dependency
    TFutureKey<AProviderActor>("ServiceB", false) 
);
```

### 4.3 Superset Waiting
Allows declaring extra dependency FutureKeys.

* **When the number of declared Keys exceeds the number of callback parameters, the extra Keys are only used to determine if dependencies are ready and do not participate in parameter injection.**

```cpp
Subsystem->FutureCall(this,
    [](AProviderActor* A, AProviderActor* B) { ... },
    TFutureKey<AProviderActor>("ServiceA"), 
    TFutureKey<AProviderActor>("ServiceB"),
    // Extra trigger condition
    TFutureKey<AProviderActor>("ServiceC")
);
```

---

## 5. DelayCall

`DelayCall` solves the "Loop Disaster" problem common with the standard `UKismetSystemLibrary::Delay` node in loop calling scenarios.

### 5.1 Lambda Delay
```cpp
Subsystem->DelayCall<AConsumerActor>(this, 2.0f, [this]()
{
    this->DoSomething(); 
});
```

### 5.2 Member Function & Delegate Delay
```cpp
// Function signature must be: void OnDelayFinished(AConsumerActor* Instance)
Subsystem->DelayCall<AConsumerActor>(this, 5.0f, &AConsumerActor::OnDelayFinished);

// Delegate: Also supports various Delegates
Subsystem->DelayCall<AConsumerActor>(this, 1.0f, MyDelegate);
```

**If the Caller is destroyed before the delay ends, the callback is automatically cancelled.**

---

## 6. Error Handling & Debugging

### Global Failure Event
When a Ticket fails due to **Critical Provider Loss** or **Consumer Loss**, `OnGlobalFutureCallFailed` is triggered.

It is recommended to listen to this event for debugging or UI notifications.

```cpp
if (auto* Subsystem = GetGameInstance()->GetSubsystem<UFutureCallSubsystem>())
{
    Subsystem->OnGlobalFutureCallFailed.AddDynamic(this, &ThisClass::OnTicketFailed);
}

// Callback Signature
void OnTicketFailed(EFailTicketReason Reason, FFutureHandle Handle, FFutureKey FailedKey)
{
    UE_LOG(LogTemp, Error, TEXT("Critical Dependency Lost: %s"), *FailedKey.Tag.ToString());
}
```

---

## 7. FAQ

* **Compile Error: C2027: use of undefined type 'TError_TypeMismatch<...>'**
    * This is a mechanism deliberately used by the system to produce friendlier compile-time errors. Please check if the `TFutureKey<Type>` in `FutureCall` strictly matches the callback function parameter type `Type*`.

* **Warning about Overwriting Provider**
    * If you register a Provider with the same Class+Tag multiple times, a warning will be logged, and the later registered Provider will overwrite the previous instance.

---

## 8. License & Contact

Copyright 2025 mutou. All Rights Reserved.
