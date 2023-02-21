---
title: "Type Parameters Proposal"
date: 2023-02-20T03:54:48+08:00
draft: false
tags: ["translate", "Golang", "Generics", "RFC"]
---

## Status

This is the design for adding generic programming using type
parameters to the Go language.
This design has been [proposed and
accepted](https://golang.org/issue/43651) as a future language change.
We currently expect that this change will be available in the Go 1.18
release in early 2022.

这是在Go语言中加入使用`Type Parameter`的泛型编程的设计。
这个设计已经被[提出并接受](https://golang.org/issue/43651)为未来的语言变化。
我们目前预计，这一变化将在2022年初的`Go 1.18`版本中出现。


## Abstract

We suggest extending the Go language to add optional type parameters
to type and function declarations.
Type parameters are constrained by interface types.
Interface types, when used as type constraints, support embedding
additional elements that may be used to limit the set of types that
satisfy the constraint.
Parameterized types and functions may use operators with type
parameters, but only when permitted by all types that satisfy the
parameter's constraint.
Type inference via a unification algorithm permits omitting type
arguments from function calls in many cases.
The design is fully backward compatible with Go 1.

我们建议扩展`Go`语言，在类型和函数声明中增加可选的`type parameter`。

`type parameter`受到接口类型的约束。

接口类型作为类型约束时，支持嵌入额外的元素，可用于限制满足约束的类型集。

参数化的类型和函数可以使用带有`type parameter`的操作符，但只有在满足参数约束的所有类型允许的情况下。

通过统一算法进行的类型推断允许在许多情况下从函数调用中省略`type parameter`。

该设计完全向后兼容 `Go 1`。

## How to read this proposal

This document is long.
Here is some guidance on how to read it.

* We start with a high level overview, describing the concepts very
  briefly.
* We then explain the full design starting from scratch, introducing
  the details as we need them, with simple examples.
* After the design is completely described, we discuss implementation,
  some issues with the design, and a comparison with other approaches
  to generics.
* We then present several complete examples of how this design would
  be used in practice.
* Following the examples some minor details are discussed in an
  appendix.

这份文件很长, 这里有一些关于如何阅读它的指导：
* 我们从高级概述开始，非常简要地描述概念。
* 然后我们从头开始解释完整的设计，在我们需要的时候，用简单的例子介绍细节。
* 在完整地描述了设计之后，我们讨论了实现，设计中的一些问题，以及与其他泛型方法的比较。
* 然后，我们提出了几个完整的例子，说明这种设计在实践中会如何使用。
* 在例子之后，在附录中讨论了一些小的细节


## Very high level overview

This section explains the changes suggested by the design very
briefly.
This section is intended for people who are already familiar with how
generics would work in a language like Go.
These concepts will be explained in detail in the following sections.

* Functions can have an additional type parameter list that uses
  square brackets but otherwise looks like an ordinary parameter list:
  `func F[T any](p T) { ... }`.
* These type parameters can be used by the regular parameters and in
  the function body.
* Types can also have a type parameter list: `type M[T any] []T`.
* Each type parameter has a type constraint, just as each ordinary
  parameter has a type: `func F[T Constraint](p T) { ... }`.
* Type constraints are interface types.
* The new predeclared name `any` is a type constraint that permits any
  type.
* Interface types used as type constraints can embed additional
  elements to restrict the set of type arguments that satisfy the
  contraint:
  * an arbitrary type `T` restricts to that type
  * an approximation element `~T` restricts to all types whose
    underlying type is `T`
  * a union element `T1 | T2 | ...` restricts to any of the listed
    elements
* Generic functions may only use operations supported by all the types
  permitted by the constraint.
* Using a generic function or type requires passing type arguments.
* Type inference permits omitting the type arguments of a function
  call in common cases.

这一节非常简要地解释了泛型在`Go`语言中的设计，是为已经有在某些语言中有使用泛型的经验的人而准备的。

这些概念将在下面的章节中详细解释。

* 函数可以有一个额外的`type parameter`列表，它使用方括号，但其他方面看起来像一个普通的参数列表 `func F[T any](p T) { ... }`。
* 这些类型的参数可以由常规参数和在函数体中使用。
* 类型也可以有一个`type parameter`列表：`type M[T any] []T`。
* 每个`type parameter`都有一个类型约束，就像每个普通参数都有一个类型一样：`func F[T Constraint](p T) { ... }`。
* 类型约束是接口类型。
* 将会预置一个新的类型`any`, 是允许任何类型的类型约束。
* 用作类型约束的接口类型可以嵌入额外的元素来限制满足约束的`type arguments`集：
    * 任意类型 `T` 限制为该类型。
    * 一个近似元素`~T`限制了底层类型为`T`的所有类型。
    * 也可以使用组合元素  `T1 | T2 | ...`，限制到所列的任何元素。
* `Generic Functions` 只能使用约束条件所允许的所有类型所支持的操作。
* 使用`Generic Functions`或类型需要传递`type arguments`。
* 类型推断允许在常见情况下省略函数调用的`type arguments`。


## Background

There have been many [requests to add additional support for generic
programming](https://github.com/golang/go/wiki/ExperienceReports#generics)
in Go.
There has been extensive discussion on
[the issue tracker](https://golang.org/issue/15292) and on
[a living document](https://docs.google.com/document/d/1vrAy9gMpMoS3uaVphB32uVXX4pi-HnNjkMEgyAHX4N4/view).

This design suggests extending the Go language to add a form of
parametric polymorphism, where the type parameters are bounded not by
a declared subtyping relationship (as in some object oriented
languages) but by explicitly defined structural constraints.

This version of the design has many similarities to a design draft
presented on July 31, 2019, but contracts have been removed and
replaced by interface types, and the syntax has changed.

There have been several proposals for adding type parameters, which
can be found through the links above.
Many of the ideas presented here have appeared before.
The main new features described here are the syntax and the careful
examination of interface types as constraints.

This design does not support template metaprogramming or any other
form of compile time programming.

As the term _generic_ is widely used in the Go community, we will
use it below as a shorthand to mean a function or type that takes type
parameters.
Don't confuse the term generic as used in this design with the same
term in other languages like C++, C#, Java, or Rust; they have
similarities but are not the same.

有很多人要求在Go中[增加对通用编程的额外支持](https://github.com/golang/go/wiki/ExperienceReports#generics)。这些功能在[the issue tracker](https://golang.org/issue/15292) 和 [a living document](https://docs.google.com/document/d/1vrAy9gMpMoS3uaVphB32uVXX4pi-HnNjkMEgyAHX4N4/view) 中进行了广泛的讨论。

某些人建议对Go语言进行扩展，增加一种参数化的多态性，其中`type parameters`不是由已声明的子类型关系（如一些面向对象的语言）来约束，而是由明确定义的结构约束。这个版本的设计与2019年7月31日提交的设计草案有许多相似之处，但`contracts`已被删除，由接口类型取代，而且语法也有变化。


已经有几个关于增加`type parameters`的建议，可以通过上面的链接找到。这里提出的许多想法以前就出现过。这里描述的主要新特征是语法和对接口类型作为约束条件的仔细检查。

这种设计不支持模板元编程或任何其他形式的编译时编程。~~~哈哈哈（^.^）

由于泛型一词在`Go`社区中被广泛使用，我们将在下文中把它作为一个速记符号，指的是一个接受`type parameters`的函数或类型。不要把本设计中使用的泛型一词与其他语言如`C++`、`C#`、`Java`或`Rust`中的相同术语混淆，它们有相似之处，但并不一样。 ~~~哈哈哈哈（^.^）


## Design

We will describe the complete design in stages based on simple
examples.

我们将根据简单的例子，分阶段描述完整的设计。



### Type parameters

Generic code is written using abstract data types that we call _type
parameters_.

泛型代码是使用抽象的数据类型编写的，我们称之为`type parameters`。

When running the generic code, the type parameters are replaced by
_type arguments_.

当运行泛型代码时，`type arguments`被类型实体参数所取代。

```go
// 原文没有 下面好多 type parameters 与 type arguments 别搞混了 
Print[T any](s []T)  // T is Parameters
Print[int]([]int{1, 2, 3}) // int is arguments
```

Here is a function that prints out each element of a slice, where the
element type of the slice, here called `T`, is unknown.
This is a trivial example of the kind of function we want to permit in
order to support generic programming.
(Later we'll also discuss [generic types](#Generic-types)).

这是一个打印出切片的每个元素的函数，其中切片的元素类型 `T` , 然而它是未知的类型。
这是我们为了支持泛型编程而希望允许的那种函数的一个简单示例。（稍后我们将会讨论 [generic types](#Generic-types)）
```go
// Print prints the elements of a slice.
// It should be possible to call this with any slice value.
func Print(s []T) { // Just an example, not the suggested syntax.
	for _, v := range s {
		fmt.Println(v)
	}
}
```

With this approach, the first decision to make is: how should the type
parameter `T` be declared?
In a language like Go, we expect every identifier to be declared in
some way.

采用这种方法，首先要做的决定是：应该如何声明`type parameters` `T`？

在像Go这样的语言中，我们希望每个标识符都能以某种方式被声明

Here we make a design decision: type parameters are similar to
ordinary non-type function parameters, and as such should be listed
along with other parameters.
However, type parameters are not the same as non-type parameters, so
although they appear in the list of parameters we want to distinguish
them.
That leads to our next design decision: we define an additional
optional parameter list describing type parameters.

这里我们做了一个设计决定：`type parameters`与普通的非类型函数参数相似，因此应该与其他参数一起列出。
然而，`type parameters`与非`type parameters`不一样，所以尽管它们出现在参数列表中，我们还是要将它们区分开来。
这导致了我们的下一个设计决定：我们定义了一个额外的可选参数列表来描述`type parameters`。

This type parameter list appears before the regular parameters.
To distinguish the type parameter list from the regular parameter
list, the type parameter list uses square brackets rather than
parentheses.
Just as regular parameters have types, type parameters have
meta-types, also known as constraints.
We will discuss the details of constraints later; for now, we will
just note that `any` is a valid constraint, meaning that any type is
permitted.

这个`type parameters`列表出现在常规参数的前面。为了区分`type parameters`列表和常规参数列表，`type parameters`列表使用方括号而不是小括号。

就像常规参数有类型一样，`type parameters`也有元类型，也被称为约束。

我们将在后面讨论约束条件的细节；现在，我们只需注意任何是一个有效的约束条件，意味着任何类型都是允许的。

```go
// Print prints the elements of any slice.
// Print has a type parameter T and has a single (non-type)
// parameter s which is a slice of that type parameter.
func Print[T any](s []T) {
	// same as above
}
```

This says that within the function `Print` the identifier `T` is a
type parameter, a type that is currently unknown but that will be
known when the function is called.
The `any` means that `T` can be any type at all.
As seen above, the type parameter may be used as a type when
describing the types of the ordinary non-type parameters.
It may also be used as a type within the body of the function.

这就是说，在函数`Print`中，标识符`T`是一个`type parameters`，这个类型目前是未知的，但当函数被调用时将会被知道。任何意味着`T`可以是任何类型。如上所述，在描述普通`non-type parameters`的类型时，类`type parameters`可以作为一个类型使用。它也可以在函数的主体中作为一个类型使用。

Unlike regular parameter lists, in type parameter lists names are
required for the type parameters.
This avoids a syntactic ambiguity, and, as it happens, there is no
reason to ever omit the type parameter names.

与普通的参数列表不同，在`type parameters`列表中，`type parameters`的名字是必须的。这就避免了语法上的歧义，而且，恰好没有理由省略`type parameters`的名称。

Since `Print` has a type parameter, any call of `Print` must provide a
type argument.
Later we will see how this type argument can usually be deduced from
the non-type argument, by using [type inference](#Type-inference).
For now, we'll pass the type argument explicitly.
Type arguments are passed much like type parameters are declared: as a
separate list of arguments.
As with the type parameter list, the list of type arguments uses
square brackets.

由于 `Print` 有一个`type parameters`，因此任何 `Print` 调用都必须提供一个`type argument`。

稍后我们将看到通常如何使用[类型推断](#Type-inference)从非类型参数中推导出此`type argument`。

现在，我们将显式传递`type argument`。`type argument`的传递很像`type parameters`的声明：作为一个单独的参数列表。

与`type parameters`列表一样，`type argument`列表使用方括号。

```go
	// Call Print with a []int.
	// Print has a type parameter T, and we want to pass a []int,
	// so we pass a type argument of int by writing Print[int].
	// The function Print[int] expects a []int as an argument.
	Print[int]([]int{1, 2, 3})

	// This will print:
	// 1
	// 2
	// 3
```

### Constraints

Let's make our example slightly more complicated.
Let's turn it into a function that converts a slice of any type into a
`[]string` by calling a `String` method on each element.

我们的例子稍微复杂一点，把它变成一个函数，通过在每个元素上调用 `String` 方法将任何类型的切片转换为 `[]string`。

```go
// This function is INVALID.
func Stringify[T any](s []T) (ret []string) {
	for _, v := range s {
		ret = append(ret, v.String()) // INVALID
	}
	return ret
}
```

This might seem OK at first glance, but in this example `v` has type
`T`, and `T` can be any type.
This means that `T` need not have a `String` method.
So the call to `v.String()` is invalid.

乍看之下这似乎没问题，但在这个例子中，`v`有`T`类型，而`T`可以是任何类型。这意味着`T`不需要有一个`String`方法。所以对`v.String()`的调用是无效的。

Naturally, the same issue arises in other languages that support
generic programming.
In C++, for example, a generic function (in C++ terms, a function
template) can call any method on a value of generic type.
That is, in the C++ approach, calling `v.String()` is fine.
If the function is called with a type argument that does not have a
`String` method, the error is reported when compiling the call to
`v.String` with that type argument.
These errors can be lengthy, as there may be several layers of generic
function calls before the error occurs, all of which must be reported
to understand what went wrong.

自然，同样的问题也出现在其他支持泛型编程的语言中。例如，在`C++`中，一个泛型函数（用`C++`的话来说，就是一个函数模板）可以调用泛型类型的值上的任何方法。也就是说，在`C++`方法中，调用`v.String()`是可以的。如果用一个没有`String`方法的`type argument`来调用该函数，那么在用该`type argument`编译调用`v.String`时就会报告错误。这些错误可能很冗长，因为在错误发生之前可能有几层泛型函数的调用，所有这些都必须被报告以了解出错的原因。

The C++ approach would be a poor choice for Go.
One reason is the style of the language.
In Go we don't refer to names, such as, in this case, `String`, and
hope that they exist.
Go resolves all names to their declarations when they are seen.

`C++` 方法对于 `Go` 来说不是一个好的选择, 原因之一是语言的风格。

在 `Go` 中，我们不会`refer to names`，例如本例中的 `String`，并希望它们存在。

 `Go` 在看到它们时将所有`names`解析为它们(可能代指 `function`)的声明。

Another reason is that Go is designed to support programming at
scale.
We must consider the case in which the generic function definition
(`Stringify`, above) and the call to the generic function (not shown,
but perhaps in some other package) are far apart.
In general, all generic code expects the type arguments to meet
certain requirements.
We refer to these requirements as _constraints_ (other languages have
similar ideas known as type bounds or trait bounds or concepts).
In this case, the constraint is pretty obvious: the type has to have a
`String() string` method.

另一个原因是，`Go`的设计是为了支持规模化的编程。我们必须考虑这样的情况：泛型函数的定义（上面的`Stringify`）和对泛型函数的调用（没有显示，但可能在其他包中）相距甚远。一般来说，所有的泛型代码都希望`type argument`能够满足某些要求。我们把这些要求称为约束（其他语言也有类似的想法，称为类型约束或特性约束或概念）。在这种情况下，约束条件是非常明显的：该类型必须有一个`String()`字符串方法。

We don't want to derive the constraints from whatever `Stringify`
happens to do (in this case, call the `String` method).
If we did, a minor change to `Stringify` might change the
constraints.
That would mean that a minor change could cause code far away, that
calls the function, to unexpectedly break.
It's fine for `Stringify` to deliberately change its constraints, and
force callers to change.
What we want to avoid is `Stringify` changing its constraints
accidentally.

我们不希望从 `Stringify` 碰巧做的任何事情中推导出约束。如果我们这样做了，对 `Stringify` 的一个小改动可能会改变约束。这意味着一个微小的变化可能会导致程序别处`Break`， 这在具有规模的项目开发里是不被希望的。 `Stringify` 可以故意更改其约束并强制调用者更改。我们要避免的是 `Stringify` 意外更改其约束。

This means that the constraints must set limits on both the type
arguments passed by the caller and the code in the generic function.
The caller may only pass type arguments that satisfy the constraints.
The generic function may only use those values in ways that are
permitted by the constraints.
This is an important rule that we believe should apply to any attempt
to define generic programming in Go: generic code can only use
operations that its type arguments are known to implement.

这意味着约束必须对调用者传递的`type argument`和泛型函数中的代码都设置限制。调用者只能传递满足约束条件的`type argument`。`Generic Functions` 只能以约束条件允许的方式使用这些值。这是一条重要的规则，我们认为它应该适用于任何在`Go`中定义泛型编程的尝试：泛型代码只能使用其`type argument`已经实现的接口约束的方法。



### Operations permitted for any type

Before we discuss constraints further, let's briefly note what happens
when the constraint is `any`.
If a generic function uses the `any` constraint for a type parameter,
as is the case for the `Print` method above, then any type argument is
permitted for that parameter.
The only operations that the generic function can use with values of
that type parameter are those operations that are permitted for values
of any type.
In the example above, the `Print` function declares a variable `v`
whose type is the type parameter `T`, and it passes that variable to a
function.

在我们进一步讨论约束之前，让我们简要说明当约束为 `any` 时会发生什么。如果泛型函数对`type parameter`使用 `any` 约束，如上述 `Print` 方法的情况，则该参数允许任何`type argument`。泛型函数可以对该`type argument`的值使用的唯一操作是那些允许用于任何类型的值的操作。在上面的示例中，Print 函数声明了一个类型为`type parameters` `T` 的变量 `v`，并将该变量传递给一个函数。

The operations permitted for any type are:


* declare variables of those types
* assign other values of the same type to those variables
* pass those variables to functions or return them from functions
* take the address of those variables
* convert or assign values of those types to the type `interface{}`
* convert a value of type `T` to type `T` (permitted but useless)
* use a type assertion to convert an interface value to the type
* use the type as a case in a type switch
* define and use composite types that use those types, such as a slice
  of that type
* pass the type to some predeclared functions such as `new`


对于 `any` 来说，它被允许的操作如下：

* 声明这些类型的变量
* 将相同类型的其他值分配给这些变量
* 将这些变量传递给函数或从函数中返回这些变量
* 获取这些变量的地址
* 将这些类型的值转换或者分配给 `interface{}`
* 使用类型断言将接口值转换为具体类型
* 在 `type switch`中 将这个类型作为 一种 `case`使用
* 定义和使用使用这些类型的复合类型，例如该类型的切片
* 将类型传递给一些预先声明的函数，例如 new

It's possible that future language changes will add other such
operations, though none are currently anticipated.

在以后上面的列出来的允许的操作有可能被更改，也有可能添加新的，但目前预计不会。



### Defining constraints

Go already has a construct that is close to what we need for a
constraint: an interface type.
An interface type is a set of methods.
The only values that can be assigned to a variable of interface type
are those whose types implement the same methods.
The only operations that can be done with a value of interface type,
other than operations permitted for any type, are to call the
methods.

Go 已经有一个接近于我们需要的约束的构造：接口类型。

接口类型是一组方法。

唯一可以分配给接口类型变量的值是那些其类型实现了相同方法的值。

除了任何类型允许的操作之外，可以使用接口类型的值完成的唯一操作是调用方法。

Calling a generic function with a type argument is similar to
assigning to a variable of interface type: the type argument must
implement the constraints of the type parameter.
Writing a generic function is like using values of interface type: the
generic code can only use the operations permitted by the constraint
(or operations that are permitted for any type).

使用`type argument`调用泛型函数类似于分配给接口类型的变量：`type argument`必须实现`type parameters`的约束。

编写泛型函数就像使用接口类型的值：泛型代码只能使用约束允许的操作（或任何类型允许的操作）。

Therefore, in this design, constraints are simply interface types.
Satisfying a constraint means implementing the interface type.
(Later we'll restate this in order to define constraints for
operations other than method calls, such as [binary operators](#Operators)).


因此，在这个设计中，约束只是接口类型。满足约束意味着实现接口类型。 （稍后我们将重申这一点，以便为方法调用以外的操作定义约束，例如[binary operators](#Operators))

For the `Stringify` example, we need an interface type with a `String`
method that takes no arguments and returns a value of type `string`.
对于Stringify的例子，我们需要一个具有String方法的接口类型，该方法不需要参数，并返回一个字符串类型的值。

```go
// Stringer is a type constraint that requires the type argument to have
// a String method and permits the generic function to call String.
// The String method should return a string representation of the value.
type Stringer interface {
	String() string
}
```

(It doesn't matter for this discussion, but this defines the same
interface as the standard library's `fmt.Stringer` type, and  real
code would likely simply use `fmt.Stringer`.)

(这对这次讨论并不重要，但这定义了与标准库的`fmt.Stringer`类型相同的接口，真正的代码可能会简单地使用`fmt.Stringer`。)


### The `any` constraint

Now that we know that constraints are simply interface types, we can
explain what `any` means as a constraint.
As shown above, the `any` constraint permits any type as a type
argument and only permits the function to use the operations permitted
for any type.
The interface type for that is the empty interface: `interface{}`.
So we could write the `Print` example as

现在我们知道约束只是接口类型，我们可以解释什么是约束。如上所示，`any` 约束允许任何类型作为`type argument`，并且只允许函数使用任何类型允许的操作。其接口类型是空接口：`interface{}`。所以我们可以将 `Print` 示例写成：

```go
// Print prints the elements of any slice.
// Print has a type parameter T and has a single (non-type)
// parameter s which is a slice of that type parameter.
func Print[T interface{}](s []T) {
	// same as above
}
```

However, it's tedious to have to write `interface{}` every time you
write a generic function that doesn't impose constraints on its type
parameters.
So in this design we suggest a type constraint `any` that is
equivalent to `interface{}`.
This will be a predeclared name, implicitly declared in the universe
block.
It will not be valid to use `any` as anything other than a type
constraint.

但是，每次编写不对其`type parameters`施加约束的泛型函数时都必须编写 `interface{}` 是很乏味的。
所以在这个设计中我们建议一个类型约束 `any` 等同于 `interface{}`。
这将是 `Go` 语言中的一个预先内置的隐式声明， 全局有效。
另外将 `any` 用作类型约束以外的任何内容都是无效的。

(Note: clearly we could make `any` generally available as an alias for
`interface{}`, or as a new defined type defined as `interface{}`.
However, we don't want this design, which is about generics, to lead
to a possibly significant change to non-generic code.
Adding `any` as a general purpose name for `interface{}` can and
should be [discussed separately](https://golang.org/issue/33232)).

（注意：我们可以使任何代码中的变量 作为 `interface{}` 的别名，或作为定义为 `interface{}` 的新定义类型。但是，我们不希望这种关于泛型的设计可能导致对非通用代码的重大更改。将 `any` 添加为 `interface{}` 的通用名称可以而且应该[单独讨论](https://golang.org/issue/33232)）。


### Using a constraint

For a generic function, a constraint can be thought of as the type of
the type argument: a meta-type.
As shown above, constraints appear in the type parameter list as the
meta-type of a type parameter.

对于一个泛型函数，约束可以被认为是`type argument`的类型：`a meta-type`。

如上所示，约束在`type parameters`列表中作为`type parameters`的`meta-type`出现。

```go
// Stringify calls the String method on each element of s,
// and returns the results.
func Stringify[T Stringer](s []T) (ret []string) {
	for _, v := range s {
		ret = append(ret, v.String())
	}
	return ret
}
```

The single type parameter `T` is followed by the constraint that
applies to `T`, in this case `Stringer`.

单一`type parameters` `T` 后跟适用于 `T` 的约束，在本例中为 `Stringer`。

### Multiple type parameters

Although the `Stringify` example uses only a single type parameter,
functions may have multiple type parameters.

尽管 `Stringify` 示例仅使用单个`type parameters`，但函数可能具有多个`type parameters`。

```
// Print2 has two type parameters and two non-type parameters.
func Print2[T1, T2 any](s1 []T1, s2 []T2) { ... }
```

对比下面的代码

```go
// Print2Same has one type parameter and two non-type parameters.
func Print2Same[T any](s1 []T, s2 []T) { ... }
```

In `Print2` `s1` and `s2` may be slices of different types.
In `Print2Same` `s1` and `s2` must be slices of the same element
type.

在 `Print2` 中，`s1` 和 `s2` 可能是不同类型的切片。
在 `Print2Same` 中，`s1` 和 `s2` 必须是相同元素类型的切片。


Just as each ordinary parameter may have its own type, each type
parameter may have its own constraint.

就像每个普通参数可以有自己的类型一样，每个`type parameters`也可以有自己的约束条件。

```go
// Stringer is a type constraint that requires a String method.
// The String method should return a string representation of the value.
type Stringer interface {
	String() string
}

// Plusser is a type constraint that requires a Plus method.
// The Plus method is expected to add the argument to an internal
// string and return the result.
type Plusser interface {
	Plus(string) string
}

// ConcatTo takes a slice of elements with a String method and a slice
// of elements with a Plus method. The slices should have the same
// number of elements. This will convert each element of s to a string,
// pass it to the Plus method of the corresponding element of p,
// and return a slice of the resulting strings.
func ConcatTo[S Stringer, P Plusser](s []S, p []P) []string {
	r := make([]string, len(s))
	for i, v := range s {
		r[i] = p[i].Plus(v.String())
	}
	return r
}
```

A single constraint can be used for multiple type parameters, just as
a single type can be used for multiple non-type function parameters.
The constraint applies to each type parameter separately.

单个约束可用于多个`type parameters`，就像单个类型可用于多个非类型函数参数一样。该约束分别应用于每个`type parameters`。

```go
// Stringify2 converts two slices of different types to strings,
// and returns the concatenation of all the strings.
func Stringify2[T1, T2 Stringer](s1 []T1, s2 []T2) string {
	r := ""
	for _, v1 := range s1 {
		r += v1.String()
	}
	for _, v2 := range s2 {
		r += v2.String()
	}
	return r
}
```

### Generic types

We want more than just generic functions: we also want generic types.
We suggest that types be extended to take type parameters.

我们想要的不仅仅是泛型函数：我们还想要泛型类型。

```go
// Vector is a name for a slice of any element type.
type Vector[T any] []T
```

A type's type parameters are just like a function's type parameters.

一个类型的`type parameters`就像一个函数的`type parameters`一样。

Within the type definition, the type parameters may be used like any
other type.

在类型定义中，`type parameters`可以像其他类型一样使用。

To use a generic type, you must supply type arguments.
This is called _instantiation_.
The type arguments appear in square brackets, as usual.
When we instantiate a type by supplying type arguments for the type
parameters, we produce a type in which each use of a type parameter in
the type definition is replaced by the corresponding type argument.

要使用泛型类型，您必须提供`type argument`。
这称为 `_instantiation_`。
像往常一样，`type arument`出现在方括号中。
当我们通过为`type paramenters`提供`type argument`来实例化一个类型时，我们生成了一个类型，其中类型定义中的每个`type parameters`的使用都被相应的`type argument`替换。

```go
// v is a Vector of int values.
//
// This is similar to pretending that "Vector[int]" is a valid identifier,
// and writing
//   type "Vector[int]" []int
//   var v "Vector[int]"
// All uses of Vector[int] will refer to the same "Vector[int]" type.
//
var v Vector[int]
```

Generic types can have methods.
The receiver type of a method must declare the same number of type
parameters as are declared in the receiver type's definition.
They are declared without any constraint.

泛型的类型可以有方法。方法的`receiver`类型必须声明与`receiver`类型定义中声明的相同数量的类型参数。它们的声明没有任何限制。

```go
// Push adds a value to the end of a vector.
func (v *Vector[T]) Push(x T) { *v = append(*v, x) }
```

The type parameters listed in a method declaration need not have the
same names as the type parameters in the type declaration.
In particular, if they are not used by the method, they can be `_`.

方法声明中列出的类型参数不必与类型声明中的类型参数同名。特别是，如果它们不被方法使用，它们可以是`_`。

A generic type can refer to itself in cases where a type can
ordinarily refer to itself, but when it does so the type arguments
must be the type parameters, listed in the same order.
This restriction prevents infinite recursion of type instantiation.

在类型通常可以引用自身的情况下，泛型类型可以引用自身，但是当它这样做时，`type arguments`必须是`type parameter`，***以相同的顺序列出***。此限制可防止类型实例化的无限递归。

```go
// List is a linked list of values of type T.
type List[T any] struct {
	next *List[T] // this reference to List[T] is OK
	val  T
}

// This type is INVALID.
type P[T1, T2 any] struct {
	F *P[T2, T1] // INVALID; must be [T1, T2] (0.0) 注意这里
}
```

This restriction applies to both direct and indirect references.

此限制适用于直接和间接引用。

```go
// ListHead is the head of a linked list.
type ListHead[T any] struct {
	head *ListElement[T]
}

// ListElement is an element in a linked list with a head.
// Each element points back to the head.
type ListElement[T any] struct {
	next *ListElement[T]
	val  T
	// Using ListHead[T] here is OK.
	// ListHead[T] refers to ListElement[T] refers to ListHead[T].
	// Using ListHead[int] would not be OK, as ListHead[T]
	// would have an indirect reference to ListHead[int].
	head *ListHead[T]
}
```

(Note: with more understanding of how people want to write code, it
may be possible to relax this rule to permit some cases that use
different type arguments.)
(注意：随着社区的深入讨论，我们对`Gopher`写代码的方式有了更多的了解，也许可以放宽这个规则，允许一些使用不同`type argument`的情况)。

The type parameter of a generic type may have constraints other than
`any`.

泛型类型的`type parameters`可能具有除任何约束之外的约束。

```go
// StringableVector is a slice of some type, where the type
// must have a String method.
type StringableVector[T Stringer] []T

func (s StringableVector[T]) String() string {
	var sb strings.Builder
	for i, v := range s {
		if i > 0 {
			sb.WriteString(", ")
		}
		// It's OK to call v.String here because v is of type T
		// and T's constraint is Stringer.
		sb.WriteString(v.String())
	}
	return sb.String()
}
```

### Methods may not take additional type arguments

Although methods of a generic type may use the type's parameters,
methods may not themselves have additional type parameters.
Where it would be useful to add type arguments to a method, people
will have to write a suitably parameterized top-level function.

There is more discussion of this in [the issues
section](#No-parameterized-methods).

虽然泛型类型的方法可以使用类型的参数，但方法本身可能没有额外的`type's parameters`。在向方法添加`type parameters`有用的地方，人们将不得不编写一个适当参数化的顶级函数。
在[the issues
section](#No-parameterized-methods)对此有更多讨论。

### Operators

As we've seen, we are using interface types as constraints.
Interface types provide a set of methods, and nothing else.
This means that with what we've seen so far, the only thing that
generic functions can do with values of type parameters, other than
operations that are permitted for any type, is call methods.

正如我们所见，我们使用接口类型作为约束。接口类型提供一组方法，仅此而已。
这意味着，就我们目前所见，泛型函数唯一可以对`type parameters`值执行的操作是调用方法，而不是允许对任何类型执行的操作。

However, method calls are not sufficient for everything we want to
express.
Consider this simple function that returns the smallest element of a
slice of values, where the slice is assumed to be non-empty.

然而，对于我们想要表达的一切，方法调用是不够的。考虑这个简单的函数，它返回值切片的最小元素，其中切片被假定为非空。

```go
// This function is INVALID.
func Smallest[T any](s []T) T {
	r := s[0] // panic if slice is empty
	for _, v := range s[1:] {
		if v < r { // INVALID
			r = v
		}
	}
	return r
}
```

Any reasonable generics implementation should let you write this
function.
The problem is the expression `v < r`.
This assumes that `T` supports the `<` operator, but the constraint on
`T` is simply `any`.
With the `any` constraint the function `Smallest` can only use
operations that are available for all types, but not all Go types
support `<`.
Unfortunately, since `<` is not a method, there is no obvious way to
write a constraint&mdash;an interface type&mdash;that permits `<`.

任何合理的泛型实现都应该让您编写此函数。问题是表达式 `v < r`。
这假定 `T `支持 `<` 运算符，但对 T 的约束只是`any`。在 `any` 约束下，函数 `Smallest` 只能使用适用于所有类型的操作，但并非所有 `Go` 类型都支持 `<`。
不幸的是，由于 `<` 不是方法，因此没有明显的方法来编写允许 `<` 的约束（接口类型）。

We need a way to write a constraint that accepts only types that
support `<`.
In order to do that, we observe that, aside from two exceptions that
we will discuss later, all the arithmetic, comparison, and logical
operators defined by the language may only be used with types that are
predeclared by the language, or with defined types whose underlying
type is one of those predeclared types.
That is, the operator `<` can only be used with a predeclared type
such as `int` or `float64`, or a defined type whose underlying type is
one of those types.
Go does not permit using `<` with a composite type or with an
arbitrary defined type.

我们需要一种方法来编写一个只接受支持 `<` 的类型的约束。
为了做到这一点，我们观察到，除了我们稍后将讨论的两个例外，语言定义的所有算术、比较和逻辑运算符只能与语言预先声明的类型或定义的类型一起使用其基础类型是那些预先声明的类型之一。
也就是说，运算符 `<` 只能与预先声明的类型（例如 `int` 或 `float64`）一起使用，或者与基础类型是这些类型之一的已定义类型一起使用。 `Go` 不允许将 `<` 与复合类型或任意定义的类型一起使用。

This means that rather than try to write a constraint for `<`, we can
approach this the other way around: instead of saying which operators
a constraint should support, we can say which types a constraint
should accept.
We do this by defining a _type set_ for a constraint.

这意味着与其尝试为 `<` 编写约束，我们还可以反过来处理：我们可以说约束应该接受哪些类型，而不是说约束应该支持哪些运算符。我们通过为约束定义一个`type set`来做到这一点。

#### Type sets

Although we are primarily interested in defining the type set of
constraints, the most straightforward approach is to define the type
set of all types.
The type set of a constraint is then constructed out of the type sets
of its elements.
This may seem like a digression from the topic of using operators with
parameterized types, but we'll get there in the end.

尽管我们主要对定义约束的类型集感兴趣，但最直接的方法是定义所有类型的类型集。然后，一个约束的类型集是由其元素的类型集构造出来的。这似乎是对使用具有参数化类型的操作符这一主题的偏离，但我们最后会达到这个目的。

Every type has an associated type set.
The type set of a non-interface type `T` is simply the set `{T}`: a
set that contains just `T` itself.
The type set of an ordinary interface type is the set of all types
that declare all the methods of the interface.

每个类型都有一个相关的类型集。一个非接口类型T的类型集只是`{T}`的集合：一个只包含`T`本身的集合。一个普通接口类型的类型集是所有声明了该接口所有方法的类型的集合。

Note that the type set of an ordinary interface type is an infinite
set.
For any given type `T` and interface type `IT` it's easy to tell
whether `T` is in the type set of `IT` (by checking if all methods of
`IT` are declared by `T`), but there is no reasonable way to enumerate
all the types in the type set of `IT`.
The type `IT` is a member of its own type set, because an interface
inherently declares all of its own methods.
The type set of the empty interface `interface{}` is the set of all
possible types.

请注意，一个普通接口类型的类型集是一个无限的集合。对于任何给定的类型`T`和`inteface type IT`，很容易知道T是否在`IT`的类型集中（通过检查`IT`的所有方法是否由`T`声明），但是没有合理的方法来列举`IT`类型集中的所有类型。`IT`的类型是它自己的类型集的成员，因为一个接口本来就声明了它自己的所有方法。空接口`interface{}`的类型集是所有可能类型的集合。

It will be useful to construct the type set of an interface type by
looking at the elements of the interface.
This will produce the same result in a different way.
The elements of an interface can be either a method signature or an
embedded interface type.
Although a method signature is not a type, it's convenient to define a
type set for it: the set of all types that declare that method.
The type set of an embedded interface type `E` is simply that of `E`:
the set of all types that declare all the methods of `E`.

通过查看接口的元素来构造接口类型的类型集将很有用。

这将以不同的方式产生相同的结果。

接口的元素可以是方法签名或嵌入式接口类型。

虽然方法签名不是类型，但为它定义一个类型集很方便：声明该方法的所有类型的集合。

嵌入式接口类型 `E` 的类型集就是 `E` 的类型集：声明 `E` 的所有方法的所有类型的集合。

For any method signature `M`, the type set of `interface{ M }` is
the type of `M`: the set of all types that declare `M`.
For any method signatures `M1` and `M2`, the type set of `interface{
M1; M2 }` is set of all types that declare both `M1` and `M2`.
This is the intersection of the type set of `M1` and the type set of
`M2`.
To see this, observe that the type set of `M1` is the set of all types
with a method `M1`, and similarly for `M2`.
If we take the intersection of those two type sets, the result is the
set of all types that declare both `M1` and `M2`.
That is exactly the type set of `interface{ M1; M2 }`.

对于任何方法签名`M`，`interface{ M }`的类型集是M的类型：声明M的所有类型的集合。

对于任何方法签名`M1`和`M2`，`interface{ M1; M2 }`的类型集是同时声明`M1`和`M2`的所有类型的集合。这就是`M1`的类型集和`M2`的类型集的交集。

要看到这一点，请注意，`M1`的类型集是所有具有`M1`方法的类型的集合，同样，`M2`也是如此。

如果我们取这两个***类型集的交集***，结果就是同时声明`M1`和`M2`的所有类型的集合。这正是i`nterface{ M1; M2 }`的类型集。

The same applies to embedded interface types.
For any two interface types `E1` and `E2`, the type set of `interface{
E1; E2 }` is the intersection of the type sets of `E1` and `E2`.

这同样适用于嵌入式接口类型。对于任何两个接口类型`E1`和`E2`，`interface{ E1; E2 }`的类型集是`E1`和`E2`的类型集的交集。

Therefore, the type set of an interface type is the intersection of
the type sets of the element of the interface.

***因此，接口类型的类型集是接口元素的类型集的交集。***

#### Type sets of constraints

Now that we have described the type set of an interface type, we will
redefine what it means to satisfy the constraint.
Earlier we said that a type argument satisfies a constraint if it
implements the constraint.
Now we will say that a type argument satisfies a constraint if it is a
member of the constraint's type set.

现在我们已经描述了接口类型的类型集，我们将重新定义满足约束的含义。
之前我们说过，如果`type argument`实现了约束，那么它就满足约束。
现在我们说如果`type argument`是约束类型集的成员，则它满足约束。

For an ordinary interface type, one whose only elements are method
signatures and embedded ordinary interface types, the meaning is
exactly the same: the set of types that implement the interface type
is exactly the set of types that are in its type set.

对于一个普通的接口类型，其唯一的元素是方法签名和嵌入的普通接口类型，其含义完全相同：实现该接口类型的类型集正好是其类型集中的类型集。

We will now proceed to define additional elements that may appear in
an interface type that is used as a constraint, and define how those
additional elements can be used to further control the type set of the
constraint.

现在我们将继续定义可能出现在作为约束条件的接口类型中的额外元素，并定义如何使用这些额外元素来进一步控制约束条件的类型集。

#### Constraint elements

The elements of an ordinary interface type are method signatures and
embedded interface types.
We propose permitting three additional elements that may be used in an
interface type used as a constraint.
If any of these additional elements are used, the interface type may
not be used as an ordinary type, but may only be used as a constraint.

普通接口类型的元素是方法签名和嵌入式接口类型。我们建议允许在作为约束的接口类型中使用三个额外的元素。如果使用了这些额外的元素，该接口类型就不能作为普通类型使用，而只能作为约束使用。


##### Arbitrary type constraint element

The first new element is to simply permit listing any type, not just
an interface type.
For example: `type Integer interface{ int }`.
When a non-interface type `T` is listed as an element of a constraint,
its type set is simply `{T}`.
The type set of `int` is `{int}`.
Since the type set of a constraint is the intersection of the type
sets of all elements, the type set of `Integer` is also `{int}`.
This constraint `Integer` can be satisfied by any type that is a
member of the set `{int}`.
There is exactly one such type: `int`.

***第一个新元素是简单地允许列出任何类型，而不仅仅是一个接口类型。例如：`type Integer interface{ int }`。当一个非接口类型`T`被列为一个约束的元素时，它的类型集只是`{T}`。`int`的类型集是`{int}`。由于约束的类型集是所有元素的类型集的交集，所以`Integer`的类型集也是`{int}`。这个约束`Integer`可以被任何属于集合`{int}`成员的类型所满足。***

The type may be a type literal that refers to a type parameter (or
more than one), but it may not be a plain type parameter.

类型可以是指一个` type parameter`（或不止一个）的类型字面量，但它不可以是一个普通的` type parameter`。

```go
// EmbeddedParameter is INVALID.
type EmbeddedParameter[T any] interface {
	T // INVALID: may not list a plain type parameter
}
```

##### Approximation constraint element

Listing a single type is useless by itself.
For constraint satisfaction, we want to be able to say not just `int`,
but "any type whose underlying type is `int`".
Consider the `Smallest` example above.
We want it to work not just for slices of the predeclared ordered
types, but also for types defined by a program.
If a program uses `type MyString string`, the program can use the `<`
operator with values of type `MyString`.
It should be possible to instantiate `Smallest` with the type
`MyString`.

列出一个单一的类型本身是没有用的。为了满足约束条件，我们希望不仅能说`int`，而且能说 "任何底层类型为`int`的类型"。考虑一下上面的`Smallest`的例子。我们希望它不仅对预先声明的有序类型的片断起作用，而且对程序定义的类型也起作用。如果一个程序使用`MyString`类型的字符串，该程序可以使用`<`操作符与`MyString`类型的值, 并且可以用`MyString`类型来实例化`Smallest`。

To support this, the second new element we permit in a constraint is a
new syntactic construct: an approximation element, written as `~T`.
The type set of `~T` is the set of all types whose underlying type is
`T`.

***为了支持这一点，我们在约束中允许的第二个新元素是一个新的语法结构：一个近似元素，写成~T。~T的类型集是其基本类型为T的所有类型的集合。***

For example: `type AnyString interface{ ~string }`.
The type set of `~string`, and therefore the type set of `AnyString`,
is the set of all types whose underlying type is `string`.
That includes the type `MyString`; `MyString` used as a type argument
will satisfy the constraint `AnyString`.
 
例如: `ype AnyString interface{ ~string }` 类型集是 `～string`, 因此 AnyString 的类型集 的底层类型是 `string`, 这包括MyString类型；作为类型参数的MyString将满足AnyString的约束。

This new `~T` syntax will be the first use of `~` as a token in Go.

这个新的 `~T`语法将是 `Go` 中首次使用 `~` 作为标记。

Since `~T` means the set of all types whose underlying type is `T`, it
will be an error to use `~T` with a type `T` whose underlying type is
not itself.
Types whose underlying types are themselves are:



1. Type literals, such as `[]byte` or `struct{ f int }`.
2. Most predeclared types, such as `int` or `string` (but not
   `error`).

因为~T是指底层类型为T的所有类型的集合，所以对底层类型不是自己的类型T使用~T将是一个错误。底层类型为其自身的类型是：
1. 字面类型，例如: `[]byte` 或者  `struct {f int}`。
2. 大多数预先声明的类型，如int或string（但不包含error）。

Using `~T` is not permitted if `T` is a type parameter or if `T` is an
interface type.

***如果 `T` 是 `type parameters` 或 `T` 是 `interface type`，则不允许使用 `~T`。***

```go
type MyString string

// AnyString matches any type whose underlying type is string.
// This includes, among others, the type string itself, and
// the type MyString.
type AnyString interface {
	~string
}

// ApproximateMyString is INVALID.
type ApproximateMyString interface {
	~MyString // INVALID: underlying type of MyString is not MyString 注意这里
}

// ApproximateParameter is INVALID.
type ApproximateParameter[T any] interface {
	~T // INVALID: T is a type parameter 注意这里
}
```

##### Union constraint element

The third new element we permit in a constraint is also a new
syntactic construct: a union element, written as a series of
constraint elements separated by vertical bars (`|`).
For example: `int | float32` or `~int8 | ~int16 | ~int32 | ~int64`.
The type set of a union element is the union of the type sets of each
element in the sequence.
The elements listed in a union must all be different.
For example:

我们在约束中允许的第三个新元素也是一个新的语法结构：联合元素，写成一系列的约束元素，用竖条`|`隔开。例如：`int | float32 ` 或 `~int8 | ~int16 | ~int32 | ~int64`。一个联合元素的类型集是序列中每个元素的类型集的联合。在一个联合中列出的元素必须都是不同的。比如说

```go
// PredeclaredSignedInteger is a constraint that matches the
// five predeclared signed integer types.
type PredeclaredSignedInteger interface {
	int | int8 | int16 | int32 | int64
}
```

The type set of this union element is the set `{int, int8, int16,
int32, int64}`.
Since the union is the only element of `PredeclaredSignedInteger`,
that is also the type set of `PredeclaredSignedInteger`.
This constraint can be satisfied by any of those five types.

Here is an example using approximation elements:

这个 `union` 元素的类型集是集合 `{int, int8, int16, int32, int64}`。由于`int | int8 | int16 | int32 | int64` 是 `PredeclaredSignedInteger` 的唯一元素，因此它也是 `PredeclaredSignedInteger` 的类型集。这五种类型中的任何一种都可以满足此约束。
下面是一个例子：

```go
// SignedInteger is a constraint that matches any signed integer type.
type SignedInteger interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64
}
```

The type set of this constraint is the set of all types whose
underlying type is one of `int`, `int8`, `int16`, `int32`, or
`int64`.
Any of those types will satisfy this constraint.

The new constraint element syntax is

此约束的类型集是其基础类型为 int、int8、int16、int32 或 int64 之一的所有类型的集合。这些类型中的任何一种都将满足此约束。
新的约束元素语法是:

```t
InterfaceType  = "interface" "{" {(MethodSpec | InterfaceTypeName | ConstraintElem) ";" } "}" .
ConstraintElem = ConstraintTerm { "|" ConstraintTerm } .
ConstraintTerm = ["~"] Type .
```

#### Operations based on type sets

The purpose of type sets is to permit generic functions to use
operators, such as `<`, with values whose type is a type parameter.

类型集的目的是允许泛型函数对`type parameter`的值使用运算符，例如 `<`。

The rule is that a generic function may use a value whose type is a
type parameter in any way that is permitted by every member of the
type set of the parameter's constraint.
This applies to operators like '<' or '+' or other general operators.
For special purpose operators like `range` loops, we permit their use
if the type parameter has a structural constraint, as [defined
later](#Constraint-type-inference); the definition here is basically
that the constraint has a single underlying type.
If the function can be compiled successfully using each type in the
constraint's type set, or when applicable using the structural type,
then the use is permitted.

规则是泛型函数可以使用参数约束类型集中每个成员允许的任何方式使用其类型为` type parameter`的值。
这适用于“<”或“+”等运算符或其他通用运算符。
对于像`range loop`这样的特殊用途运算符，如果` type parameter`具有结构约束（如后文定义），我们允许使用它们；这里的定义基本上是约束具有单一的基础类型。
如果函数可以使用约束类型集中的每种类型成功编译，或者在适用时使用结构类型，则允许使用。

对于前面显示的最小的例子，我们可以使用这样的约束:

```go
package constraints

// Ordered is a type constraint that matches any ordered type.
// An ordered type is one that supports the <, <=, >, and >= operators.
type Ordered interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64 |
		~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
		~float32 | ~float64 |
		~string
}
```

In practice this constraint would likely be defined and exported in a
new standard library package, `constraints`, so that it could be used
by function and type definitions.

Given that constraint, we can write this function, now valid:

在实践中，这个约束可能会被定义并导出到一个新的标准库包，即约束，这样它就可以被函数和类型定义所使用。

通过上述理论，我们可以把写这样的一个对比函数:

```go 
// Smallest returns the smallest element in a slice.
// It panics if the slice is empty.
func Smallest[T constraints.Ordered](s []T) T {
	r := s[0] // panics if slice is empty
	for _, v := range s[1:] {
		if v < r {
			r = v
		}
	}
	return r
}
```

#### Comparable types in constraints

Earlier we mentioned that there are two exceptions to the rule that
operators may only be used with types that are predeclared by the
language.
The exceptions are `==` and `!=`, which are permitted for struct,
array, and interface types.
These are useful enough that we want to be able to write a constraint
that accepts any comparable type.

To do this we introduce a new predeclared type constraint:
`comparable`.
The type set of the `comparable` constraint is the set of all
comparable types.
This permits the use of `==` and `!=` with values of that type
parameter.

For example, this function may be instantiated with any comparable
type:

前面我们提到过运算符只能与语言预先声明的类型一起使用的规则有两个例外。
例外情况是 `==` 和 `!=`，它们允许用于`struct`、`array`和`interace types`。
这非常有用，以至于我们希望能够编写一个接受任何可比较类型的约束。
为此，我们引入了一个新的预声明类型约束：`comparable`。
可比较约束的类型集是所有可比较类型的集合。
这允许将 `== `和 `!=` 与该`type parameter`的值一起使用。
例如，这个函数可以用任何可比较的类型实例化：

```go
// Index returns the index of x in s, or -1 if not found.
func Index[T comparable](s []T, x T) int {
	for i, v := range s {
		// v and x are type T, which has the comparable
		// constraint, so we can use == here.
		if v == x {
			return i
		}
	}
	return -1
}
```

Since `comparable` is a constraint, it can be embedded in another
interface type used as a constraint.

由于`comparable`是一种约束，它可以被嵌入到另一个用作约束的接口类型中。

```go
// ComparableHasher is a type constraint that matches all
// comparable types with a Hash method.
type ComparableHasher interface {
	comparable
	Hash() uintptr
}
```

The constraint `ComparableHasher` is implemented by any type that is
comparable and also has a `Hash() uintptr` method.
A generic function that uses `ComparableHasher` as a constraint can
compare values of that type and can call the `Hash` method.

`ComparableHasher`约束是由任何可比较的类型实现的，并且也有一个`Hash() uintptr`方法。
一个使用`ComparableHasher`作为约束的通用函数可以比较该类型的值，并且可以调用`Hash`方法。

It's possible to use `comparable` to produce a constraint that can not
be satisfied by any type.
See also the [discussion of empty type sets below](#Empty-type-sets).

有可能使用`comparable`来产生一个不能被任何类型满足的约束。也请看下面关于空类型集的[讨论](#Empty-type-sets)。

```
// ImpossibleConstraint is a type constraint that no type can satisfy,
// because slice types are not comparable.
type ImpossibleConstraint interface {
	comparable
	[]int
}
```

### Mutually referencing type parameters 

Within a single type parameter list, constraints may refer to any of
the other type parameters, even ones that are declared later in the
same list.
(The scope of a type parameter starts at the beginning of the type
parameter list and extends to the end of the enclosing function or
type declaration.)

在一个单一的` type parameter`列表中，约束条件可以引用任何其他的` type parameter`，甚至是在同一列表中稍后声明的参数。
(一个` type parameter`的范围从` type parameter`列表的开头开始，一直延伸到包围函数或类型声明的结尾）。

For example, consider a generic graph package that contains generic
algorithms that work with graphs.
The algorithms use two types, `Node` and `Edge`.
`Node` is expected to have a method `Edges() []Edge`.
`Edge` is expected to have a method `Nodes() (Node, Node)`.
A graph can be represented as a `[]Node`.

例如，考虑一个通用图包，它包含了处理图的通用算法。这些算法使用两种类型：`Node`和`Edge`。`Node`应该有一个方法`Edges() []Edge`。`Edge`应该有一个方法`Nodes() (Node, Node)`。一个图可以表示为一个`[]Node`。

This simple representation is enough to implement graph algorithms
like finding the shortest path.

这种简单的表示足以实现图形算法，例如寻找最短路径。

```go
package graph

// NodeConstraint is the type constraint for graph nodes:
// they must have an Edges method that returns the Edge's
// that connect to this Node.
type NodeConstraint[Edge any] interface {
	Edges() []Edge
}

// EdgeConstraint is the type constraint for graph edges:
// they must have a Nodes method that returns the two Nodes
// that this edge connects.
type EdgeConstraint[Node any] interface {
	Nodes() (from, to Node)
}

// Graph is a graph composed of nodes and edges.
type Graph[Node NodeConstraint[Edge], Edge EdgeConstraint[Node]] struct { ... }

// New returns a new graph given a list of nodes.
func New[Node NodeConstraint[Edge], Edge EdgeConstraint[Node]] (nodes []Node) *Graph[Node, Edge] {
	...
}

// ShortestPath returns the shortest path between two nodes,
// as a list of edges.
func (g *Graph[Node, Edge]) ShortestPath(from, to Node) []Edge { ... }
```

There are a lot of type arguments and instantiations here.
In the constraint on `Node` in `Graph`, the `Edge` being passed to the
type constraint `NodeConstraint` is the second type parameter of
`Graph`.
This instantiates `NodeConstraint` with the type parameter `Edge`, so
we see that `Node` must have a method `Edges` that returns a slice of
`Edge`, which is what we want.
The same applies to the constraint on `Edge`, and the same type
parameters and constraints are repeated for the function `New`.
We aren't claiming that this is simple, but we are claiming that it is
possible.

这里有很多`type argument`被实例化在这。在 `Graph` 对 `Node` 的约束中，传递给类型约束 `NodeConstraint` 的 `Edge` 是 `Graph` 的第二个` type parameter`。
这用` type parameter` `Edge` 实例化了 `NodeConstraint`，因此我们看到 `Node` 必须有一个返回 `Edge` 切片的方法 `Edges`，这正是我们想要的。这同样适用于对 `Edge` 的约束，并且对函数 `New` 重复相同类型的参数和约束。我们并不是说这很简单，而是说这是可能的。

It's worth noting that while at first glance this may look like a
typical use of interface types, `Node` and `Edge` are non-interface
types with specific methods.
In order to use `graph.Graph`, the type arguments used for `Node` and
`Edge` have to define methods that follow a certain pattern, but they
don't have to actually use interface types to do so.
In particular, the methods do not return interface types.

For example, consider these type definitions in some other package:

值得注意的是，虽然乍一看这可能看起来像是接口类型的典型用法，但 `Node` 和 `Edge` 是具有特定方法的非接口类型。
为了使用 `graph.Graph`，用于 `Node` 和 `Edge` 的`type argument`必须定义遵循特定模式的方法，但它们不必实际使用接口类型来这样做。特别是，这些方法不返回接口类型。

例如，考虑其他包中的这些类型定义：

```go
// Vertex is a node in a graph.
type Vertex struct { ... }

// Edges returns the edges connected to v.
func (v *Vertex) Edges() []*FromTo { ... }

// FromTo is an edge in a graph.
type FromTo struct { ... }

// Nodes returns the nodes that ft connects.
func (ft *FromTo) Nodes() (*Vertex, *Vertex) { ... }
```

There are no interface types here, but we can instantiate
`graph.Graph` using the type arguments `*Vertex` and `*FromTo`.

这里没有接口类型，但我们可以使用类型参数`*Vertex`和`*FromTo`来实例化`Graph.Graph`。

```go
var g = graph.New[*Vertex, *FromTo]([]*Vertex{ ... })
```

`*Vertex` and `*FromTo` are not interface types, but when used
together they define methods that implement the constraints of
`graph.Graph`.
Note that we couldn't pass plain `Vertex` or `FromTo` to `graph.New`,
since `Vertex` and `FromTo` do not implement the constraints.
The `Edges` and `Nodes` methods are defined on the pointer types
`*Vertex` and `*FromTo`; the types `Vertex` and `FromTo` do not have
any methods.

`*Vertex` 和 `*FromTo` 不是接口类型，但一起使用时它们定义了实现 `graph.Graph` 约束的方法。请注意，我们不能将普通的 `Vertex` 或 `FromTo` 传递给 `graph.New`，因为 `Vertex` 和 `FromTo` 没有实现约束。 `Edges` 和 `Nodes` 方法定义在指针类型` *Vertex` 和 `*FromTo` 上； `Vertex` 和 `FromTo` 类型没有任何方法。

When we use a generic interface type as a constraint, we first
instantiate the type with the type argument(s) supplied in the type
parameter list, and then compare the corresponding type argument
against the instantiated constraint.

当我们使用泛型接口类型作为约束时，我们首先使用`type parameter list`列表中提供的`type argument(s)`实例化该类型，然后将相应的类型参数与实例化的约束进行比较

In this example, the `Node` type argument to `graph.New` has a
constraint `NodeConstraint[Edge]`.
When we call `graph.New` with a `Node` type argument of `*Vertex` and
an `Edge` type argument of `*FromTo`, in order to check the constraint
on `Node` the compiler instantiates `NodeConstraint` with the type
argument `*FromTo`.
That produces an instantiated constraint, in this case a requirement
that `Node` have a method `Edges() []*FromTo`, and the compiler
verifies that `*Vertex` satisfies that constraint.

在这个例子中，`graph.New`的`Node`约束是`NodeConstraint[Edge]`。
当我们用`*Vertex`和`*FromTo`调用`graph.New`时，为了检查`Node`的约束，编译器用`*FromTo`实例化`NodeConstraint`。
这就产生了一个实例化的约束，在这种情况下，要求`Node`有一个`Edges()[]*FromTo`的方法，编译器会验证`*Vertex`是否满足该约束。

Although `Node` and `Edge` do not have to be instantiated with
interface types, it is also OK to use interface types if you like.

虽然Node和Edge不一定要用接口类型来实例化，但如果你喜欢，使用接口类型也是可以的。

```go
type NodeInterface interface { Edges() []EdgeInterface }
type EdgeInterface interface { Nodes() (NodeInterface, NodeInterface) }
```

We could instantiate `graph.Graph` with the types `NodeInterface` and
`EdgeInterface`, since they implement the type constraints.
There isn't much reason to instantiate a type this way, but it is
permitted.

我们可以用 `NodeInterface` 和 `EdgeInterface` 类型实例化 `graph.Graph`，因为它们实现了类型约束。没有太多理由以这种方式实例化类型，但它是被允许的。

This ability for type parameters to refer to other type parameters
illustrates an important point: it should be a requirement for any
attempt to add generics to Go that it be possible to instantiate
generic code with multiple type arguments that refer to each other in
ways that the compiler can check.

这种`type parameters`引用其他`type parameters`的能力说明了一个重要的观点：任何向 `Go` 中添加泛型特性的尝试，都应该满足能够使用多个`type arguments`实例化泛型代码，这些`type arguments` 每个都可以被编译器检查。(基调 0.0 )

### Type inference

In many cases we can use type inference to avoid having to explicitly
write out some or all of the type arguments.
We can use _function argument type inference_ for a function call to
deduce type arguments from the types of the non-type arguments.
We can use _constraint type inference_ to deduce unknown type arguments
from known type arguments.

In the examples above, when instantiating a generic function or type,
we always specified type arguments for all the type parameters.
We also permit specifying just some of the type arguments, or omitting
the type arguments entirely, when the missing type arguments can be
inferred.
When only some type arguments are passed, they are the arguments for
the first type parameters in the list.

For example, a function like this:


在多数情况下，我们可以使用类型推断来避免必须显式写出部分或全部`type arguments`。

我们可以` _function argument type inference_`，从`non-type arguments`的类型中推断出`type arguments`。

我们可以使用` _constraint type inference_ `, 从已知`type arguments`中推导出未知`type arguments`。

在上面的示例中，在实例化泛型函数或类型时，我们总是为所有`type parmameters`指定`type arguments`。当可以推断缺少的`type arguments`时，我们还允许仅指定一些`type arguments`，或完全省略`type arguments`。

当只传递一些`type arguments`时，它们是列表中第一个`type parameters`的参数。

例如，像这样的函数：

```go
func Map[F, T any](s []F, f func(F) T) []T { ... }
```

can be called in these ways.  (We'll explain below how type inference
works in detail; this example is to show how an incomplete list of
type arguments is handled.)

可以通过这些方式调用。(我们将在下面详细解释类型推理是如何工作的；这个例子是为了说明如何处理一个不完整的`type arguments`列表)。

```go
	var s []int
	f := func(i int) int64 { return int64(i) }
	var r []int64
	// Specify both type arguments explicitly.
	r = Map[int, int64](s, f)
	// Specify just the first type argument, for F,
	// and let T be inferred.
	r = Map[int](s, f)
	// Don't specify any type arguments, and let both be inferred.
	r = Map(s, f)
```

If a generic function or type is used without specifying all the type
arguments, it is an error if any of the unspecified type arguments
cannot be inferred.

如果在没有指定所有`type arguments`的情况下使用泛型函数或类型，如果任何未指定的`type arguments`不能被推断出来，那就是一个错误。

(Note: type inference is a convenience feature.
Although we think it is an important feature, it does not add any
functionality to the design, only convenience in using it.
It would be possible to omit it from the initial implementation, and
see whether it seems to be needed.
That said, this feature doesn't require additional syntax, and
produces more readable code.)

(注意：类型推断是一个方便的功能。尽管我们认为它是一个重要的特性，但它并没有为设计增加任何功能，只是在使用时提供了便利。可以在最初的实现中省略它，看看是否看起来需要它。也就是说，这个功能不需要额外的语法，而且能产生更多可读的代码）。

#### Type unification

Type inference is based on _type unification_.
Type unification applies to two types, either or both of which may be
or contain type parameters.

类型推断基于 `_type unification_`。
`Type unification`适用于两类, 其中一个或两个都`可能是`或`包含``type parmameters`。

Type unification works by comparing the structure of the types.
Their structure disregarding type parameters must be identical, and
types other than type parameters must be equivalent.
A type parameter in one type may match any complete subtype in the
other type.

`Type unification`是通过比较类型的结构来实现的。
不考虑`type parmameters`的它们的结构必须是相同的，除`type parmameters`以外的类型必须是等价的。
一个类型中的`type parmameters`可以匹配另一个类型中的任何完整的子类型。

If the structure differs, or types other than type parameters are not
equivalent, then type unification fails.
A successful type unification provides a list of associations of type
parameters with other types (which may themselves be or contain type
parameters).

如果结构不同，或者`type parmameters`以外的类型不等价，则`Type unification`失败。成功的`Type unification`提供了类型参数与其他类型（它们本身可能是或包含`type parmameters`）的关联列表。

For type unification, two types that don't contain any type parameters
are equivalent if they are
[identical](https://golang.org/ref/spec#Type_identity), or if they are
channel types that are identical ignoring channel direction, or if
their underlying types are equivalent.
It's OK to permit types to not be identical during type inference,
because we will still check the constraints if inference succeeds, and
we will still check that the function arguments are assignable to the
inferred types.

对于`Type unification`，如果两个不包含任何`type parmameters`的类型是[相同的](https://golang.org/ref/spec#Type_identity)，或者如果它们是忽略`channel`方向的`channel type`是相同的，或者如果它们的底层类型是等价的。在类型推理过程中，允许类型不完全相同是可以的，因为如果推理成功，我们仍然会检查约束条件，而且我们仍然会检查函数参数是否可以分配给推理的类型。

For example, if `T1` and `T2` are type parameters, `[]map[int]bool`
can be unified with any of the following:

例如，如果 `T1` 和 `T2` 是 `type parameters`，则 `[]map[int]bool` 可以与以下任何一个统一：

  * `[]map[int]bool`
  * `T1` (`T1` matches `[]map[int]bool`)
  * `[]T1` (`T1` matches `map[int]bool`)
  * `[]map[T1]T2` (`T1` matches `int`, `T2` matches `bool`)

(This is not an exclusive list, there are other possible successful
unifications.)

(当然应该还有跟多的例子可以匹配上述情况)

举个反面的栗子，`[]map[int]bool` 不能匹配到下面情况
  * `int`
  * `struct{}`
  * `[]struct{}`
  * `[]map[T1]string`

(This list is of course also not exclusive; there are an infinite
number of types that cannot be successfully unified.)

(当然应该还有跟多的例子可以匹配上述情况)

In general we can also have type parameters on both sides, so in some
cases we might associate `T1` with, for example, `T2`, or `[]T2`.

通常我们也可以在两侧都有`type parmameters`，因此在某些情况下我们可能会将 T1 与例如 T2 或 []T2 相关联。

#### Function argument type inference

Function argument type inference is used with a function call to infer
type arguments from non-type arguments.
Function argument type inference is not used when a type is
instantiated, and it is not used when a function is instantiated but
not called.

To see how it works, let's go back to [the example](#Type-parameters)
of a call to the simple `Print` function:

函数参数类型推断与函数调用一起使用，从`non-type argument`中推断出`type argument`。当一个类型被实例化时，不使用函数参数类型推理，当一个函数被实例化但没有被调用时，也不使用。

为了看看它是如何工作的，让我们回到调用简单的Print函数的例子中:

```go
	Print[int]([]int{1, 2, 3})
```

The type argument `int` in this function call can be inferred from the
type of the non-type argument.

这个函数调用中的 `type argument` `int`可以从`non-type argument`的类型中推断出来。

The only type arguments that can be inferred are those that are used
for the types of the function's (non-type) input parameters.
If there are some type parameters that are used only for the
function's result parameter types, or only in the body of the
function, then those type arguments cannot be inferred using function
argument type inference.

唯一可以推断的`type argument`是那些用于函数输入参数的类型。
如果有一些`type parameters`只用于函数的结果参数类型，或者只在函数的主体中使用，那么这些`type argument`不能用`function argument type inference`。

To infer function type arguments, we unify the types of the function
call arguments with the types of the function's non-type parameters.
On the caller side we have the list of types of the actual (non-type)
arguments, which for the `Print` example is simply `[]int`.
On the function side is the list of the types of the function's
non-type parameters, which for `Print` is `[]T`.
In the lists, we discard respective arguments for which the function
side does not use a type parameter.
We must then apply type unification to the remaining argument types.

为了推断函数`type argument`，我们将函数调用参数的类型与函数的`non-type parameters`的类型统一起来。
在调用方，我们有实际（非类型）参数的类型列表，对于Print的例子，它只是`[]int`。
在函数一侧是函数的`non-type parameters`的类型列表，对于`Print`来说是`[]T`。在这些列表中，我们放弃了函数侧没有使用类型参数的各个参数。
然后我们必须对剩余的参数类型应用类型统一。

Function argument type inference is a two-pass algorithm.
In the first pass, we ignore untyped constants on the caller side and
their corresponding types in the function definition.
We use two passes so that in some cases later arguments can determine
the type of an untyped constant.

函数参数类型推断是通过 `two-pass algorithm`来实现的。在第一遍中，我们忽略调用方的无类型常量及其在函数定义中的相应类型。我们使用两次传递，以便在某些情况下后面的参数可以确定无类型常量的类型。

We unify corresponding types in the lists.
This will give us an association of type parameters on the function
side to types on the caller side.
If the same type parameter appears more than once on the function
side, it will match multiple argument types on the caller side.
If those caller types are not equivalent, we report an error.

我们在列表中统一了相应的类型。这将给我们提供一个函数侧的`type parameters`与调用者侧的类型的关联。如果同一个类型的参数在函数端出现多次，它将与调用方的多个参数类型相匹配。如果这些调用方的类型不对等，我们会报告一个错误。

After the first pass, we check any untyped constants on the caller
side.
If there are no untyped constants, or if the type parameters in the
corresponding function types have matched other input types, then
type unification is complete.

在第一遍之后，我们检查调用方的任何未类型化的常量。如果没有未定型的常量，或者相应的函数类型中的`type parameters`已经与其他输入类型相匹配，那么类型统一就完成了。

Otherwise, for the second pass, for any untyped constants whose
corresponding function types are not yet set, we determine the default
type of the untyped constant in [the usual
way](https://golang.org/ref/spec#Constants).
Then we unify the remaining types again, this time with no untyped
constants.

否则，在第二遍中，对于任何未定型的常量，其对应的函数类型尚未设置，我们以[通常的方式](https://golang.org/ref/spec#Constants)确定未定型常量的默认类型。然后，我们再次统一剩余的类型，这一次没有未定型常量。

When constraint type inference is possible, as described below, it is
applied between the two passes.

当约束类型推断是可能的时，如下所述，它被应用在两个过程之间。

看看栗子

```go
	s1 := []int{1, 2, 3}
	Print(s1)
```

we compare `[]int` with `[]T`, match `T` with `int`, and we are done.
The single type parameter `T` is `int`, so we infer that the call to
`Print` is really a call to `Print[int]`.

我们将 `[]int` 与 `[]T` 进行比较，将 `T `与 ` int` 进行匹配，然后我们就完成了。单个`type parameters` `T` 是 `int`，因此我们推断对 `Print` 的调用实际上是对` Print[int]` 的调用。

For a more complex example, consider

来点复杂的栗子，看看下面的代码

```go
// Map calls the function f on every element of the slice s,
// returning a new slice of the results.
func Map[F, T any](s []F, f func(F) T) []T {
	r := make([]T, len(s))
	for i, v := range s {
		r[i] = f(v)
	}
	return r
}
```

The two type parameters `F` and `T` are both used for input
parameters, so function argument type inference is possible.
In the call

两个`type parameters` `F` 和 `T` 都用于输入参数，因此函数参数类型推断是可能的。看看下面的函数调用

```go
	strs := Map([]int{1, 2, 3}, strconv.Itoa)
```

we unify `[]int` with `[]F`, matching `F` with `int`.
We unify the type of `strconv.Itoa`, which is `func(int) string`,
with `func(F) T`, matching `F` with `int` and `T` with `string`.
The type parameter `F` is matched twice, both times with `int`.
Unification succeeds, so the call written as `Map` is a call of
`Map[int, string]`.

我们统一 `[]int` 和 `[]F`，匹配 `F` 和` int`。我们将`strconv.Itoa`的类型统一为`func(int) string`，与`func(F) T`匹配，`F`匹配`int`，`T`匹配`string`。`type parameters`` F` 被匹配了两次，两次都是` int`。统一成功，所以写成`Map`的调用就是对`Map[int, string]`的调用。

To see the untyped constant rule in effect, consider:

要查看有效的无类型常量规则，请考虑：

```go
// NewPair returns a pair of values of the same type.
func NewPair[F any](f1, f2 F) *Pair[F] { ... }
```

In the call `NewPair(1, 2)` both arguments are untyped constants, so
both are ignored in the first pass.
There is nothing to unify.
We still have two untyped constants after the first pass.
Both are set to their default type, `int`.
The second run of the type unification pass unifies `F` with
`int`, so the final call is `NewPair[int](1, 2)`.

在调用 `NewPair(1, 2)` 时，两个参数都是无类型常量，因此在第一遍中都将被忽略。没有什么可以统一的。第一次通过后，我们仍然有两个无类型常量。两者都设置为其默认类型 `int`。类型统一过程的第二次运行将 `F` 与 `int` 统一，因此最终调用是 `NewPair[int](1, 2)`。

In the call `NewPair(1, int64(2))` the first argument is an untyped
constant, so we ignore it in the first pass.
We then unify `int64` with `F`.
At this point the type parameter corresponding to the untyped constant
is fully determined, so the final call is `NewPair[int64](1,
int64(2))`.

在 `NewPair(1, int64(2))` 调用中，第一个参数是一个无类型常量，所以我们在第一遍中忽略它。然后我们将`int64`和`F`统一起来，此时无类型常量对应的`type parameters`就完全确定了，所以最后调用的是`NewPair[int64](1, int64(2))`。

In the call `NewPair(1, 2.5)` both arguments are untyped constants,
so we move on the second pass.
This time we set the first constant to `int` and the second to
`float64`.
We then try to unify `F` with both `int` and `float64`, so unification
fails, and we report a compilation error.

在 `NewPair(1, 2.5)` 调用中，两个参数都是无类型常量，所以我们继续第二遍。这次我们将第一个常量设置为 `int`，将第二个常量设置为` float64`。然后我们尝试将 `F` 与 `int` 和` float64` 统一，因此统一失败，并报告编译错误。

As mentioned earlier, function argument type inference is done without
regard to constraints.
First we use function argument type inference to determine type
arguments to use for the function, and then, if that succeeds, we
check whether those type arguments implement the constraints (if
any).

Note that after successful function argument type inference, the
compiler must still check that the arguments can be assigned to the
parameters, as for any function call.


如前所述，函数参数类型推理是在不考虑约束条件的情况下进行的。首先，我们使用函数参数类型推理来确定函数要使用的`type parameters`，然后，如果成功了，我们检查这些`type parameters`是否实现了约束条件（如果有的话）。
注意，在成功的函数参数类型推理之后，编译器仍然必须检查参数是否可以被分配给参数，就像任何函数调用一样。

#### Constraint type inference

Constraint type inference permits inferring a type argument from
another type argument, based on type parameter constraints.
Constraint type inference is useful when a function wants to have a
type name for an element of some other type parameter, or when a
function wants to apply a constraint to a type that is based on some
other type parameter.

约束类型推断允许根据`type parameter`约束从另一个`type argument`推断`type argument`。
当一个函数想要为某个其他`type parameter`的元素使用类型名称，或者当一个函数想要将约束应用于基于某个其他`type parameter`的类型时，约束类型推断很有用。

Constraint type inference can only infer types if some type parameter
has a constraint that has a type set with exactly one type in it, or a
type set for which the underlying type of every type in the type set
is the same type.

约束类型推理只有在某些`type parameter`
有一个约束，该约束有一个类型集，其中正好有一个类型，或者一个
类型集中每个类型的基本类型都是同一类型。
是同一类型。

The two cases are slightly different, as in the first case, in which
the type set has exactly one type, the single type need not be its own
underlying type.
Either way, the single type is called a _structural type_, and the
constraint is called a _structural constraint_.
The structural type describes the required structure of the type
parameter.
A structural constraint may also define methods, but the methods are
ignored by constraint type inference.
For constraint type inference to be useful, the structural type will
normally be defined using one or more type parameters.

这两种情况略有不同，在第一种情况下，类型集正好有一个类型，这个单一类型不需要是它自己的底层类型。
无论哪种情况，单一类型被称为结构类型，而约束被称为结构约束。结构类型描述了类型参数的必要结构。
结构性约束也可以定义方法，但是这些方法会被约束类型推理所忽略。
为了使约束类型推理有用，结构类型通常将使用一个或多个类型参数来定义。

Constraint type inference is only tried if there is at least one type
parameter whose type argument is not yet known.

仅当存在至少一个其类型参数未知的类型参数时，才会尝试约束类型推断。

While the algorithm we describe here may seem complex, for typical
concrete examples it is straightforward to see what constraint type
inference will deduce.
The description of the algorithm is followed by a couple of examples.

虽然我们在此描述的算法可能看起来很复杂，但对于典型的具体示例，很容易看出约束类型推断将推导出什么。算法的描述之后是几个例子。

We start by creating a mapping from type parameters to type arguments.
We initialize the mapping with all type parameters whose type
arguments are already known, if any.

我们首先创建一个从类型参数到类型参数的映射。我们用已经知道类型参数的`type arguments`来初始化该映射，如果有的话。

For each type parameter with a structural constraint, we unify the
type parameter with the structural type.
This will have the effect of associating the type parameter with its
constraint.
We add the result into the mapping we are maintaining.
If unification finds any associations of type parameters, we add those
to the mapping as well.
When we find multiple associations of any one type parameter, we unify
each such association to produce a single mapping entry.
If a type parameter is associated directly with another type
parameter, meaning that they must both be matched with the same type,
we unify the associations of each parameter together.
If any of these various unifications fail, then constraint type
inference fails.

对于每个具有结构约束的`type parameter`，我们将类型参数与结构类型统一起来。
这样可以讲类型参数与约束相关联。我们将结果添加到我们正在维护的映射中。如果统一发现类型参数的任何关联，我们也将它们添加到映射中。
当我们找到任何一种类型参数的多个关联时，我们统一每个这样的关联以生成一个映射条目。如果一个类型参数直接与另一个类型参数相关联，这意味着它们必须与同一类型匹配，我们将每个参数的关联统一在一起。如果这些不同的统一中的任何一个失败，则约束类型推断失败。

After merging all type parameters with structural constraints, we have
a mapping of various type parameters to types (which may be or contain
other type parameters).

在将所有类型参数与结构约束合并后，我们得到了各种类型参数到类型（可能是或包含其他类型参数）的映射。

We continue by looking for a type parameter `T` that is mapped to a
fully known type argument `A`, one that does not contain any type
parameters.

我们继续寻找映射到完全已知`type argument` `A` 的`type parameter` `T`，该类型参数不包含任何类型参数。

Anywhere that `T` appears in a type argument in the mapping, we
replace `T` with `A`.

在映射中 T 出现在` type argument`中的地方，我们用 `A` 替换 `T`。

We repeat this process until we have replaced every type parameter.

我们重复这个过程，直到我们替换了每个`type parameter`。

When constraint type inference is possible, type inference proceeds as
followed:



* Build the mapping using known type arguments.
* Apply constraint type inference.
* Apply function type inference using typed arguments.
* Apply constraint type inference again.
* Apply function type inference using the default types of any
  remaining untyped arguments.
* Apply constraint type inference again.

当可以进行约束类型推断时，类型推断如下进行：
* 使用已知`type arguments`构建映射。
* 使用约束类型推断。
* 用 `type arguments` 来进行 `function type inference`
* 使用约束类型推断。
* 将剩下未知类型使用默认的方式来进行函数类型推断
* 使用约束类型推断。

##### Element constraint example

For an example of where constraint type inference is useful, let's
consider a function that takes a defined type that is a slice of
numbers, and returns an instance of that same defined type in which
each number is doubled.

It's easy to write a function similar to this if we ignore the
[defined type](https://golang.org/ref/spec#Type_definitions)
requirement.

对于约束类型推断有用的示例，让我们考虑一个函数，该函数采用作为数字切片的定义类型，并返回相同定义类型的实例，其中每个数字都加倍。
如果我们忽略[defined type](https://golang.org/ref/spec#Type_definitions)要求，就很容易编写类似这样的函数。

```go
// Double returns a new slice that contains all the elements of s, doubled.
func Double[E constraints.Integer](s []E) []E {
	r := make([]E, len(s))
	for i, v := range s {
		r[i] = v + v
	}
	return r
}
```

However, with that definition, if we call the function with a defined
slice type, the result will not be that defined type.

但是，根据该定义，如果我们使用定义的切片类型调用函数，结果将不是该定义的类型。

```go
// MySlice is a slice of ints.
type MySlice []int

// The type of V1 will be []int, not MySlice.
// Here we are using function argument type inference,
// but not constraint type inference.
var V1 = Double(MySlice{1})
```

We can do what we want by introducing a new type parameter.

我们可以通过引入一个新的类型参数来做我们想做的事情。

```go
// DoubleDefined returns a new slice that contains the elements of s,
// doubled, and also has the same type as s.
func DoubleDefined[S ~[]E, E constraints.Integer](s S) S {
	// Note that here we pass S to make, where above we passed []E.
	r := make(S, len(s))
	for i, v := range s {
		r[i] = v + v
	}
	return r
}
```

Now if we use explicit type arguments, we can get the right type.

现在，如果我们使用显式类型参数，我们可以获得正确的类型。

```go
// The type of V2 will be MySlice.
var V2 = DoubleDefined[MySlice, int](MySlice{1})
```

Function argument type inference by itself is not enough to infer the
type arguments here, because the type parameter E is not used for any
input parameter.
But a combination of function argument type inference and constraint
type inference works.

函数参数类型推断本身不足以推断此处的类型参数，因为类型参数 E 不用于任何输入参数。
但是函数参数类型推断和约束类型推断的组合有效。

```go
// The type of V3 will be MySlice.
var V3 = DoubleDefined(MySlice{1})
```

First we apply function argument type inference.
We see that the type of the argument is `MySlice`.
Function argument type inference matches the type parameter `S` with
`MySlice`.

We then move on to constraint type inference.
We know one type argument, `S`.
We see that the type argument `S` has a structural type constraint.

We create a mapping of known type arguments:

首先我们应用函数参数类型推断。我们看到参数的类型是 `MySlice`。函数参数类型推断将`type parameter` `S` 与 `MySlice` 相匹配。
然后我们继续进行约束类型推断。我们知道一个`type argument` `S`。我们看到`type argument` `S` 具有结构类型约束。
我们创建已知类型参数的映射：

```go
{S -> MySlice}
```

We then unify each type parameter with a structural constraint with
the single type in that constraint's type set.
In this case the structural constraint is `~[]E` which has the
structural type `[]E`, so we unify `S` with `[]E`.
Since we already have a mapping for `S`, we then unify `[]E` with
`MySlice`.
As `MySlice` is defined as `[]int`, that associates `E` with `int`.
We now have:

然后，我们将每个`type parameter`与结构约束和该约束的类型集中的单一类型统一起来。
在这种情况下，结构约束是` ~[]E`，其结构类型为 `[]E`，因此我们将 `S `与 `[]E` 统一。
因为我们已经有了 `S `的映射，所以我们将 `[]E` 与 `MySlice` 统一起来。由于 `MySlice` 被定义为 `[]int`，
因此将 `E` 与 `int` 相关联。

我们现在有：

```go
{S -> MySlice, E -> int}
```

We then substitute `E` with `int`, which changes nothing, and we are
done.
The type arguments for this call to `DoubleDefined` are `[MySlice,
int]`.

This example shows how we can use constraint type inference to set a
type name for an element of some other type parameter.
In this case we can name the element type of `S` as `E`, and we can
then apply further constraints to `E`, in this case requiring that it
be a number.


然后我们用 `int` 替换 `E`，这没有任何改变，我们就完成了。
调用 `DoubleDefined` 的`type arguments`是 `[MySlice, int]`。
这个例子展示了我们如何使用约束类型推断来为某个其他类型参数的元素设置类型名称。
在这种情况下，我们可以将 `S` 的元素类型命名为 `E`，然后我们可以对 `E` 应用进一步的约束，在这种情况下要求它是一个数字。

##### Pointer method example

Consider this example of a function that expects a type `T` that has a
`Set(string)` method that initializes a value based on a string.

考虑这个例子，一个函数期望有一个类型`T`，它有一个`Set(string)`方法，可以根据字符串初始化一个值。

```go
// Setter is a type constraint that requires that the type
// implement a Set method that sets the value from a string.
type Setter interface {
	Set(string)
}

// FromStrings takes a slice of strings and returns a slice of T,
// calling the Set method to set each returned value.
//
// Note that because T is only used for a result parameter,
// function argument type inference does not work when calling
// this function.
func FromStrings[T Setter](s []string) []T {
	result := make([]T, len(s))
	for i, v := range s {
		result[i].Set(v)
	}
	return result
}
```

Now let's see some calling code (this example is invalid).
现在让我们看一些调用代码（这个例子是不可用的）。
```go
// Settable is an integer type that can be set from a string.
type Settable int

// Set sets the value of *p from a string.
func (p *Settable) Set(s string) {
	i, _ := strconv.Atoi(s) // real code should not ignore the error
	*p = Settable(i)
}

func F() {
	// INVALID
	nums := FromStrings[Settable]([]string{"1", "2"})
	// Here we want nums to be []Settable{1, 2}.
	...
}
```

The goal is to use `FromStrings` to get a slice of type `[]Settable`.
Unfortunately, this example is not valid and will not compile.

目标是使用 `FromStrings` 获取 `[]Settable` 类型的切片。
不幸的是，这个例子是无法通过编译器的。

The problem is that `FromStrings` requires a type that has a
`Set(string)` method.
The function `F` is trying to instantiate `FromStrings` with
`Settable`, but `Settable` does not have a `Set` method.
The type that has a `Set` method is `*Settable`.

问题是 `FromStrings` 需要一个具有 `Set(string)` 方法的类型。
函数 `F` 试图用 `Settable` 实例化 `FromStrings`，但 `Settable` 没有 `Set` 方法。具有 `Set` 方法的类型是 `*Settable`。

So let's rewrite `F` to use `*Settable` instead.

重来：

```go
func F() {
	// Compiles but does not work as desired.
	// This will panic at run time when calling the Set method.
	nums := FromStrings[*Settable]([]string{"1", "2"})
	...
}
```

This compiles but unfortunately it will panic at run time.
The problem is that `FromStrings` creates a slice of type `[]T`.
When instantiated with `*Settable`, that means a slice of type
`[]*Settable`.

这可以编译，但不幸的是它会在运行时出现 panic。
问题在于 FromStrings 创建了 []T 类型的切片。当用 *Settable 实例化时，这意味着 []*Settable 类型的切片。

When `FromStrings` calls `result[i].Set(v)`, that invokes the `Set`
method on the pointer stored in `result[i]`.
That pointer is `nil`.
The `Settable.Set` method will be invoked with a `nil` receiver, and
will raise a panic due to a `nil` dereference error.

当 `FromStrings` 调用` result[i].Set(v)` 时，它会调用存储在 `result[i] `中的指针上的 `Set` 方法。该指针为零。 
`Settable.Set` 方法将使用 `nil` 接收器调用，并且会由于 `nil` 取消引用错误而引发恐慌。

The pointer type `*Settable` implements the constraint, but the code
really wants to use the non-pointer type`Settable`.
What we need is a way to write `FromStrings` such that it can take the
type `Settable` as an argument but invoke a pointer method.
To repeat, we can't use `Settable` because it doesn't have a `Set`
method, and we can't use `*Settable` because then we can't create a
slice of type `Settable`.

指针类型`*Settable`实现了约束，但是代码要求非指针类型`Settable`。
我们需要的是一种编写 `FromStrings` 的方法，它可以将 `Settable` 类型作为参数但调用指针方法。
重复一遍，我们不能使用 `Settable`，因为它没有 `Set` 方法，我们不能使用 `*Settable`，因为那样我们就不能创建 `Settable` 类型的切片。

What we can do is pass both types.

我们能做的就是传递这两种类型。

```go
// Setter2 is a type constraint that requires that the type
// implement a Set method that sets the value from a string,
// and also requires that the type be a pointer to its type parameter.
type Setter2[B any] interface {
	Set(string)
	*B // non-interface type constraint element
}

// FromStrings2 takes a slice of strings and returns a slice of T,
// calling the Set method to set each returned value.
//
// We use two different type parameters so that we can return
// a slice of type T but call methods on *T aka PT.
// The Setter2 constraint ensures that PT is a pointer to T.
func FromStrings2[T any, PT Setter2[T]](s []string) []T {
	result := make([]T, len(s))
	for i, v := range s {
		// The type of &result[i] is *T which is in the type set
		// of Setter2, so we can convert it to PT.
		p := PT(&result[i])
		// PT has a Set method.
		p.Set(v)
	}
	return result
}
```

We can then call `FromStrings2` like this:

然后我们可以这样调用 FromStrings2：

```go
func F2() {
	// FromStrings2 takes two type parameters.
	// The second parameter must be a pointer to the first.
	// Settable is as above.
	nums := FromStrings2[Settable, *Settable]([]string{"1", "2"})
	// Now nums is []Settable{1, 2}.
	...
}
```

This approach works as expected, but it is awkward to have to repeat
`Settable` in the type arguments.
Fortunately, constraint type inference makes it less awkward.
Using constraint type inference we can write

这种方法按预期工作，但必须在类型参数中重复 `Settable` 是很尴尬的。
幸运的是，约束类型推断使它不那么尴尬。

使用约束类型推断我们可以写：

```go
func F3() {
	// Here we just pass one type argument.
	nums := FromStrings2[Settable]([]string{"1", "2"})
	// Now nums is []Settable{1, 2}.
	...
}
```

There is no way to avoid passing the type argument `Settable`.
But given that type argument, constraint type inference can infer the
type argument `*Settable` for the type parameter `PT`.

As before, we create a mapping of known type arguments:


无法避免传递类型参数 `Settable`。
但是给定该类型参数，约束类型推断可以为类型参数 `PT` 推断类型参数 `*Settable`。
和以前一样，我们创建一个已知类型参数的映射：

```go
{T -> Settable}
```

然后我们将每个类型参数与结构约束统一起来。在这种情况下，我们将 `PT` 统一为 `Setter2[T]` 的单一类型，即 *T。

现在的映射

```go
{T -> Settable, PT -> *T}
```

然后我们用Settable代替T，使我们得到：

```
{T -> Settable, PT -> *Settable}
```

After this nothing changes, and we are done.
Both type arguments are known.

This example shows how we can use constraint type inference to apply a
constraint to a type that is based on some other type parameter.
In this case we are saying that `PT`, which is `*T`, must have a `Set`
method.
We can do this without requiring the caller to explicitly mention
`*T`.

在此之后什么都没有改变，我们就完成了。两种`type arguments `都是已知的。
这个例子展示了我们如何使用约束类型推断来将约束应用于基于其他`type parameter`的类型。
在这种情况下，我们说 `PT`，也就是` *T`，必须有一个 `Set` 方法。我们可以在不要求调用者显式提及 `*T` 的情况下执行此操作。

##### Constraints apply even after constraint type inference

Even when constraint type inference is used to infer type arguments
based on constraints, we must still check the constraints after the
type arguments are determined.

In the `FromStrings2` example above, we were able to deduce the type
argument for `PT` based on the `Setter2` constraint.
But in doing so we only looked at the type set, we didn't look at the
methods.
We still have to verify that the method is there, satisfying the
constraint, even if constraint type inference succeeds.

For example, consider this invalid code:


即使使用约束类型推断来根据约束推断` type arguments`，我们仍然必须在确定` type arguments`后检查约束。
在上面的 `FromStrings2` 示例中，我们能够根据 `Setter2` 约束推断出 `PT` 的` type arguments`。
但在这样做时，我们只查看了类型集，没有查看方法。即使约束类型推断成功，我们仍然必须验证该方法是否存在，是否满足约束。
例如，考虑以下无效代码：

```
// Unsettable is a type that does not have a Set method.
type Unsettable int

func F4() {
	// This call is INVALID.
	nums := FromStrings2[Unsettable]([]string{"1", "2"})
	...
}
```

When this call is made, we will apply constraint type inference just
as before.
It will succeed, just as before, and infer that the type arguments are
`[Unsettable, *Unsettable]`.
Only after constraint type inference is complete will we check whether
`*Unsettable` implements the constraint `Setter2[Unsettable]`.
Since `*Unsettable` does not have a `Set` method, constraint checking
will fail, and this code will not compile.


进行此调用时，我们将像以前一样应用约束类型推断。
它会像以前一样成功，并推断`type arguments`是 `[Unsettable, *Unsettable]`。
只有在约束类型推断完成后，我们才会检查` *Unsettable` 是否实现了约束 `Setter2[Unsettable`]。由于 `*Unsettable` 没有 `Set` 方法，约束检查将失败，并且此代码将无法编译。

### Using types that refer to themselves in constraints

It can be useful for a generic function to require a type argument
with a method whose argument is the type itself.
For example, this arises naturally in comparison methods.
(Note that we are talking about methods here, not operators.)
Suppose we want to write an `Index` method that uses an `Equal` method
to check whether it has found the desired value.
We would like to write that like this:

对于一个泛型函数来说，要求一个`type argument`和一个参数是类型本身的方法是很有用的。
例如，这在比较方法中很自然地出现。 （请注意，我们在这里谈论的是方法，而不是运算符。）
假设我们要编写一个 `Index` 方法，它使用 `Equal` 方法来检查它是否找到了所需的值。

我们想这样写：

```go
// Index returns the index of e in s, or -1 if not found.
func Index[T Equaler](s []T, e T) int {
	for i, v := range s {
		if e.Equal(v) {
			return i
		}
	}
	return -1
}
```

In order to write the `Equaler` constraint, we have to write a
constraint that can refer to the type argument being passed in.
The easiest way to do this is to take advantage of the fact that a
constraint does not have to be a defined type, it can simply be an
interface type literal.
This interface type literal can then refer to the type parameter.

为了编写 Equaler 约束，我们必须编写一个可以引用传入的`type argument`的约束。
最简单的方法是利用约束不必是已定义类型这一事实，它可以只是一个接口类型文字。
然后，此接口类型文字可以引用`type parameter`。

```go
// Index returns the index of e in s, or -1 if not found.
func Index[T interface { Equal(T) bool }](s []T, e T) int {
	// same as above
}
```

This version of `Index` would be used with a type like `equalInt`
defined here:

```go
// equalInt is a version of int that implements Equaler.
type equalInt int

// The Equal method lets equalInt implement the Equaler constraint.
func (a equalInt) Equal(b equalInt) bool { return a == b }

// indexEqualInts returns the index of e in s, or -1 if not found.
func indexEqualInt(s []equalInt, e equalInt) int {
	// The type argument equalInt is shown here for clarity.
	// Function argument type inference would permit omitting it.
	return Index[equalInt](s, e)
}
```

In this example, when we pass `equalInt` to `Index`, we check whether
`equalInt` implements the constraint `interface { Equal(T) bool }`.
The constraint has a type parameter, so we replace the type parameter
with the type argument, which is `equalInt` itself.
That gives us `interface { Equal(equalInt) bool }`.
The `equalInt` type has an `Equal` method with that signature, so all
is well, and the compilation succeeds.

在这个例子中，当我们将 `equalInt` 传递给 `Index` 时，我们检查 `equalInt` 是否实现了约束 `interface { Equal(T) bool }`。
约束有一个类型参数，因此我们将`type parameter`替换为`type argument`，即 equalInt 本身。
这样我们就能得到 `interface { Equal(equalInt) bool }`。
`equalInt` 类型有一个带有该签名的 `Equal` 方法，编译成功。

### Values of type parameters are not boxed

In the current implementations of Go, interface values always hold
pointers.
Putting a non-pointer value in an interface variable causes the value
to be _boxed_.
That means that the actual value is stored somewhere else, on the heap
or stack, and the interface value holds a pointer to that location.

在 Go 的当前实现中，接口值始终保存指针。将非指针值放在接口变量中会导致该值被装箱。这意味着实际值存储在其他地方，例如在堆或堆栈上，并且接口值包含指向该位置的指针。

In this design, values of generic types are not boxed.
For example, let's look back at our earlier example of
`FromStrings2`.
When it is instantiated with type `Settable`, it returns a value of
type `[]Settable`.
For example, we can write

在此设计中，泛型类型的值未装箱。
例如，让我们回顾一下我们之前的 `FromStrings2` 示例。当它以 `Settable` 类型实例化时，它返回一个 `[]Settable` 类型的值。


```go
// Settable is an integer type that can be set from a string.
type Settable int

// Set sets the value of *p from a string.
func (p *Settable) Set(s string) {
	// same as above
}

func F() {
	// The type of nums is []Settable.
	nums := FromStrings2[Settable]([]string{"1", "2"})
	// Settable can be converted directly to int.
	// This will set first to 1.
	first := int(nums[0])
	...
}
```

When we call `FromStrings2` instantiated with the type `Settable` we
get back a `[]Settable`.
The elements of that slice will be `Settable` values, which is to say,
they will be integers.
They will not be boxed, even though they were created and set by a
generic function.

Similarly, when a generic type is instantiated it will have the
expected types as components.

当我们调用以 `Settable` 类型实例化的 `FromStrings2` 时，我们得到一个 `[]Settable`。
该切片的元素将是可设置的值，也就是说，它们将是整数。

它们不会被装箱，即使它们是由泛型函数创建和设置的。

``` go
type Pair[F1, F2 any] struct {
	first  F1
	second F2
}
```

When this is instantiated, the fields will not be boxed, and no
unexpected memory allocations will occur.
The type `Pair[int, string]` is convertible to `struct { first int;
second string }`.

实例化时，字段不会被装箱，也不会发生意外的内存分配。类型` Pair[int, string] ` 可转换为 `struct { first int; second string}`。

### More on type sets

Let's return now to type sets to cover some less important details
that are still worth noting.

现在让我们回到类型集来讨论一些不太重要但仍然值得注意的细节。

#### Both elements and methods in constraints

As seen earlier for `Setter2`, a constraint may use both constraint
elements and methods.

正如前面看到的 Setter2，约束可以同时使用约束元素和方法。

```go 
// StringableSignedInteger is a type constraint that matches any
// type that is both 1) defined as a signed integer type;
// 2) has a String method.
type StringableSignedInteger interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64
	String() string
}
```

The rules for type sets define what this means.

类型集的规则定义了这一含义。

The type set of the union element is the set of all types whose
underlying type is one of the predeclared signed integer types.

联合元素的类型集是所有类型的集合，其基础类型是预先声明的有符号整数类型之一。

The type set of `String() string` is the set of all types that define
that method.

`String()` 字符串的类型集是定义该方法的所有类型的集合。

The type set of `StringableSignedInteger` is the intersection of those
two type sets.

`StringableSignedInteger` 的类型集是这两个类型集的交集。

The result is the set of all types whose underlying type is one of the
predeclared signed integer types and that defines the method `String()
string`.

结果是所有类型的集合，其基础类型是预先声明的有符号整数类型之一，并且定义了方法` String() string`。

A function that uses a parameterized type `P` that uses
`StringableSignedInteger` as a constraint may use the operations
permitted for any integer type (`+`, `*`, and so forth) on a value of
type `P`.

使用将 `StringableSignedInteger` 作为约束的参数化类型 `P` 的函数可以对类型 `P` 的值使用任何整数类型`（+、* 等）`允许的操作。

It may also call the `String` method on a value of type `P` to get
back a `string`.

它还可以对类型 `P` 的值调用 `String` 方法以取回字符串。

It's worth noting that the `~` is essential here.

值得注意的是`~`在这里是必不可少的。

The `StringableSignedInteger` constraint uses `~int`, not `int`.

`StringableSignedInteger` 约束使用 `~int`，而不是 `int`。

The type `int` would not itself be permitted as a type argument, since`int` does not have a `String` method.

类型 `int` 本身不允许作为类型参数，因为 `int` 没有 `String` 方法。


```go
// MyInt is a stringable int.
type MyInt int

// The String method returns a string representation of mi.
func (mi MyInt) String() string {
	return fmt.Sprintf("MyInt(%d)", mi)
}
```

#### Composite types in constraints

As we've seen in some earlier examples, a constraint element may be a
type literal.

正如我们在前面的一些示例中看到的，约束元素可能是字面量。

```go
type byteseq interface {
	string | []byte
}
```

The usual rules apply: the type argument for this constraint may be
`string` or `[]byte`; 

通常的规则适用：此约束的类型参数可以是`string`或` []byte`；

a generic function with this constraint may use
any operation permitted by both `string` and `[]byte`.

具有此约束的通用函数可以使用字符串和 []byte 允许的任何操作。

The `byteseq` constraint permits writing generic functions that work
for either `string` or `[]byte` types.

`byteseq`约束允许编写通用函数，对`string`或`[]byte`类型都有效。

```go
// Join concatenates the elements of its first argument to create a
// single value. sep is placed between elements in the result.
// Join works for string and []byte types.
func Join[T byteseq](a []T, sep T) (ret T) {
	if len(a) == 0 {
		// Use the result parameter as a zero value;
		// see discussion of zero value in the Issues section.
		return ret
	}
	if len(a) == 1 {
		// We know that a[0] is either a string or a []byte.
		// We can append either a string or a []byte to a []byte,
		// producing a []byte. We can convert that []byte to
		// either a []byte (a no-op conversion) or a string.
		return T(append([]byte(nil), a[0]...))
	}
	// We can call len on sep because we can call len
	// on both string and []byte.
	n := len(sep) * (len(a) - 1)
	for _, v := range a {
		// Another case where we call len on string or []byte.
		n += len(v)
	}

	b := make([]byte, n)
	// We can call copy to a []byte with an argument of
	// either string or []byte.
	bp := copy(b, a[0])
	for _, s := range a[1:] {
		bp += copy(b[bp:], sep)
		bp += copy(b[bp:], s)
	}
	// As above, we can convert b to either []byte or string.
	return T(b)
}
```

For composite types (string, pointer, array, slice, struct, function,
map, channel) we impose an additional restriction: an operation may
only be used if the operator accepts identical input types (if any)
and produces identical result types for all of the types in the type
set.
To be clear, this additional restriction is only imposed when a
composite type appears in a type set.
It does not apply when a composite type is formed from a type
parameter outside of a type set, as in `var v []T` for some type
parameter `T`.

对于复合类型（字符串、指针、数组、切片、结构、函数、映射、通道），我们施加了一个额外的限制：只有当运算符接受相同的输入类型（如果有的话）并为所有类型产生相同的结果类型时，才可以使用一个操作类型集中的类型。

需要明确的是，仅当复合类型出现在类型集中时才会施加此附加限制。

当复合类型由类型集之外的类型参数形成时，它不适用，例如在 `var v []T` 中用于某些类型参数 `T`。

```go
// structField is a type constraint whose type set consists of some
// struct types that all have a field named x.
type structField interface {
	struct { a int; x int } |
		struct { b int; x float64 } |
		struct { c int; x uint64 }
}

// This function is INVALID.
func IncrementX[T structField](p *T) {
	v := p.x // INVALID: type of p.x is not the same for all types in set
	v++
	p.x = v
}

// sliceOrMap is a type constraint for a slice or a map.
type sliceOrMap interface {
	[]int | map[int]int
}

// Entry returns the i'th entry in a slice or the value of a map
// at key i. This is valid as the result of the operator is always int.
func Entry[T sliceOrMap](c T, i int) int {
	// This is either a slice index operation or a map key lookup.
	// Either way, the index and result types are type int.
	return c[i]
}

// sliceOrFloatMap is a type constraint for a slice or a map.
type sliceOrFloatMap interface {
	[]int | map[float64]int
}

// This function is INVALID.
// In this example the input type of the index operation is either
// int (for a slice) or float64 (for a map), so the operation is
// not permitted.
func FloatEntry[T sliceOrFloatMap](c T) int {
	return c[1.0] // INVALID: input type is either int or float64.
}
```

Imposing this restriction makes it easier to reason about the type of
some operation in a generic function.
It avoids introducing the notion of a value with a constructed type
set based on applying some operation to each element of a type set.

施加此限制可以更容易地推断出泛型函数中某些操作的类型。它避免引入基于对类型集的每个元素应用某些操作的构造类型集的值的概念。

(Note: with more understanding of how people want to write code, it
may be possible to relax this restriction in the future.)

(注意: 有可能 随着人们越来越了解`Go`的泛型，在后续实现上会放宽这个限制)


#### Type parameters in type sets

A type literal in a constraint element can refer to type parameters of
the constraint.
In this example, the generic function `Map` takes two type parameters.
The first type parameter is required to have an underlying type that
is a slice of the second type parameter.
There are no constraints on the second type parameter.

约束元素中的字面量可以引用约束的`type parameters`。
在此示例中，通用函数 `Map` 采用两个`type parameters`。
第一个`type parameters`需要有一个基础类型，它是第二个类型参数的一部分。
对第二个`type parameters`没有限制。

```go
// SliceConstraint is a type constraint that matches a slice of
// the type parameter.
type SliceConstraint[T any] interface {
	~[]T
}

// Map takes a slice of some element type and a transformation function,
// and returns a slice of the function applied to each element.
// Map returns a slice that is the same type as its slice argument,
// even if that is a defined type.
func Map[S SliceConstraint[E], E any](s S, f func(E) E) S {
	r := make(S, len(s))
	for i, v := range s {
		r[i] = f(v)
	}
	return r
}

// MySlice is a simple defined type.
type MySlice []int

// DoubleMySlice takes a value of type MySlice and returns a new
// MySlice value with each element doubled in value.
func DoubleMySlice(s MySlice) MySlice {
	// The type arguments listed explicitly here could be inferred.
	v := Map[MySlice, int](s, func(e int) int { return 2 * e })
	// Here v has type MySlice, not type []int.
	return v
}
```

We showed other examples of this earlier in the discussion of
[constraint type inference](#Constraint-type-inference).

早期的一些例子 [constraint type inference](#Constraint-type-inference).

#### Type conversions

In a function with two type parameters `From` and `To`, a value of
type `From` may be converted to a value of type `To` if all the types
in the type set of `From`'s constraint can be converted to all the
types in the type set of `To`'s constraint.

This is a consequence of the general rule that a generic function may
use any operation that is permitted by all types listed in the type
set.

在具有两个类型参数 `From` 和 `To` 的函数中，如果 `From` 的约束类型集中的所有类型都可以转换为 `To` 的约束类型集中的所有类型，则 `From` 类型的值可以转换为 `To` 类型的值.
这是一般规则的结果，即泛型函数可以使用类型集中列出的所有类型所允许的任何操作。

```go
type integer interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64 |
		~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr
}

func Convert[To, From integer](from From) To {
	to := To(from)
	if From(to) != from {
		panic("conversion out of range")
	}
	return to
}
```

The type conversions in `Convert` are permitted because Go permits
every integer type to be converted to every other integer type.

`Convert` 中的类型转换是允许的，因为 `Go` 允许将每个整数类型转换为每个其他整数类型。

#### Untyped constants

Some functions use untyped constants.
An untyped constant is permitted with a value of a type parameter if
it is permitted with every type in the type set of the type
parameter's constraint.

As with type conversions, this is a consequence of the general rule
that a generic function may use any operation that is permitted by all
types in the type set.

一些函数使用无类型常量。
如果类型参数约束的类型集中的每个类型都允许无类型常量，则类型参数的值允许无类型常量。
与类型转换一样，这是一般规则的结果，即泛型函数可以使用类型集中所有类型允许的任何操作。

```go
type integer interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64 |
		~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr
}

func Add10[T integer](s []T) {
	for i, v := range s {
		s[i] = v + 10 // OK: 10 can convert to any integer type
	}
}

// This function is INVALID.
func Add1024[T integer](s []T) {
	for i, v := range s {
		s[i] = v + 1024 // INVALID: 1024 not permitted by int8/uint8
	}
}
```

#### Type sets of embedded constraints

When a constraint embeds another constraint, the type set of the
outer constraint is the intersection of all the type sets involved.
If there are multiple embedded types, intersection preserves the
property that any type argument must satisfy the requirements of all
constraint elements.

当一个约束嵌入另一个约束时，外部约束的类型集是所有涉及的类型集的交集。如果有多个嵌入类型，交集保留任何类型参数必须满足所有约束元素的要求的属性。

```go
// Addable is types that support the + operator.
type Addable interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64 |
		~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
		~float32 | ~float64 | ~complex64 | ~complex128 |
		~string
}

// Byteseq is a byte sequence: either string or []byte.
type Byteseq interface {
	~string | ~[]byte
}

// AddableByteseq is a byte sequence that supports +.
// This is every type that is both Addable and Byteseq.
// In other words, just the type set ~string.
type AddableByteseq interface {
	Addable
	Byteseq
}
```

An embedded constraint may appear in a union element.
The type set of the union is, as usual, the union of the type sets of
the elements listed in the union.

嵌入式约束可能出现在联合元素中。像往常一样，联合的类型集是联合中列出的元素的类型集的联合。

```go
// Signed is a constraint with a type set of all signed integer
// types.
type Signed interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64
}

// Unsigned is a constraint with a type set of all unsigned integer
// types.
type Unsigned interface {
	~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr
}

// Integer is a constraint with a type set of all integer types.
type Integer interface {
	Signed | Unsigned
}
```

#### Interface types in union elements

We've said that the type set of a union element is the union of the
type sets of all types in the union.
For most types `T` the type set of `T` is simply `T` itself.
For interface types (and approximation elements), however, that is not
the case.

The type set of an interface type that does not embed a non-interface
element is, as we said earlier, the set of all types that declare all
the methods of the interface, including the interface type itself.
Using such an interface type in a union element will add that type set
to the union.

我们已经说过，联合元素的类型集是联合中所有类型的类型集的联合。对于大多数类型 `T`，`T` 的类型集就是 `T` 本身。然而，对于接口类型（和近似元素），情况并非如此。
不嵌入非接口元素的接口类型的类型集，正如我们前面所说的，是声明接口所有方法的所有类型的集合，包括接口类型本身。在 `union` 元素中使用这样的接口类型会将该类型集添加到 `union` 中。

```go
type Stringish interface {
	string | fmt.Stringer
}
```

The type set of `Stringish` is the type `string` and all types that
implement `fmt.Stringer`.
Any of those types (including `fmt.Stringer` itself) will be permitted
as a type argument for this constraint.
No operations will be permitted for a value of a type parameter that
uses `Stringish` as a constraint (other than operations supported by
all types).
This is because `fmt.Stringer` is in the type set of `Stringish`, and
`fmt.Stringer`, an interface type, does not support any type-specific
operations.
The operations permitted by `Stringish` are those operations supported
by all the types in the type set, including `fmt.Stringer`, so in this
case there are no operations other than those supported by all types.
A parameterized function that uses this constraint will have to use
type assertions or reflection in order to use the values.
Still, this may be useful in some cases for stronger static type
checking.
The main point is that it follows directly from the definition of type
sets and constraint satisfaction.

`Stringish`的类型集是`string`类型和所有实现了`fmt.Stringer`的类型。

任何这些类型（包括 `fmt.Stringer` 本身）都将被允许作为此约束的类型参数。不允许对使用 Stringish 作为约束的类型参数的值进行任何操作（所有类型都支持的操作除外）。

这是因为` fmt.Stringer` 属于 `Stringish` 的类型集，而` fmt.Stringer` 这种接口类型不支持任何特定于类型的操作。 

Stringish 允许的操作是类型集中所有类型都支持的操作，包括 `fmt.Stringer`，所以在这种情况下，除了所有类型都支持的操作之外，没有其他操作。使用此约束的参数化函数必须使用类型断言或反射才能使用这些值。

不过，这在某些情况下对于更强的静态类型检查可能很有用。要点是它直接遵循类型集和约束满足的定义。

#### Empty type sets

It is possible to write a constraint with an empty type set.
There is no type argument that will satisfy such a constraint,
so any attempt to instantiate a function that uses constraint with an
empty type set will fail.
It is not possible in general for the compiler to detect all such
cases.
Probably the vet tool should give an error for cases that it is able
to detect.

写一个具有空类型集的约束是可能的。
没有一个类型参数可以满足这样的约束，所以任何试图实例化一个使用空类型集的约束的函数都会失败。
一般来说，编译器不可能检测到所有这样的情况。
也许`vet tools`应该对它能够检测到的情况给出一个错误。

```go
// Unsatisfiable is an unsatisfiable constraint with an empty type set.
// No predeclared types have any methods.
// If this used ~int | ~float32 the type set would not be empty.
type Unsatisfiable interface {
	int | float32
	String() string
}
```

#### General notes on type sets

It may seem awkward to explicitly list types in a constraint, but it
is clear both as to which type arguments are permitted at the call
site, and which operations are permitted by the generic function.

在约束中明确地列出类型似乎很别扭，但它既清楚地说明了哪些类型的参数在调用点是允许的，也清楚地说明了哪些操作是通用函数所允许的。

If the language later changes to support operator methods (there are
no such plans at present), then constraints will handle them as they
do any other kind of method.

如果语言后来更改为支持运算符方法（目前没有这样的计划），那么约束将像处理任何其他类型的方法一样处理它们。

There will always be a limited number of predeclared types, and a
limited number of operators that those types support.
Future language changes will not fundamentally change those facts, so
this approach will continue to be useful.

总是会有有限数量的预声明类型，以及有限数量的这些类型支持的运算符。未来的语言变化不会从根本上改变这些事实，因此这种方法将继续有用。

This approach does not attempt to handle every possible operator.
The expectation is that composite types will normally be handled using
composite types in generic function and type declarations, rather than
putting composite types in a type set.
For example, we expect functions that want to index into a slice to be
parameterized on the slice element type `T`, and to use parameters or
variables of type `[]T`.

这种方法不会尝试处理每个可能的运算符。期望复合类型通常会在泛型函数和类型声明中使用复合类型来处理，而不是将复合类型放在类型集中。例如，我们期望想要索引到切片的函数在切片元素类型 `T` 上进行参数化，并使用类型为 `[]T `的参数或变量。

As shown in the `DoubleMySlice` example above, this approach makes it
awkward to declare generic functions that accept and return a
composite type and want to return the same result type as their
argument type.
Defined composite types are not common, but they do arise.
This awkwardness is a weakness of this approach.
Constraint type inference can help at the call site.

如上面的 `DoubleMySlice` 示例所示，这种方法使得声明接受和返回复合类型并希望返回与其参数类型相同的结果类型的泛型函数变得很尴尬。
定义的复合类型并不常见，但它们确实出现了。这种尴尬是这种方法的弱点。

### Reflection

We do not propose to change the reflect package in any way.
When a type or function is instantiated, all of the type parameters
will become ordinary non-generic types.
The `String` method of a `reflect.Type` value of an instantiated type
will return the name with the type arguments in square brackets.
For example, `List[int]`.

It's impossible for non-generic code to refer to generic code without
instantiating it, so there is no reflection information for
uninstantiated generic types or functions.

我们不建议以任何方式更改反射包。当一个类型或函数被实例化时，所有的类型参数都会变成普通的非泛型类型。实例化类型的 reflect.Type 值的 String 方法将返回带有方括号中的类型参数的名称。例如，列表 [int]。
非泛型代码不可能在没有实例化的情况下引用泛型代码，因此没有未实例化的泛型类型或函数的反射信息。

### Implementation

Russ Cox [famously observed](https://research.swtch.com/generic) that
generics require choosing among slow programmers, slow compilers, or
slow execution times.

`Russ Cox`有句名言：泛型需要在`研发人员的工作效率`、`编译器编译的效率`或`程序的执行效率`中做出选择。

We believe that this design permits different implementation choices.
Code may be compiled separately for each set of type arguments, or it
may be compiled as though each type argument is handled similarly to
an interface type with method calls, or there may be some combination
of the two.

我们相信这种设计允许不同的实现选择。可以为每组类型参数单独编译代码，或者可以将每个类型参数的处理方式与具有方法调用的接口类型类似地进行编译，或者可以将两者结合起来。

In other words, this design permits people to stop choosing slow
programmers, and permits the implementation to decide between slow
compilers (compile each set of type arguments separately) or slow
execution times (use method calls for each operation on a value of a
type argument).

换句话说，程序员可以自己去做选择

### Summary

While this document is long and detailed, the actual design reduces to
a few major points.

虽然本文档又长又详细，但实际设计减少了几个要点。

* Functions and types can have type parameters, which are defined
  using constraints, which are interface types.

* 函数和类型可以有类型参数，它们是使用约束定义的，它们是接口类型。

* Constraints describe the methods required and the types permitted
  for a type argument.

* 约束描述了类型参数所需的方法和允许的类型。

* Constraints describe the methods and operations permitted for a type
  parameter.

* 约束描述了类型参数允许的方法和操作。


* Type inference will often permit omitting type arguments when
  calling functions with type parameters.

* 类型推断通常允许在调用带有类型参数的函数时省略类型参数。



This design is completely backward compatible.

We believe that this design addresses people's needs for generic
programming in Go, without making the language any more complex than
necessary.

We can't truly know the impact on the language without years of
experience with this design.
That said, here are some speculations.

这种设计是完全向后兼容的。
我们相信这种设计满足了人们对 Go 泛型编程的需求，而不会使语言变得过于复杂。
如果没有多年的这种设计经验，我们就无法真正了解对语言的影响。也就是说，这里有一些猜测。

#### Complexity

One of the great aspects of Go is its simplicity.
Clearly this design makes the language more complex.

We believe that the increased complexity is small for people reading
well written generic code, rather than writing it.
Naturally people must learn the new syntax for declaring type
parameters.
This new syntax, and the new support for type sets in interfaces, are
the only new syntactic constructs in this design.
The code within a generic function reads like ordinary Go code, as can
be seen in the examples below.
It is an easy shift to go from `[]int` to `[]T`.
Type parameter constraints serve effectively as documentation,
describing the type.

We expect that most packages will not define generic types or
functions, but many packages are likely to use generic types or
functions defined elsewhere.
In the common case, generic functions work exactly like non-generic
functions: you simply call them.
Type inference means that you do not have to write out the type
arguments explicitly.
The type inference rules are designed to be unsurprising: either the
type arguments are deduced correctly, or the call fails and requires
explicit type parameters.
Type inference uses type equivalence, with no attempt to resolve two
types that are similar but not equivalent, which removes significant
complexity.

Packages using generic types will have to pass explicit type
arguments.
The syntax for this is straightforward.
The only change is passing arguments to types rather than only to
functions.

In general, we have tried to avoid surprises in the design.
Only time will tell whether we succeeded.

Go的一个伟大方面是它的简单性。显然，这种设计使语言更加复杂。
我们相信，对于阅读写得很好的通用代码的人来说，增加的复杂性是很小的，而不是写出来的。当然，人们必须学习声明类型参数的新语法。这种新的语法，以及对接口中类型集的新支持，是本设计中唯一的新语法结构。泛型函数中的代码读起来就像普通的Go代码，在下面的例子中可以看到。从`[]int`到`[]T`是一个简单的转变。类型参数约束有效地充当了文档，描述了类型。
我们期望大多数包不会定义通用类型或函数，但许多包可能会使用其他地方定义的通用类型或函数。在普通情况下，泛型函数的工作方式与非泛型函数完全一样：你只需调用它们。类型推理意味着你不需要明确地写出类型参数。类型推理规则被设计成不出人意料的：要么类型参数被正确推导出来，要么调用失败，需要显式类型参数。类型推理使用类型等价，不试图解决两个相似但不等价的类型，这消除了显著的复杂性。
使用通用类型的包将不得不传递显式类型参数。这方面的语法是直截了当的。唯一的变化是将参数传递给类型而不是只传递给函数。
总的来说，我们试图在设计中避免出现意外。只有时间能证明我们是否成功了

#### Pervasiveness

We expect that a few new packages will be added to the standard
library.

我们希望将一些新包添加到标准库中。

A new `slices` packages will be similar to the existing bytes and
strings packages, operating on slices of any element type.



新的slices包将类似于现有的byte和strings包，对任何元素类型的片断进行操作。


New `maps` and `chans` packages will provide algorithms that are
currently duplicated for each element type.

新的maps和chans包将提供目前对每个元素类型重复的算法。

A `sets` package may be added.

可能会增加一个集合包。

A new `constraints` package will provide standard constraints, such as
constraints that permit all integer types or all numeric types.


一个新的约束包将提供标准约束，比如允许所有整数类型或所有数字类型的约束。

Packages like `container/list` and `container/ring`, and types like
`sync.Map` and `sync/atomic.Value`, will be updated to be compile-time
type-safe, either using new names or new versions of the packages.

像container/list和container/ring这样的包，以及像sync.Map和sync/atomic.Value这样的类型，将被更新为编译时类型安全的，要么使用新的名字，要么使用包的新版本。

The `math` package will be extended to provide a set of simple
standard algorithms for all numeric types, such as the ever popular
`Min` and `Max` functions.


`math package`将被扩展，为所有数字类型提供一套简单的标准算法，比如一直很流行的`Min`和`Max`函数。


We may add generic variants to the `sort` package.

我们可能会在`sort package` 中添加泛型能力。

It is likely that new special purpose compile-time type-safe container
types will be developed.

We do not expect approaches like the C++ STL iterator types to become
widely used.
In Go that sort of idea is more naturally expressed using an interface
type.
In C++ terms, using an interface type for an iterator can be seen as
carrying an abstraction penalty, in that run-time efficiency will be
less than C++ approaches that in effect inline all code; we believe
that Go programmers will continue to find that sort of penalty to be
acceptable.

As we get more container types, we may develop a standard `Iterator`
interface.
That may in turn lead to pressure to modify the language to add some
mechanism for using an `Iterator` with the `range` clause.
That is very speculative, though.

#### Efficiency

It is not clear what sort of efficiency people expect from generic
code.

Generic functions, rather than generic types, can probably be compiled
using an interface-based approach.
That will optimize compile time, in that the function is only compiled
once, but there will be some run time cost.

Generic types may most naturally be compiled multiple times for each
set of type arguments.
This will clearly carry a compile time cost, but there shouldn't be
any run time cost.
Compilers can also choose to implement generic types similarly to
interface types, using special purpose methods to access each element
that depends on a type parameter.

Only experience will show what people expect in this area.

#### Omissions

We believe that this design covers the basic requirements for
generic programming.
However, there are a number of programming constructs that are not
supported.

* No specialization.
  There is no way to write multiple versions of a generic function
  that are designed to work with specific type arguments.
* No metaprogramming.
  There is no way to write code that is executed at compile time to
  generate code to be executed at run time.
* No higher level abstraction.
  There is no way to use a function with type arguments other than to
  call it or instantiate it.
  There is no way to use a generic type other than to instantiate it.
* No general type description.
  In order to use operators in a generic function, constraints list
  specific types, rather than describing the characteristics that a
  type must have.
  This is easy to understand but may be limiting at times.
* No covariance or contravariance of function parameters.
* No operator methods.
  You can write a generic container that is compile-time type-safe,
  but you can only access it with ordinary methods, not with syntax
  like `c[k]`.
* No currying.
  There is no way to partially instantiate a generic function or type,
  other than by using a helper function or a wrapper type.
  All type arguments must be either explicitly passed or inferred at
  instantiation time.
* No variadic type parameters.
  There is no support for variadic type parameters, which would permit
  writing a single generic function that takes different numbers of
  both type parameters and regular parameters.
* No adaptors.
  There is no way for a constraint to define adaptors that could be
  used to support type arguments that do not already implement the
  constraint, such as, for example, defining an `==` operator in terms
  of an `Equal` method, or vice-versa.
* No parameterization on non-type values such as constants.
  This arises most obviously for arrays, where it might sometimes be
  convenient to write `type Matrix[n int] [n][n]float64`.
  It might also sometimes be useful to specify significant values for
  a container type, such as a default value for elements.

#### Issues

There are some issues with this design that deserve a more detailed
discussion.
We think these issues are relatively minor compared to the design as a
whole, but they still deserve a complete hearing and discussion.

##### The zero value

This design has no simple expression for the zero value of a type
parameter.
For example, consider this implementation of optional values that uses
pointers:

```
type Optional[T any] struct {
	p *T
}

func (o Optional[T]) Val() T {
	if o.p != nil {
		return *o.p
	}
	var zero T
	return zero
}
```

In the case where `o.p == nil`, we want to return the zero value of
`T`, but we have no way to write that.
It would be nice to be able to write `return nil`, but that wouldn't
work if `T` is, say, `int`; in that case we would have to write
`return 0`.
And, of course, there is no way to write a constraint to support
either `return nil` or `return 0`.

Some approaches to this are:

* Use `var zero T`, as above, which works with the existing design
  but requires an extra statement.
* Use `*new(T)`, which is cryptic but works with the existing
  design.
* For results only, name the result parameter, and use a naked
  `return` statement to return the zero value.
* Extend the design to permit using `nil` as the zero value of any
  generic type (but see [issue 22729](https://golang.org/issue/22729)).
* Extend the design to permit using `T{}`, where `T` is a type
  parameter, to indicate the zero value of the type.
* Change the language to permit using `_` on the right hand of an
  assignment (including `return` or a function call) as proposed in
  [issue 19642](https://golang.org/issue/19642).
* Change the language to permit `return ...` to return zero values of
  the result types, as proposed in
  [issue 21182](https://golang.org/issue/21182).

We feel that more experience with this design is needed before
deciding what, if anything, to do here.

##### Identifying the matched predeclared type

The design doesn't provide any way to test the underlying type matched
by a `~T` constraint element.
Code can test the actual type argument through the somewhat awkward
approach of converting to an empty interface type and using a type
assertion or a type switch.
But that lets code test the actual type argument, which is not the
same as the underlying type.

Here is an example that shows the difference.

```
type Float interface {
	~float32 | ~float64
}

func NewtonSqrt[T Float](v T) T {
	var iterations int
	switch (interface{})(v).(type) {
	case float32:
		iterations = 4
	case float64:
		iterations = 5
	default:
		panic(fmt.Sprintf("unexpected type %T", v))
	}
	// Code omitted.
}

type MyFloat float32

var G = NewtonSqrt(MyFloat(64))
```

This code will panic when initializing `G`, because the type of `v` in
the `NewtonSqrt` function will be `MyFloat`, not `float32` or
`float64`.
What this function actually wants to test is not the type of `v`, but
the approximate type that `v` matched in the constraint's type set.

One way to handle this would be to permit writing approximate types in
a type switch, as in `case ~float32:`.
Such a case would match any type whose underlying type is `float32`.
This would be meaningful, and potentially useful, even in type
switches outside of generic functions.

##### No way to express convertibility

The design has no way to express convertibility between two different
type parameters.
For example, there is no way to write this function:

```
// Copy copies values from src to dst, converting them as they go.
// It returns the number of items copied, which is the minimum of
// the lengths of dst and src.
// This implementation is INVALID.
func Copy[T1, T2 any](dst []T1, src []T2) int {
	for i, x := range src {
		if i > len(dst) {
			return i
		}
		dst[i] = T1(x) // INVALID
	}
	return len(src)
}
```

The conversion from type `T2` to type `T1` is invalid, as there is no
constraint on either type that permits the conversion.
Worse, there is no way to write such a constraint in general.
In the particular case where both `T1` and `T2` have limited type sets
this function can be written as described earlier when discussing
[type conversions using type sets](#Type-conversions).
But, for example, there is no way to write a constraint for the case
in which `T1` is an interface type and `T2` is a type that implements
that interface.

It's worth noting that if `T1` is an interface type then this can be
written using a conversion to the empty interface type and a type
assertion, but this is, of course, not compile-time type-safe.

```
// Copy copies values from src to dst, converting them as they go.
// It returns the number of items copied, which is the minimum of
// the lengths of dst and src.
func Copy[T1, T2 any](dst []T1, src []T2) int {
	for i, x := range src {
		if i > len(dst) {
			return i
		}
		dst[i] = (interface{})(x).(T1)
	}
	return len(src)
}
```

##### No parameterized methods

This design does not permit methods to declare type parameters that
are specific to the method.
The receiver may have type parameters, but the method may not add any
type parameters.

In Go, one of the main roles of methods is to permit types to
implement interfaces.
It is not clear whether it is reasonably possible to permit
parameterized methods to implement interfaces.
For example, consider this code, which uses the obvious syntax for
parameterized methods.
This code uses multiple packages to make the problem clearer.

```
package p1

// S is a type with a parameterized method Identity.
type S struct{}

// Identity is a simple identity method that works for any type.
func (S) Identity[T any](v T) T { return v }

package p2

// HasIdentity is an interface that matches any type with a
// parameterized Identity method.
type HasIdentity interface {
	Identity[T any](T) T
}

package p3

import "p2"

// CheckIdentity checks the Identity method if it exists.
// Note that although this function calls a parameterized method,
// this function is not itself parameterized.
func CheckIdentity(v interface{}) {
	if vi, ok := v.(p2.HasIdentity); ok {
		if got := vi.Identity[int](0); got != 0 {
			panic(got)
		}
	}
}

package p4

import (
	"p1"
	"p3"
)

// CheckSIdentity passes an S value to CheckIdentity.
func CheckSIdentity() {
	p3.CheckIdentity(p1.S{})
}
```

In this example, we have a type `p1.S` with a parameterized method and
a type `p2.HasIdentity` that also has a parameterized method.
`p1.S` implements `p2.HasIdentity`.
Therefore, the function `p3.CheckIdentity` can call `vi.Identity` with
an `int` argument, which in the call from `p4.CheckSIdentity` will be
a call to `p1.S.Identity[int]`.
But package p3 does not know anything about the type `p1.S`.
There may be no other call to `p1.S.Identity` elsewhere in the
program.
We need to instantiate `p1.S.Identity[int]` somewhere, but how?

We could instantiate it at link time, but in the general case that
requires the linker to traverse the complete call graph of the program
to determine the set of types that might be passed to `CheckIdentity`.
And even that traversal is not sufficient in the general case when
type reflection gets involved, as reflection might look up methods
based on strings input by the user.
So in general instantiating parameterized methods in the linker might
require instantiating every parameterized method for every possible
type argument, which seems untenable.

Or, we could instantiate it at run time.
In general this means using some sort of JIT, or compiling the code to
use some sort of reflection based approach.
Either approach would be very complex to implement, and would be
surprisingly slow at run time.

Or, we could decide that parameterized methods do not, in fact,
implement interfaces, but then it's much less clear why we need
methods at all.
If we disregard interfaces, any parameterized method can be
implemented as a parameterized function.

So while parameterized methods seem clearly useful at first glance, we
would have to decide what they mean and how to implement that.

##### No way to require pointer methods

In some cases a parameterized function is naturally written such that
it always invokes methods on addressable values.
For example, this happens when calling a method on each element of a
slice.
In such a case, the function only requires that the method be in the
slice element type's pointer method set.
The type constraints described in this design  have no way to write
that requirement.

For example, consider a variant of the `Stringify` example we [showed
earlier](#Using-a-constraint).

```
// Stringify2 calls the String method on each element of s,
// and returns the results.
func Stringify2[T Stringer](s []T) (ret []string) {
	for i := range s {
		ret = append(ret, s[i].String())
	}
	return ret
}
```

Suppose we have a `[]bytes.Buffer` and we want to convert it into a
`[]string`.
The `Stringify2` function here won't help us.
We want to write `Stringify2[bytes.Buffer]`, but we can't, because
`bytes.Buffer` doesn't have a `String` method.
The type that has a `String` method is `*bytes.Buffer`.
Writing `Stringify2[*bytes.Buffer]` doesn't help because that function
expects a `[]*bytes.Buffer`, but we have a `[]bytes.Buffer`.

We discussed a similar case in the [pointer method
example](#Pointer-method-example) above.
There we used constraint type inference to help simplify the problem.
Here that doesn't help, because `Stringify2` doesn't really care about
calling a pointer method.
It just wants a type that has a `String` method, and it's OK if the
method is only in the pointer method set, not the value method set.
But we also want to accept the case where the method is in the value
method set, for example if we really do have a `[]*bytes.Buffer`.

What we need is a way to say that the type constraint applies to
either the pointer method set or the value method set.
The body of the function would be required to only call the method on
addressable values of the type.

It's not clear how often this problem comes up in practice.

##### No association between float and complex

Constraint type inference lets us give a name to the element of a
slice type, and to apply other similar type decompositions.
However, there is no way to associate a float type and a complex type.
For example, there is no way to write the predeclared `real`, `imag`,
or `complex` functions with this design.
There is no way to say "if the argument type is `complex64`, then the
result type is `float32`."

One possible approach here would be to permit `real(T)` as a type
constraint meaning "the float type associated with the complex type
`T`".
Similarly, `complex(T)` would mean "the complex type associated with
the floating point type `T`".
Constraint type inference would simplify the call site.
However, that would be unlike other type constraints.

#### Discarded ideas

This design is not perfect, and there may be ways to improve it.
That said, there are many ideas that we've already considered in
detail.
This section lists some of those ideas in the hopes that it will help
to reduce repetitive discussion.
The ideas are presented in the form of a FAQ.

##### What happened to contracts?

An earlier draft design of generics implemented constraints using a
new language construct called contracts.
Type sets appeared only in contracts, rather than in interface types.
However, many people had a hard time understanding the difference
between contracts and interface types.
It also turned out that contracts could be represented as a set of
corresponding interfaces; there was no loss in expressive power
without contracts.
We decided to simplify the approach to use only interface types.

##### Why not use methods instead of type sets?

_Type sets are weird._
_Why not write methods for all operators?_

It is possible to permit operator tokens as method names, leading to
methods such as `+(T) T`.
Unfortunately, that is not sufficient.
We would need some mechanism to describe a type that matches any
integer type, for operations such as shifts `<<(integer) T` and
indexing `[](integer) T` which are not restricted to a single int
type.
We would need an untyped boolean type for operations such as `==(T)
untyped bool`.
We would need to introduce new notation for operations such as
conversions, or to express that one may range over a type, which would
likely require some new syntax.
We would need some mechanism to describe valid values of untyped
constants.
We would have to consider whether support for `<(T) bool` means that a
generic function can also use `<=`, and similarly whether support for
`+(T) T` means that a function can also use `++`.
It might be possible to make this approach work but it's not
straightforward.
The approach used in this design seems simpler and relies on only one
new syntactic construct (type sets) and one new name (`comparable`).

##### Why not put type parameters on packages?

We investigated this extensively.
It becomes problematic when you want to write a `list` package, and
you want that package to include a `Transform` function that converts
a `List` of one element type to a `List` of another element type.
It's very awkward for a function in one instantiation of a package to
return a type that requires a different instantiation of the same
package.

It also confuses package boundaries with type definitions.
There is no particular reason to think that the uses of generic types
will break down neatly into packages.
Sometimes they will, sometimes they won't.

##### Why not use the syntax `F<T>` like C++ and Java?

When parsing code within a function, such as `v := F<T>`, at the point
of seeing the `<` it's ambiguous whether we are seeing a type
instantiation or an expression using the `<` operator.
This is very difficult to resolve without type information.

For example, consider a statement like

```
	a, b = w < x, y > (z)
```

Without type information, it is impossible to decide whether the right
hand side of the assignment is a pair of expressions (`w < x` and `y >
(z)`), or whether it is a generic function instantiation and call that
returns two result values (`(w<x, y>)(z)`).

It is a key design decision of Go that parsing be possible without
type information, which seems impossible when using angle brackets for
generics.

##### Why not use the syntax `F(T)`?

An earlier version of this design used that syntax.
It was workable but it introduced several parsing ambiguities.
For example, when writing `var f func(x(T))` it wasn't clear whether
the type was a function with a single unnamed parameter of the
instantiated type `x(T)` or whether it was a function with a parameter
named `x` with type `(T)` (more usually written as `func(x T)`, but in
this case with a parenthesized type).

There were other ambiguities as well.
For `[]T(v1)` and `[]T(v2){}`, at the point of the open parentheses we
don't know whether this is a type conversion (of the value `v1` to the
type `[]T`) or a type literal (whose type is the instantiated type
`T(v2)`).
For `interface { M(T) }` we don't know whether this an interface with
a method `M` or an interface with an embedded instantiated interface
`M(T)`.
These ambiguities are solvable, by adding more parentheses, but
awkward.

Also some people were troubled by the number of parenthesized lists
involved in declarations like `func F(T any)(v T)(r1, r2 T)` or in
calls like `F(int)(1)`.

##### Why not use `F«T»`?

We considered it but we couldn't bring ourselves to require
non-ASCII.

##### Why not define constraints in a builtin package?

_Instead of writing out type sets, use names like_
_`constraints.Arithmetic` and `constraints.Comparable`._

Listing all the possible combinations of types gets rather lengthy.
It also introduces a new set of names that not only the writer of
generic code, but, more importantly, the reader, must remember.
One of the driving goals of this design is to introduce as few new
names as possible.
In this design we introduce only two new predeclared names,
`comparable` and `any`.

We expect that if people find such names useful, we can introduce a
package `constraints` that defines those names in the form of
constraints that can be used by other types and functions and embedded
in other constraints.
That will define the most useful names in the standard library while
giving programmers the flexibility to use other combinations of types
where appropriate.

##### Why not permit type assertions on values whose type is a type parameter?

In an earlier version of this design, we permitted using type
assertions and type switches on variables whose type was a type
parameter, or whose type was based on a type parameter.
We removed this facility because it is always possible to convert a
value of any type to the empty interface type, and then use a type
assertion or type switch on that.
Also, it was sometimes confusing that in a constraint with a type
set that uses approximation elements, a type assertion or type switch
would use the actual type argument, not the underlying type of the
type argument (the difference is explained in the section on
[identifying the matched predeclared type](#Identifying-the-matched-predeclared-type)).

#### Comparison with Java

Most complaints about Java generics center around type erasure.
This design does not have type erasure.
The reflection information for a generic type will include the full
compile-time type information.

In Java type wildcards (`List<? extends Number>`, `List<? super
Number>`) implement covariance and contravariance.
These concepts are missing from Go, which makes generic types much
simpler.

#### Comparison with C++

C++ templates do not enforce any constraints on the type arguments
(unless the concept proposal is adopted).
This means that changing template code can accidentally break far-off
instantiations.
It also means that error messages are reported only at instantiation
time, and can be deeply nested and difficult to understand.
This design avoids these problems through mandatory and explicit
constraints.

C++ supports template metaprogramming, which can be thought of as
ordinary programming done at compile time using a syntax that is
completely different than that of non-template C++.
This design has no similar feature.
This saves considerable complexity while losing some power and run
time efficiency.

C++ uses two-phase name lookup, in which some names are looked up in
the context of the template definition, and some names are looked up
in the context of the template instantiation.
In this design all names are looked up at the point where they are
written.

In practice, all C++ compilers compile each template at the point
where it is instantiated.
This can slow down compilation time.
This design offers flexibility as to how to handle the compilation of
generic functions.

#### Comparison with Rust

The generics described in this design are similar to generics in
Rust.

One difference is that in Rust the association between a trait bound
and a type must be defined explicitly, either in the crate that
defines the trait bound or the crate that defines the type.
In Go terms, this would mean that we would have to declare somewhere
whether a type satisfied a constraint.
Just as Go types can satisfy Go interfaces without an explicit
declaration, in this design Go type arguments can satisfy a constraint
without an explicit declaration.

Where this design uses type sets, the Rust standard library defines
standard traits for operations like comparison.
These standard traits are automatically implemented by Rust's
primitive types, and can be implemented by user defined types as
well.
Rust provides a fairly extensive list of traits, at least 34, covering
all of the operators.

Rust supports type parameters on methods, which this design does not.

## Author

* Ian Lance Taylor
* Robert Griesemer
* August 20, 2021