# JAVA20新特性

JAVA20中新增的特性全部是预览和孵化功能

## Amber项目

### switch 模式匹配 (Pattern Matching for switch) 
进入第 4 预览阶段
用 switch 表达式和语句的模式匹配，以及对模式语言的扩展来增强 Java 编程语言。将模式匹配扩展到 switch 中，允许针对一些模式测试表达式，这样就可以简明而安全地表达复杂的面向数据的查询。

- null值处理
   - 以前对于null值需要特殊处理
```java
static void testFooBar(String s) {
    if (s == null) {
        System.out.println("Oops!");
        return;
    }
    switch (s) {
        case "Foo", "Bar" -> System.out.println("Great");
        default           -> System.out.println("Ok");
    }
}
```

   - 现在可以直接switch
```java
static void testFooBar(String s) {
    switch (s) {
        case null         -> System.out.println("Oops");
        case "Foo", "Bar" -> System.out.println("Great");
        default           -> System.out.println("Ok");
    }
}
```

- case when的支持
   - 以前
```java
class Shape {}
class Rectangle extends Shape {}
class Triangle  extends Shape { int calculateArea() { ... } }

static void testTriangle(Shape s) {
    switch (s) {
        case null:
            break;
        case Triangle t:
            if (t.calculateArea() > 100) {
                System.out.println("Large triangle");
                break;
            }
        default:
            System.out.println("A shape, possibly a small triangle");
    }
}
```

   - 现在
```java
static void testTriangle(Shape s) {
    switch (s) {
        case null -> 
            { break; }
        case Triangle t
        when t.calculateArea() > 100 ->
            System.out.println("Large triangle");
        default ->
            System.out.println("A shape, possibly a small triangle");
    }
}
```

- 支持record泛型的推断
```java
record MyPair<S,T>(S fst, T snd){};

static void recordInference(MyPair<String, Integer> pair){
    switch (pair) {
        case MyPair(var f, var s) -> 
            ... // Inferred record Pattern MyPair<String,Integer>(var f, var s)
        ...
    }
}
```

### 记录模式 (Record Patterns) 
进入第 2 预览阶段
Record Patterns 可对 record 的值进行解构，Record patterns 和 Type patterns 通过嵌套能够实现强大的、声明性的、可组合的数据导航和处理形式。**目标包括扩展模式匹配以表达更复杂、可组合的数据查询，并且不改变类型模式的语法或语义**

## Loom项目

### 作用域值（Scoped Values）
进入孵化阶段
引入 Scoped Values，它可以在线程内和线程间共享不可变数据。它们优于线程局部变量，尤其是在使用大量虚拟线程时。
**可以作为ThreadLocal的替代，在线程之间共享不可变的数据**
**但是ThreadLocal提供了set方法，变量是可变的，另外remove方法很容易被忽略，导致在线程池场景下很容易造成内存泄露。ScopedValue则提供了一种不可变、不拷贝的方案，即不提供set方法，子线程不需要拷贝就可以访问父线程的变量**
```java
final static ScopedValue<...> V = new ScopedValue<>();

// In some method
ScopedValue.where(V, <value>)
           .run(() -> { ... V.get() ... call methods ... });

// In a method called directly or indirectly from the lambda expression
... V.get() ...
```
通过ScopedValue.where可以绑定ScopedValue的值，然后在run方法里可以使用，方法执行完毕自行释放，可以被垃圾收集器回收

### 虚拟线程 (Virtual Threads) 
进入第 2 预览阶段
为 Java 引入虚拟线程，虚拟线程是 JDK 实现的轻量级线程，它在其他多线程语言中已经被证实是十分有用的，比如 Go 中的 Goroutine、Erlang 中的进程。虚拟线程避免了上下文切换的额外耗费，兼顾了多线程的优点，简化了高并发程序的复杂，可以有效减少编写、维护和观察高吞吐量并发应用程序的工作量。**与java19相比几乎没有什么变化**

### 结构化并发 (Structured Concurrency)
 进入第 2 孵化阶段
JDK 19 引入了结构化并发，这是一种多线程编程方法，目的是为了通过结构化并发 API 来简化多线程编程，并不是为了取代 java.util.concurrent。结构化并发将不同线程中运行的多个任务视为单个工作单元，从而简化错误处理、提高可靠性并增强可观察性。也就是说，结构化并发保留了单线程代码的可读性、可维护性和可观察性。**与java19相比增加了对Scoped Values继承的支持**

## Panama项目

### 外部函数和内存 API (Foreign Function & Memory API) 
进入第 2 预览阶段
引入一个 API，通过它，Java 程序可以与 Java 运行时之外的代码和数据进行互操作。通过有效地调用外部函数，以及安全地访问外部内存，该 API 使 Java 程序能够调用本地库并处理本地数据，而不会像 JNI 那样有漏洞和危险。

### 矢量API (Vector API) 
进入第 5 孵化阶段
矢量计算由对向量的一系列操作组成。矢量 API 用来表达矢量计算，该计算可以在运行时可靠地编译为支持的 CPU 架构上的最佳矢量指令，从而实现优于等效标量计算的性能。矢量 API 的目标是为用户提供简洁易用且与平台无关的表达范围广泛的矢量计算。

注意：JAVA 20 是一个短期维护版本，它会在六个月之后被下一个 LTS 版本 JAVA 21 取代；但是仍然可以应用于生产环境中，不过直接将 Java 升级到 JAVA 20 的开发者们也需要慎重。


> 原文: <https://www.yuque.com/tulingzhouyu/sfx8p0/gzxtinwv6wu5mdtg>