---
title: "Type Parameters Proposal"
date: 2023-02-20T03:54:48+08:00
draft: false
tags: ["translate", "Golang", "Generics", "Proposal"]
---

## Status

This is the design for adding generic programming using type
parameters to the Go language.
This design has been [proposed and
accepted](https://golang.org/issue/43651) as a future language change.
We currently expect that this change will be available in the Go 1.18
release in early 2022.

这是在Go语言中加入使用类型参数的通用编程的设计。
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

我们建议扩展`Go`语言，在类型和函数声明中增加可选的类型参数。

类型参数受到接口类型的约束。

接口类型作为类型约束时，支持嵌入额外的元素，可用于限制满足约束的类型集。

参数化的类型和函数可以使用带有类型参数的操作符，但只有在满足参数约束的所有类型允许的情况下。

通过统一算法进行的类型推断允许在许多情况下从函数调用中省略类型参数。

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

* 函数可以有一个额外的类型参数列表，它使用方括号，但其他方面看起来像一个普通的参数列表 `func F[T any](p T) { ... }`。
* 这些类型的参数可以由常规参数和在函数体中使用。
* 类型也可以有一个类型参数列表：`type M[T any] []T`。
* 每个类型参数都有一个类型约束，就像每个普通参数都有一个类型一样：`func F[T Constraint](p T) { ... }`。
* 类型约束是接口类型。
* 将会预置一个新的类型`any`, 是允许任何类型的类型约束。
* 用作类型约束的接口类型可以嵌入额外的元素来限制满足约束的类型参数集：
    * 任意类型 `T` 限制为该类型。
    * 一个近似元素`~T`限制了底层类型为`T`的所有类型。
    * 也可以使用组合元素  `T1 | T2 | ...`，限制到所列的任何元素。
* `Generic Functions` 只能使用约束条件所允许的所有类型所支持的操作。
* 使用`Generic Functions`或类型需要传递类型参数。
* 类型推断允许在常见情况下省略函数调用的类型参数。


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

某些人建议对Go语言进行扩展，增加一种参数化的多态性，其中类型参数不是由已声明的子类型关系（如一些面向对象的语言）来约束，而是由明确定义的结构约束。这个版本的设计与2019年7月31日提交的设计草案有许多相似之处，但`contracts`已被删除，由接口类型取代，而且语法也有变化。


已经有几个关于增加类型参数的建议，可以通过上面的链接找到。这里提出的许多想法以前就出现过。这里描述的主要新特征是语法和对接口类型作为约束条件的仔细检查。

这种设计不支持模板元编程或任何其他形式的编译时编程。~~~哈哈哈（^.^）

由于泛型一词在`Go`社区中被广泛使用，我们将在下文中把它作为一个速记符号，指的是一个接受类型参数的函数或类型。不要把本设计中使用的泛型一词与其他语言如`C++`、`C#`、`Java`或`Rust`中的相同术语混淆，它们有相似之处，但并不一样。 ~~~哈哈哈哈（^.^）


## Design

We will describe the complete design in stages based on simple
examples.

我们将根据简单的例子，分阶段描述完整的设计。



### Type parameters

Generic code is written using abstract data types that we call _type
parameters_.

泛型代码是使用抽象的数据类型编写的，我们称之为类型参数。

When running the generic code, the type parameters are replaced by
_type arguments_.

当运行泛型代码时，类型参数被类型实体参数所取代。

```go
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

采用这种方法，首先要做的决定是：应该如何声明类型参数`T`？

在像Go这样的语言中，我们希望每个标识符都能以某种方式被声明

Here we make a design decision: type parameters are similar to
ordinary non-type function parameters, and as such should be listed
along with other parameters.
However, type parameters are not the same as non-type parameters, so
although they appear in the list of parameters we want to distinguish
them.
That leads to our next design decision: we define an additional
optional parameter list describing type parameters.

这里我们做了一个设计决定：类型参数与普通的非类型函数参数相似，因此应该与其他参数一起列出。然而，类型参数与非类型参数不一样，所以尽管它们出现在参数列表中，我们还是要将它们区分开来。这导致了我们的下一个设计决定：我们定义了一个额外的可选参数列表来描述类型参数。

This type parameter list appears before the regular parameters.
To distinguish the type parameter list from the regular parameter
list, the type parameter list uses square brackets rather than
parentheses.
Just as regular parameters have types, type parameters have
meta-types, also known as constraints.
We will discuss the details of constraints later; for now, we will
just note that `any` is a valid constraint, meaning that any type is
permitted.

这个类型参数列表出现在常规参数的前面。为了区分类型参数列表和常规参数列表，类型参数列表使用方括号而不是小括号。

就像常规参数有类型一样，类型参数也有元类型，也被称为约束。

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

这就是说，在函数`Print`中，标识符`T`是一个类型参数，这个类型目前是未知的，但当函数被调用时将会被知道。任何意味着`T`可以是任何类型。如上所述，在描述普通非类型参数的类型时，类型参数可以作为一个类型使用。它也可以在函数的主体中作为一个类型使用。

Unlike regular parameter lists, in type parameter lists names are
required for the type parameters.
This avoids a syntactic ambiguity, and, as it happens, there is no
reason to ever omit the type parameter names.

与普通的参数列表不同，在类型参数列表中，类型参数的名字是必须的。这就避免了语法上的歧义，而且，恰好没有理由省略类型参数的名称。

Since `Print` has a type parameter, any call of `Print` must provide a
type argument.
Later we will see how this type argument can usually be deduced from
the non-type argument, by using [type inference](#Type-inference).
For now, we'll pass the type argument explicitly.
Type arguments are passed much like type parameters are declared: as a
separate list of arguments.
As with the type parameter list, the list of type arguments uses
square brackets.

由于 `Print` 有一个类型参数，因此任何 `Print` 调用都必须提供一个类型参数。

稍后我们将看到通常如何使用[类型推断](#Type-inference)从非类型参数中推导出此类型参数。

现在，我们将显式传递类型参数。类型参数的传递很像类型参数的声明：作为一个单独的参数列表。

与类型参数列表一样，类型参数列表使用方括号。

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

自然，同样的问题也出现在其他支持泛型编程的语言中。例如，在`C++`中，一个泛型函数（用`C++`的话来说，就是一个函数模板）可以调用泛型类型的值上的任何方法。也就是说，在`C++`方法中，调用`v.String()`是可以的。如果用一个没有`String`方法的类型参数来调用该函数，那么在用该类型参数编译调用`v.String`时就会报告错误。这些错误可能很冗长，因为在错误发生之前可能有几层泛型函数的调用，所有这些都必须被报告以了解出错的原因。

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

另一个原因是，`Go`的设计是为了支持规模化的编程。我们必须考虑这样的情况：泛型函数的定义（上面的`Stringify`）和对泛型函数的调用（没有显示，但可能在其他包中）相距甚远。一般来说，所有的泛型代码都希望类型参数能够满足某些要求。我们把这些要求称为约束（其他语言也有类似的想法，称为类型约束或特性约束或概念）。在这种情况下，约束条件是非常明显的：该类型必须有一个`String()`字符串方法。

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

这意味着约束必须对调用者传递的类型参数和泛型函数中的代码都设置限制。调用者只能传递满足约束条件的类型参数。`Generic Functions` 只能以约束条件允许的方式使用这些值。这是一条重要的规则，我们认为它应该适用于任何在`Go`中定义泛型编程的尝试：泛型代码只能使用其类型参数已经实现的接口约束的方法。



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

在我们进一步讨论约束之前，让我们简要说明当约束为 `any` 时会发生什么。如果泛型函数对类型参数使用 `any` 约束，如上述 `Print` 方法的情况，则该参数允许任何类型参数。通用函数可以对该类型参数的值使用的唯一操作是那些允许用于任何类型的值的操作。在上面的示例中，Print 函数声明了一个类型为类型参数 `T` 的变量 `v`，并将该变量传递给一个函数。

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

使用类型参数调用泛型函数类似于分配给接口类型的变量：类型参数必须实现类型参数的约束。

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

现在我们知道约束只是接口类型，我们可以解释什么是约束。如上所示，`any` 约束允许任何类型作为类型参数，并且只允许函数使用任何类型允许的操作。其接口类型是空接口：`interface{}`。所以我们可以将 `Print` 示例写成：

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

但是，每次编写不对其类型参数施加约束的泛型函数时都必须编写 `interface{}` 是很乏味的。
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

对于一个泛型函数，约束可以被认为是类型参数的类型：`a meta-type`。

如上所示，约束在类型参数列表中作为类型参数的`meta-type`出现。

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

单一类型参数 `T` 后跟适用于 `T` 的约束，在本例中为 `Stringer`。

### Multiple type parameters

Although the `Stringify` example uses only a single type parameter,
functions may have multiple type parameters.

尽管 `Stringify` 示例仅使用单个类型参数，但函数可能具有多个类型参数。

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

就像每个普通参数可以有自己的类型一样，每个类型参数也可以有自己的约束条件。

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

单个约束可用于多个类型参数，就像单个类型可用于多个非类型函数参数一样。该约束分别应用于每个类型参数。

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

一个类型的类型参数就像一个函数的类型参数一样。

Within the type definition, the type parameters may be used like any
other type.

在类型定义中，类型参数可以像其他类型一样使用。

To use a generic type, you must supply type arguments.
This is called _instantiation_.
The type arguments appear in square brackets, as usual.
When we instantiate a type by supplying type arguments for the type
parameters, we produce a type in which each use of a type parameter in
the type definition is replaced by the corresponding type argument.

要使用泛型类型，您必须提供类型参数。
这称为 `_instantiation_`。
像往常一样，类型参数出现在方括号中。
当我们通过为类型参数提供类型参数来实例化一个类型时，我们生成了一个类型，其中类型定义中的每个类型参数的使用都被相应的类型参数替换。

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
(注意：随着社区的深入讨论，我们对`Gopher`写代码的方式有了更多的了解，也许可以放宽这个规则，允许一些使用不同类型参数的情况)。

The type parameter of a generic type may have constraints other than
`any`.

泛型类型的类型参数可能具有除任何约束之外的约束。

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

虽然泛型类型的方法可以使用类型的参数，但方法本身可能没有额外的类型参数。在向方法添加类型参数有用的地方，人们将不得不编写一个适当参数化的顶级函数。
在[the issues
section](#No-parameterized-methods)对此有更多讨论。

### Operators

As we've seen, we are using interface types as constraints.
Interface types provide a set of methods, and nothing else.
This means that with what we've seen so far, the only thing that
generic functions can do with values of type parameters, other than
operations that are permitted for any type, is call methods.

正如我们所见，我们使用接口类型作为约束。接口类型提供一组方法，仅此而已。
这意味着，就我们目前所见，泛型函数唯一可以对类型参数值执行的操作是调用方法，而不是允许对任何类型执行的操作。

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

Every type has an associated type set.
The type set of a non-interface type `T` is simply the set `{T}`: a
set that contains just `T` itself.
The type set of an ordinary interface type is the set of all types
that declare all the methods of the interface.

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

It will be useful to construct the type set of an interface type by
looking at the elements of the interface.
This will produce the same result in a different way.
The elements of an interface can be either a method signature or an
embedded interface type.
Although a method signature is not a type, it's convenient to define a
type set for it: the set of all types that declare that method.
The type set of an embedded interface type `E` is simply that of `E`:
the set of all types that declare all the methods of `E`.

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

The same applies to embedded interface types.
For any two interface types `E1` and `E2`, the type set of `interface{
E1; E2 }` is the intersection of the type sets of `E1` and `E2`.

Therefore, the type set of an interface type is the intersection of
the type sets of the element of the interface.

#### Type sets of constraints

Now that we have described the type set of an interface type, we will
redefine what it means to satisfy the constraint.
Earlier we said that a type argument satisfies a constraint if it
implements the constraint.
Now we will say that a type argument satisfies a constraint if it is a
member of the constraint's type set.

For an ordinary interface type, one whose only elements are method
signatures and embedded ordinary interface types, the meaning is
exactly the same: the set of types that implement the interface type
is exactly the set of types that are in its type set.

We will now proceed to define additional elements that may appear in
an interface type that is used as a constraint, and define how those
additional elements can be used to further control the type set of the
constraint.

#### Constraint elements

The elements of an ordinary interface type are method signatures and
embedded interface types.
We propose permitting three additional elements that may be used in an
interface type used as a constraint.
If any of these additional elements are used, the interface type may
not be used as an ordinary type, but may only be used as a constraint.

## Author

* Ian Lance Taylor
* Robert Griesemer
* August 20, 2021