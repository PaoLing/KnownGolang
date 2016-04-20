
反射的规则 [The Laws Of Reflection](http://blog.golang.org/laws-of-reflection)
----------------------------------------------

##引言

反射在计算中是程序检查它自己结构的能力，特别是通过类型来检查。它是元编程的一种方式，It's also a great source of confusion.

此篇我们试图通过解释Go中的反射如何工作来使你明白。每种语言的反射模型都是不同的（许多语言还不是完全支持反射的），但此文章是关于Go的，故这篇文章余下的地方该把“反射”这个单词的意思称做“Go中的反射”。（so for the rest of this article the word "reflection" should be taken to mean "reflection in Go".）

##类型和接口

因为反射构建在类型系统，让我们通过温习Go中的类型开始。

Go是静态类型，每个变量都有一个静态类型，精确知道一个变量的类型并在编译期固定：`int, float32, *MyType, []byte` 等等，如果我们声明一个变量类型：

```go
type MyInt int

var i int
var j MyInt
```

变量 `i` 拥有 `int` 类型, 变量 `j` 拥有 `MyInt` 类型， 它们拥有明确的静态类型，虽然他们拥有相同的基础类型（int），但它们并不能越过强制转换来赋给另一个变量。（they cannot be assigned to one another without a conversion.）

类型的一个重要类别是接口（interface），接口有一套固定的方法集。一个接口变量可以储存任何具体（非接口）值，只要实现了接口所需的方法。一对已知的例子是 `os.Reader` 和 `io.Writer`， `Reader` 和 `Writer` 接口在 [io](http://golang.org/pkg/io/)包内定义。

```go
// Reader is the interface that wraps the basic Read method.
type Reader interface {
    Read(p []byte) (n int, err error)
}

// Writer is the interface that wraps the basic Write method.
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

任何实现了 `Read`(或`Write`) 方法签名的类型被认为是实现了`io.Reader`(或`io.Writer`)。讨论这些的意义是要说明，一个 `io.Reader` 接口类型的变量可以持有任何实现了 `Read` 方法的类型。

```go
var r io.Reader
r = os.Stdin
r = bufio.NewReader(r)
r = new(bytes.Buffer)
// and so on
```

It's important to be clear that whatever concrete value r may hold, `r`'s type is always io.Reader: Go is statically typed and the static type of r is io.Reader.

一个极其重要的例子是空接口：
`interface{}`

它表示空方法并满足所有值（It represents the empty set of methods and is satisfied by any value at all, ），因为任何值都有零个或多个方法。

有人说Go语言的接口是动态类型，但它是有误导性的。它们是静态类型：一个接口类型的变量总是同样的静态类型，即使在运行时（run-time）期间把值储存进接口变量可能会改变类型，值总是满足接口。

我们需要清楚反射和接口是亲密的一对。

##接口综述
Russ Cox写了篇叙述Go中的接口值的文章[detailed blog post](http://research.swtch.com/interfaces), 没必要在此重复全部内容，但简述一下合乎顺序的：

A variable of interface type stores a pair: the concrete value assigned to the variable, and that value's type descriptor. To be more precise, the value is the underlying concrete data item that implements the interface and the type describes the full type of that item. 举个例子：

```go
var r io.Reader
tty, err := os.OpenFile("/dev/tty", os.O_RDWR, 0)
if err != nil {
    return nil, err
}
r = tty
```

`r` 包含一对　`(value, type)` 对应 `(tty, *os.File)`, 注意：类型 `*os.File` 实现的方法不光只有 `Read` ；even though the interface value provides access only to the Read method, the value inside carries all the type information about that value. 这就是为什么我们可以这样做：

```go
var w io.Writer
w = r.(io.Writer)
```
这个赋值表达式是一个**类型断言**，在r内部也实现了 `io.Writer`所需的方法，所以我们可以赋值给w。在赋值后，w将包含`(tty, *os.File)`对，和r拥有的一样。接口的静态类型决定了在一个接口变量中可能挂钩的方法，即使具体值内部包含了更多的方法集。

我们可以继续这么做：

```go
var empty interface{}
empty = w
```

空接口值empty将再次包含同样的签名对 `(tty, *os.File)`。就如同信手拈来：一个空接口可以持有任何值，包含我们需要的值的所以信息。

（在这里我们没有使用断言是因为它知道w满足空接口的定义）。在此例中我们将一个值从 `Reader` 接口移动到 `Writer`接口，我们需要明确使用一个类型断言是因为 `Writer`的方法不是 `Reader` 的子集。
>Reader中只有`Read`方法，并没有`Write`方法，所以我们需要断言一下，获得实体类型`*os.File`，因为它即实现了Read又实现了Write方法。
>许式伟《Go语言编程》3.5.3节中有一个未使用断言，直接赋值导致失败的例子，就是因为这个问题。

一个重要的细节是一个接口内部总是有`(value, concrete type)`，不可以是`(value, interface type)`，接口无法含有接口值。

现在我们准备开始了解reflect。

##





