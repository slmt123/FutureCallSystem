# FutureCall System (UE Plugin)

[English](https://github.com/mutou127/FutureCallSystem/blob/main/README.md)

**FutureCall** 是为 Unreal Engine 设计的异步依赖解析与执行系统（Asynchronous Dependency Resolution System）。

它通过中心化的“票据 (Ticket)”机制将**数据的提供者 (Provider)** 与**逻辑的消费者 (Consumer)** 在时间上解耦，旨在减少初始化顺序相关的问题与频繁的空指针检查。

---

## 1. 概览

FutureCall 的核心目标是：将“等待资源就绪再执行逻辑”的流程声明化、类型化并交由中枢管理，避免硬编码的初始化顺序或在 Tick 中反复检查 `IsValid()`。

Consumer 提交一张 Ticket（FutureCall），声明它依赖的若干 Provider（以 Class + Tag 标识）。当所有依赖满足时，系统会自动把对应的对象注入到回调并执行。

---

## 2. 核心特性

* **编译期类型安全**：C++ 模板元编程确保 `TFutureKey<T>` 与回调参数类型匹配，能在编译期给出明确错误提示。
* **全类型回调支持**：支持 Lambda、成员函数、Delegate、Multicast Delegate、Dynamic Delegate 以及 Dynamic Multicast Delegate。
* **GC 与生命周期安全**：追踪 Provider 与 Consumer 的弱引用。Consumer 失效则 Ticket 失败；非关键 Provider 失效时 Ticket 回退到等待队列；关键 Provider 失效时 Ticket 失败并触发全局错误事件。
* **Loop-safe Delay**：替代原生 Delay 的延时调用方案 (`DelayCall`)，自动绑定 Caller 生命周期，Caller 销毁后自动取消延时回调。
* **线程友好**：`RegisterProvider` 支持在工作线程调用，内部会 marshal 到 GameThread 安全执行。

### 2.1 系统要求

* **纯蓝图项目**：无需任何操作，即插即用。
* **C++ 项目**：
    * **UE 5.3 及以上**：引擎默认使用 C++20，**无需配置**。
    * **UE 4.26 - 5.2**：由于插件头文件包含 C++17 模板特性，您**需要**在主游戏模块的 `Build.cs` 中显式开启 C++17 支持，否则可能产生编译错误：
```csharp
// 在您游戏项目的 Build.cs 中添加：
CppStandard = CppStandardVersion.Cpp17;
```
---

## 3. 快速开始

### 3.1 注册 Provider

在对象初始化后（例如 `BeginPlay` 或 `PostInitializeComponents`）注册自己：

```cpp
UFutureCallSubsystem* Subsystem = GetGameInstance()->GetSubsystem<UFutureCallSubsystem>();

// 默认 Tag = NAME_None
Subsystem->RegisterProvider(this); 

// 指定 Tag ("Player1")
Subsystem->RegisterProvider(this, "Player1"); 
```

### 3.2 发起 FutureCall（基础用法）

**Lambda 方式**：
```cpp
UFutureCallSubsystem* FC = GetGameInstance()->GetSubsystem<UFutureCallSubsystem>();

FC->FutureCall(this, 
    // 回调函数：参数必须是 UObject* 派生类
    [](AMyHeroCharacter* Hero){
        UE_LOG(LogTemp, Log, TEXT("Hero ready: %s"), *Hero->GetName());
    }, 
    // 依赖声明：类型必须与参数匹配
    TFutureKey<AMyHeroCharacter>(NAME_None)
);
```

**成员函数方式**：
```cpp
// 声明：void OnHeroReady(AMyHeroCharacter* Hero);
FC->FutureCall(this, 
    &ThisClass::OnHeroReady, 
    TFutureKey<AMyHeroCharacter>(NAME_None)
);
```

**多依赖注入**：
```cpp
MyGameMode->FutureCall(this,
    [](AMyHeroCharacter* Hero, AMyPlayerController* PC)
    {
        Hero->SetController(PC);
    },
    // 注意：Key 的顺序必须与回调参数顺序严格一致
    TFutureKey<AMyHeroCharacter>("Player1"),
    TFutureKey<AMyPlayerController>(NAME_None)
);
```

---

## 4. 进阶用法

### 4.1 支持所有 Delegate 类型
系统支持 Unreal C++ 的各类委托，适用于需要蓝图交互或复杂绑定的场景。

* **Delegate**、**Multicast Delegate**
* **Dynamic Delegate**、**Dynamic Multicast Delegate**

```cpp
// 示例：使用Delegate
FMyDelegate MyDelegate;
MyDelegate.BindUObject(this, &ThisClass::OnCallback);

Subsystem->FutureCall(this, 
    MyDelegate, 
    TFutureKey<AProviderActor>("ServiceA")
);

// 示例：使用 Dynamic Delegate
FMyDynamicDelegate MyDynDelegate;
MyDynDelegate.BindDynamic(this, &ThisClass::OnDynamicCallback);

Subsystem->FutureCall(this, 
    MyDynDelegate, 
    TFutureKey<AProviderActor>("ServiceA")
);
```

### 4.2 关键依赖与容错
构造 `TFutureKey` 时可指定依赖是否为“关键依赖”。

* **关键依赖 (`true`)**：如果在等待期间 Provider 被销毁（Mid-flight Loss），Ticket 立即失败并触发全局报错。
* **可选依赖 (`false`, 默认)**：如果 Provider 失效，系统会容忍该错误（视具体逻辑可能挂起等待），不会视为致命错误。

```cpp
Subsystem->FutureCall(this,
    [](AProviderActor* A, AProviderActor* B) { ... },
    // 关键依赖：中途失效会报错
    TFutureKey<AProviderActor>("ServiceA", true), 
    // 非关键依赖
    TFutureKey<AProviderActor>("ServiceB", false) 
);
```

### 4.3 超集等待
允许声明额外依赖的FutureKey

* **当声明的 Key 数量多于回调参数数量时，额外的 Key 只用于判定依赖是否就绪，不会参与参数注入。**

```cpp
Subsystem->FutureCall(this,
    [](AProviderActor* A, AProviderActor* B) { ... },
    TFutureKey<AProviderActor>("ServiceA"), 
    TFutureKey<AProviderActor>("ServiceB"),
    //额外的触发条件
    TFutureKey<AProviderActor>("ServiceC")
);
```
### 4.4 多态匹配与覆盖策略

FutureCall 支持**多态依赖注入**。这意味着：如果 Ticket 等待的是父类（如 `AWeapon`），而你注册的是子类（如 `APistol`），系统会自动匹配成功。

**覆盖规则**：
为了解决“当存在多个子类实例时，谁来充当 Provider”的歧义，系统采用以下覆盖策略：
当 **Tag 相同** 时，如果 **新注册的对象 (New)** 与 **已存在的对象 (Old)** 存在**继承关系**（无论是 `New` 是 `Old` 的子类，还是 `Old` 是 `New` 的子类），**新注册的对象将覆盖旧对象**。

**高风险警告：父类覆盖子类**

假设在设计中存在以下这种情况：
1. 存在一个`APistol`，`APistol`并不是一个抽象的基类。
2. 在后续更新中添加了`APistolProMaxUltra`继承自`APistol`
3. 某个 Ticket 依赖于`APistolProMaxUltra`，在未使用合适的`Tag`进行区分的情况下，一个`APistol`也会触发这个 Ticket。
4. 该 Ticket 的回调签名使用的是`APistolProMaxUltra* Instance`，此时系统在触发回调时会尝试将一个`APistol* Intance`注入给`APistolProMaxUltra* Instance`。
5. 这会导致**类型转换失败**，进而导致回调无法执行或产生错误。
尽管系统允许双向覆盖，但**使用父类实例去覆盖已注册的子类实例**（例如用 `APistol` 覆盖 `APistolProMaxUltra`）是一项**高风险**的操作。

**潜在风险**：
由于 `UObject` 是所有对象的基类，这就带来了一个潜在风险：
1. 你先注册了一个基础 `UObject` (Tag="MyData")。
2. 随后注册了一个 `AActor` (Tag="MyData")。
3. 由于 `AActor` 是 `UObject` 的子类，系统判定两者存在继承关系，**`AActor` 将直接覆盖掉之前的 `UObject`**。
4. 此时，如果有 Ticket 专门请求之前的那个 `UObject`，它得到的将是新的 `AActor`（如果类型兼容），可能导致你的游戏逻辑出现问题或断言失败。

**最佳实践**：
* **在这种案例中 Tag 是区分实例的唯一核心**。
* 如果你不希望两个对象发生覆盖，**请务必使用不同的 Tag**。
* 仅当你确实希望“更新”或“替换”同一个逻辑槽位上的对象时（例如：玩家切换了武器，新武器替换旧武器），才使用相同的 Tag 进行注册。

在这种涉及具体类继承的场景下，请**仔细审视您的架构设计**，确认这种覆盖行为是否符合预期。

---

## 5. DelayCall

`DelayCall` 解决了标准 `UKismetSystemLibrary::Delay` 节点在循环调用场景中常见的“循环灾难的问题。

### 5.1 Lambda 延时
```cpp
Subsystem->DelayCall<AConsumerActor>(this, 2.0f, [this]()
{
    this->DoSomething(); 
});
```

### 5.2 成员函数与委托延时
```cpp
// 函数签名需为：void OnDelayFinished(AConsumerActor* Instance)
Subsystem->DelayCall<AConsumerActor>(this, 5.0f, &AConsumerActor::OnDelayFinished);

// 委托：同样支持各类 Delegate
Subsystem->DelayCall<AConsumerActor>(this, 1.0f, MyDelegate);
```

**如果 Caller 在延时结束前被销毁，回调会自动取消。**

---

## 6. 错误处理与调试

### 全局失败事件
当 Ticket 因为**关键 Provider 失效** 或 **Consumer 失效**而失败时，会触发 `OnGlobalFutureCallFailed`。

建议监听此事件以进行调试或 UI 提示。

```cpp
if (auto* Subsystem = GetGameInstance()->GetSubsystem<UFutureCallSubsystem>())
{
    Subsystem->OnGlobalFutureCallFailed.AddDynamic(this, &ThisClass::OnTicketFailed);
}

// 回调签名
void OnTicketFailed(EFailTicketReason Reason, FFutureHandle Handle, FFutureKey FailedKey)
{
    UE_LOG(LogTemp, Error, TEXT("Critical Dependency Lost: %s"), *FailedKey.Tag.ToString());
}
```

---

## 7. 常见问题 (FAQ)

* **编译错误：C2027: use of undefined type 'TError_TypeMismatch<...>'**
    * 这是系统刻意用来制造更友好编译期错误的机制。请检查 `FutureCall` 的 `TFutureKey<Type>` 与回调函数的参数类型 `Type*` 是否严格一致。

* **覆盖 Provider 的 Warning**
    * 如果多次使用相同 Class+Tag 注册 Provider，会记录 Warning，并且后注册的 Provider 会覆盖之前的实例。

---

## 8. 许可证 & 联系方式

Copyright 2025 mutou. All Rights Reserved.
