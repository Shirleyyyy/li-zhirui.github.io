---
layout: post
title: Null 值及其处理方式
excerpt: "讨论 null 值的由来及几种典型的表示方式和处理方式。"
categories: articles
author: ktn
date: 2016-07-22
modified: 2016-07-23
tags:
  - Type System
  - Java
  - Scala
  - Kotlin
comments: true
share: true
---

## Null 值的由来

Null 值由来已久，它最早是由 Tony Hoare 图方便而创造的，后来被证明这是个错误，而他本人也对此进行了道歉，并称之为**十亿美金错误** [^tony-hoare]。

[^tony-hoare]: [Tony Hoare - Wikipedia](https://en.wikipedia.org/wiki/Tony_Hoare#Apologies_and_retractions)

> I call it my billion-dollar mistake. It was the invention of the null reference in 1965. At that time, I was designing the first comprehensive type system for references in an object oriented language (ALGOL W). My goal was to ensure that all use of references should be absolutely safe, with checking performed automatically by the compiler. But I couldn't resist the temptation to put in a null reference, simply because it was so easy to implement. This has led to innumerable errors, vulnerabilities, and system crashes, which have probably caused a billion dollars of pain and damage in the last forty years.

## C 语言的表示

在 C 语言中，`NULL` 是一个宏，C99 标准是这样说的 [^C99]：

> An integer constant expression with the value 0, or such an expression cast to type void \*, is called a *null pointer constant*.<sup>55)</sup> If a null pointer constant is converted to a pointer type, the resulting pointer, called a null pointer, is guaranteed to compare unequal to a pointer to any object or function.

[^C99]: [C99 with Technical corrigenda TC1, TC2, and TC3 included](http://www.open-std.org/JTC1/SC22/WG14/www/docs/n1256.pdf)

也就是 `NULL` 的值就是 `0`，而 C 语言的实现必须保证这个值与任意对象和函数的地址不重复，C 语言以此来表示指针的一个特殊状态，即不指向任何有意义的对象和函数。这种处理方式影响了一堆和 C 有关的语言，比如 C++，Java 等。

## 类 C 的表示方案

在 C++ 中，Bjarne Stroustrup 出于兼容 C 语言的考量保留了这个宏，但是 Stroustrup 是反对使用 `NULL` 的，他更偏爱直接使用 `0` [^stroustrup]，毕竟用宏可能会导致一些混乱，使用 `0` 则没有这样的问题。同时，他也认为空指针需要有一个名称，而这个名称就是 `nullptr` [^stroustrup]。在 C++11 中，这个名称成为了一个关键字。

引入 `nullptr` 的好处有很多，其中一个就是解决一个重载的问题，因为直接使用 `0` 会导致在选择 `int` 还是 `foo *` 的时候出现意想不到的情况。使用了 `nullptr` 则由于 `nullptr` 的类型是指针，所以不会出现这样的问题。

[^stroustrup]: [Bjarne Stroustrup's C++ Style and Technique FAQ](http://www.stroustrup.com/bs_faq2.html#null)

相对于 C++ 的处理，由于 Java 中没有指针类型的存在，而且 Java 是一个静态强类型语言，Java 选择将 `null` 表示为一个特殊的东西。在 Java 中，`null` 是一个关键字，用来表示一个引用类型的对象没有被初始化，或是没有引用任何对象的状态，这也是类似于 C 的做法。这个关键字很特殊，因为 `null` 本身没有任何运行时类型，但是却能转换为任意的引用类型 [^jvm-ref]。但你一旦对一个 `null` 调用任何方法，或者进行拆箱，就会导致一个 `NullPointerException` 的抛出。Java 虚拟机规范甚至不确保它会以一个值的形式存在 [^jvm-ref]。所以你可以将 `null` 赋值给任意引用类型的对象，但是当调用 `instanceof` 的时候，Java 又会告诉你 `null` 不是该类型的实例，这个处理事实上非常之奇怪。

[^jvm-ref]: [Chapter 2. The Structure of the Java Virtual Machine - Java Virtual Machine Specification](http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html#jvms-2.4)

至于 Python 这边，就比 Java 要好点，因为 Python 是动态类型的语言，所以不用考虑搞什么特殊值来表示 null，直接用一个特殊的类的对象来表示就可以了，只要大家约定好，都用一个类型的一个值来表示 null，就可以对一个名称的引用情况进行判断。在 Python 中，表示 null 的对象就是 `NoneType` 的 `None`。这种处理会比 Java 在概念上纯粹一些。

以上说的这几个语言用各自的方式表示了 null，但都没有解决所谓的**十亿美金错误**，所谓**十亿美金错误**的本质在于语言的粗糙设计导致类型声明不诚实。因为你并不知道一个东西到底是一个具体的对象还是一个 null 值。一个函数说它会返回一个 `String` 类型的对象，这是真的吗？不一定，它可能返回一个 `null`，这就导致每一次使用的时候都要小心翼翼地对其进行判断，否则很有可能就会在运行的时候发生错误，这是一个非常糟糕的事情。

## 使用可空类型

Kotlin（还有 Swift 等语言）给出的解决方案 [^kotlin-ref] 是使用 Nullable type，在一个类型没有明确声明为**可空**的时候，不允许赋 null 值，仅当一个对象的类型声明后面加上一个 `?` 的时候，这个对象才是可为空的。这样，编译器就可以对此进行赋值时候的基本的判断：

[^kotlin-ref]: [Null Safety - Kotlin Reference](https://kotlinlang.org/docs/reference/null-safety.html)

```kotlin
var a: String = "abc"
a = null // error

var b: String? = "abc"
b = null
```

进一步，编译器还能在调用可空对象方法的时候报错，以防止对象为 `null` 的情况，例如：

```kotlin
val l = a.length
val l = b.length // error: variable 'b' can be null
```

另外，Kotlin 还能根据上下文来改变对对象是否可空的判定：

```kotlin
val l = if (b != null) b.length else -1
```

这是一个类型的收窄，在 `if` 表达式对 `b` 进行判断之前，`b` 是可空的 `String`，但在判断之后，编译器可以根据这个判断确定在这里 `b` 不可能为 `null`，于是，这里就允许调用其中的方法。注意这里并没有进行强制类型转换，之前不能调用对象的方法而现在可以的原因是编译器认为此时该对象的值不可能为 `null`。这个方式可以解决问题吗？显然可以，它使得用户在看到一个类型为 `A` 的对象时，可以放心地调用 `A` 中声明的方法，并强制了用户对一个可能为 `null` 的对象是否为 `null` 的判断。但这有点奇怪，编译器到底应不应该管控制流的事情呢？这是值得讨论的。但编译器通过一个一般的语句来进行对可空类型的特殊处理，总觉得是一个比较怪异的事情。在 TypeScript 中，这种技术被更广泛地应用在一般的类型处理上，比如你可以声明一个对象的类型为 `A | B | null`，并在不同的分支里判断其类型，并进行不同的处理 [^type-script-ref] [^type-script-2]。这个做法相对于对可空类型作特殊处理的做法而言要更加通用一些，但是同样的，编译器的职责和分析范围是否应该支持编译器去做这样的事情，是需要讨论的。另外 Kotlin 不像 TypeScript 这样做更加通用的处理的原因可能是要和 Java 进行更好的交互，尤其是 Java 代码对 Kotlin 代码的调用。

[^type-script-ref]: [Advanced Types - TypeScript Handbook](http://www.typescriptlang.org/docs/handbook/advanced-types.html)
[^type-script-2]: [Non-nullable types #7140](https://github.com/Microsoft/TypeScript/pull/7140)

Kotlin 对可空类型还提供了其他的处理方式来方便使用，例如安全的链式调用：

```kotlin
bob?.department?.head?.name
```

如果其中任意一步返回了 `null` 则整个表达式的结果将是 `null`。这样写起来相当直观也很方便，不用担心中途对一个 `null` 进行方法调用而抛出异常，也不需要写太多的代码来进行类型的收窄或者类型的转换。

Kotlin 还提供一个能将可空类型安全转为一般类型的方式，使用 `?:` 操作符并提供一个缺省值：

```kotlin
val l = b?.length ?: -1
```

上面这段代码和之前写的

```kotlin
val l = if (b != null) b.length else -1
```

等效，使用这个操作符就可以在不对控制流进行分析的情况下，将可空类型的对象转为一般类型的对象。从概念上来看，感觉这个处理方式要更优一些。

## 利用参数化类型表示

事实上，null 表达的只不过是一个可选的值或状态，可能有值，可能没有，这就有了另一种处理 null 的方式，它来源于 ML 系的语言（例如：SML，OCaml，Haskell 等），Scala 也借鉴了这种处理方式。它们使用参数化的类型来表示 null 这个概念。例如在 Scala 中，有一个 `Option[T]` [^scala-option] 类型，对于一个可能为空的对象，不将其类型设置为 `T` 而是设置为 `Option[T]`。

```scala
val str: String = "abc"
val optionStr1: Option[String] = None
val optionStr2: Option[String] = Some("abc")
```

这就解决了问题。而最为重要的事是，这种做法不需要为这个 null 的表示而专门添加语法特性，所有的操作都可以直接通过查阅 API 文档获知。如果喜欢，还可以自己写自己的 `MyOption[T]`，又或是通过隐式转换来给 `Option[T]` 添加需要的操作，这样的设计使得语言更加纯粹。

[^scala-option]: [Option - Scala API References](http://www.scala-lang.org/api/2.11.8/#scala.Option)

Scala 给 `Option[T]` 提供了非常丰富的操作，比如可以对 `Option[T]` 所包裹的 `T` 类型对象进行 `map`，`filter`，`getOrElse` 等，例如：

```scala
val name: Option[String] = request getParameter "name"
val upper = name map { _.trim } filter { _.length != 0 } map { _.toUpperCase }
println(upper getOrElse "")
```

其中，`getOrElse` 的作用和 Kotlin 里的 `?:` 操作符的作用一样，Scala 的处理显然更好，因为根本没必要为一个可以通过方法调用解决的事情专门做一个新的语言特性。

如果对某个方法的调用也可能产生新的 `Option[R]` 又该怎么办？如果直接使用 `map` 则会导致嵌套的 `Option`，解决方案就是加一个 `flatMap` 方法：

```scala
// optA: Option[A]
optA.flatMap(_.getOptionB)
```

如果 `optA` 包裹的 `A` 类型对象返回了 `Some(b)` 则结果为 `Some(b)`，如果 `optA` 或调用 `getOptionB` 返回的值有一个为 `None`，则整个的结果为 `None`。使用 `flatMap` 会有一个问题，就是当需要同时用到几个 `Option` 包裹的值的时候会出现嵌套的 `flatMap`，考虑如下情况：

```scala
val firstName: Option[String] = request getParameter "first-name"
val lastName: Option[String] = request getParameter "last-name"
val company: Option[String] = request getParameter "company"
val record: Option[Record] = firstName.flatMap { fName =>
  lastName.flatMap { lName =>
    company.flatMap { com =>
      Record(fName, lName, com)
    }
  }
}
```

Scala 提供了 for-comprehension 来解决这个问题，这个设计类似于 Haskell 的 do-comprehension 但更为强大：

```scala
val record: Option[Record] = for {
  fName <- firstName
  lName <- lastName
  com <- company
} yield Record(fName, lName, com)
```

这种表示更为清晰直观，且避免了嵌套。而且，这个语法糖不是为了 `Option` 专门制造的，而是能用于所有定义了 `flatMap`，`map` 和 `filter` 的场合，比如 `List`。更为准确的说，for-comprehension 提供了一个更好的操作 Monad 的方式，这里就不展开叙述了。

除此之外，Scala 还能对其进行模式匹配，这也不是为 `Option` 专门设计的，但实现了类似 Kotlin 中根据控制流来进行类型收窄的效果：

```scala
val nameMaybe = request getParameter "name"
nameMaybe match {
  case Some(name) =>
    println(name.trim.toUppercase)
  case None =>
    println("No name value")
}
```

需要注意的是，这里的类型匹配相当于对一个对象进行了类型的判断，并将类型转换为指定类型，不需要编译器对某个表达式进行特化的分析就保证了类型的安全。

当然 Scala 这个解决方案相对于 Kotlin 也有一个缺点，就是它并非是强制的，为了和 Java 交互，null 这个概念必须要保留，所以，Scala 也可以对一个对象赋 `null` 值，这样，在调用 Java 的代码或是调用不可信的 Scala 代码时，还是免不了要进行 `null` 的判断。这并非是这个处理方式的缺点，如果是语言本身不支持 `null`，则完全不会出现这样的问题，例如 Haskell，Rust 等。

这个表示方式的巨大优点在于它很容易在其他语言里实现，不需要新的语言特性。在 Java 8 中，Java 也引入了这个处理方式，在 Java 中这个类型是 `Optional<T>` [^java-ref-optional]，它也提供了类似的方式，但由于没用好用的语法糖，导致使用的时候没这么美观。幸好 Java 8 有了 lambda 表达式，否则可能会变成一大堆匿名内部类的情况，估计就不会有人想用这个类了。另外，出于兼容性考量，类似于 Scala 面临的问题，由于 `null` 本身的保留，Java 即便引入了 `Optional<T>` 也难以免去对 `null` 的判断。

[^java-ref-optional]: [Optional - JavaSE 8 References](https://docs.oracle.com/javase/8/docs/api/java/util/Optional.html)

## 总结

总之，空值这个概念必然是需要的，只是表示方式有所不同。如何处理才能更好地利用编译器来帮程序员及早发现错误是一个需要精心设计的事情。所谓**十亿美金错误**的本质在于语言的粗糙设计导致类型声明不诚实，一个值或是接口的用户无法通过类型声明确信他所获得的值的类型究竟是什么。在新生代的语言中，基本上都会对 null 这个 bug 温床进行一些处理，具体处理的方式算是各有优劣。由于兼容性的问题，老的语言里可能还是免不了见到 `null`，但新写的代码最好还是使用更好的处理方式，避免**十亿美金错误**。

## 参考
