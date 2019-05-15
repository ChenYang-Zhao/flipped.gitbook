## Interface

### 1. Go 语言与鸭子类型的关系

Go 语言作为一门现代静态语言，是有后发优势的。它引入了动态语言的便利，同时又会进行静态语言的类型检查，写起来是非常 Happy 的。Go 采用了折中的做法：不要求类型显示地声明实现了某个接口，只要实现了相关的方法即可，编译器就能检测到。

来看个例子：

先定义一个接口，和使用此接口作为参数的函数：

```Go
type IGreeting interface {
    sayHello()
}

func sayHello(i IGreeting) {
    i.sayHello()
}

type Go struct {}
func (g Go) sayHello() {
    fmt.Println("Hi, I am GO!")
}

type PHP struct {}
func (p PHP) sayHello() {
    fmt.Println("Hi, I am PHP!")
}

func main() {
    golang := Go{}
    php := PHP{}

    sayHello(golang)
    sayHello(php)
}
```

输出：

```Shell
Hi, I am GO!
Hi, I am PHP!
```

在 main 函数中，调用调用 sayHello() 函数时，传入了 golang, php 对象，它们并没有显式地声明实现了 IGreeting 类型，只是实现了接口所规定的 sayHello() 函数。实际上，编译器在调用 sayHello() 函数时，会隐式地将 golang, php 对象转换成 IGreeting 类型，这也是静态语言的类型检查功能。

### 2. 值接者和指针接收者的区别

#### 方法

方法能给用户自定义的类型添加新的行为。它和函数的区别在于方法有一个接收者，给一个函数添加一个接收者，那么它就变成了方法。接收者可以是值接收者，也可以是指针接收者。

在调用方法的时候，值类型既可以调用值接收者的方法，也可以调用指针接收者的方法；指针类型既可以调用指针接收者的方法，也可以调用值接收者的方法。

也就是说，不管方法的接收者是什么类型，该类型的值和指针都可以调用，不必严格符合接收者的类型。

来看个例子：

```Go
package main

import "fmt"

type Person struct {
    age int
}

func (p Person) howOld() int {
    return p.age
}

func (p *Person) growUp() {
    p.age += 1
}

func main() {
    // qcrao 是值类型
    qcrao := Person{age: 18}

    // 值类型 调用接收者也是值类型的方法
    fmt.Println(qcrao.howOld())

    // 值类型 调用接收者是指针类型的方法
    qcrao.growUp()
    fmt.Println(qcrao.howOld())

    // ----------------------

    // stefno 是指针类型
    stefno := &Person{age: 100}

    // 指针类型 调用接收者是值类型的方法
    fmt.Println(stefno.howOld())

    // 指针类型 调用接收者也是指针类型的方法
    stefno.growUp()
    fmt.Println(stefno.howOld())
}

```

输出:

```Shell
18
19
100
101
```

调用了 growUp 函数后，不管调用者是值类型还是指针类型，它的 Age 值都改变了。

实际上，当类型和方法的接收者类型不同时，其实是编译器在背后做了一些工作，用一个表格来呈现：

-|值接收者|指针接收者
----|----|----
值类型调用者|方法会使用调用者的一个副本，类似于“传值”|使用值的引用来调用方法，上例中，qcrao.growUp() 实际上是 (&qcrao).growUp()
指针类型调用者|指针被解引用为值，上例中，stefno.howOld() 实际上是 (*stefno).howOld()|实际上也是“传值”，方法里的操作会影响到调用者，类似于指针传参，拷贝了一份指针

#### 值接收者和指针接收者

前面说过，不管接收者类型是值类型还是指针类型，都可以通过值类型或指针类型调用，这里面实际上通过语法糖起作用的。

先说结论：`实现了接收者是值类型的方法，相当于自动实现了接收者是指针类型的方法；而实现了接收者是指针类型的方法，不会自动生成对应接收者是值类型的方法`。

来看一个例子，就会完全明白：

```Go
package main

import "fmt"

type coder interface {
    code()
    debug()
}

type Gopher struct {
    language string
}

func (p Gopher) code() {
    fmt.Printf("I am coding %s language\n", p.language)
}

func (p *Gopher) debug() {
    fmt.Printf("I am debuging %s language\n", p.language)
}

func main() {
    var c coder = &Gopher{"Go"}
    c.code()
    c.debug()
}
```

上述代码里定义了一个接口 coder，接口定义了两个函数：code() and debug(); 接着定义了一个结构体 Gopher，它实现了两个方法，一个值接收者，一个指针接收者。最后，我们在 main 函数里通过接口类型的变量调用了定义的两个函数。

运行结果：

```Shell
I am coding Go language
I am debuging Go language
```

但是如果我们把 main 函数的第一条语句换一下：

```Go
func main() {
    var c coder = Gopher{"Go"}
    c.code()
    c.debug()
}
```

运行一下，报错：

```Shell
./main.go:24:6: cannot use Programmer literal (type Programmer) as type coder in assignment:
    Programmer does not implement coder (debug method has pointer receiver)
```

看出这两处代码的差别了吗？第一次是将 &Gopher 赋给了 coder；第二次则是将 Gopher 赋给了 coder。

第二次报错是说，Gopher 没有实现 coder。很明显了吧，因为 Gopher 类型并没有实现 debug 方法；表面上看， *Gopher 类型也没有实现 code 方法，但是因为 Gopher 类型实现了 code 方法，所以让 *Gopher 类型自动拥有了 code 方法。

当然，上面的说法有一个简单的解释：接收者是指针类型的方法，很可能在方法中会对接收者的属性进行更改操作，从而影响接收者；而对于接收者是值类型的方法，在方法中不会对接收者本身产生影响。

所以，当实现了一个接收者是值类型的方法，就可以自动生成一个接收者是对应指针类型的方法，因为两者都不会影响接收者。但是，当实现了一个接收者是指针类型的方法，如果此时自动生成一个接收者是值类型的方法，原本期望对接收者的改变（通过指针实现），现在无法实现，因为值类型会产生一个拷贝，不会真正影响调用者。

最后，只要记住下面这点就可以了： 

> 如果实现了接收者是值类型的方法，会隐含地也实现了接收者是指针类型的方法。

如果方法的接收者是值类型，无论调用者是对象还是对象指针，修改的都是对象的副本，不影响调用者；如果方法的接收者是指针类型，则调用者修改的是指针指向的对象本身。

### 3.iface 和 eface 的区别是什么

iface 和 eface 都是 Go 中描述接口的底层结构体，区别在于 iface 描述的接口包含方法，而 eface 则是不包含任何方法的空接口：interface{}。

从源码层面看一下：

```Go
type iface struct {
    tab  *itab
    data unsafe.Pointer
}

type itab struct {
    inter  *interfacetype //接口类型
    _type  *_type //实体类型
    link   *itab
    hash   uint32 // copy of _type.hash. Used for type switches.
    bad    bool   // type does not implement interface
    inhash bool   // has this itab been added to hash?
    unused [2]byte
    fun    [1]uintptr // variable sized //实体类型对接口具体实现的方法地址 
}
```

iface 内部维护两个指针，tab 指向一个 itab 实体， 它表示接口的类型以及赋给这个接口的实体类型。data 则指向接口具体的值，一般而言是一个指向堆内存的指针。

再来仔细看一下 itab 结构体：_type 字段描述了实体的类型，包括内存对齐方式，大小等；inter 字段则描述了接口的类型。fun 字段放置和接口方法对应的具体数据类型的方法地址，实现接口调用方法的动态分派，一般在每次给接口赋值发生转换时会更新此表，或者直接拿缓存的 itab。这里只会列出实体类型和接口相关的方法，实体类型的其他方法并不会出现在这里。如果你学过 C++ 的话，这里可以类比虚函数的概念。

再看一下 interfacetype 类型，它描述的是接口的类型：

```Go
type interfacetype struct {
    typ     _type
    pkgpath name
    mhdr    []imethod
}
```
可以看到，它包装了 _type 类型，_type 实际上是描述 Go 语言中各种数据类型的结构体。我们注意到，这里还包含一个 mhdr 字段，表示接口所定义的函数列表， pkgpath 记录定义了接口的包名。

这里通过一张图来看下 iface 结构体的全貌：

![](../../imgs/component/interface-iface.png)

接着来看一下 eface 的源码：

```Go
type eface struct {
    _type *_type
    data  unsafe.Pointer
}
```

相比 iface，eface 就比较简单了。只维护了一个 _type 字段，表示空接口所承载的具体的实体类型。data 描述了具体的值。

我们来看个例子：

```Go
package main

import "fmt"

func main() {
    x := 200
    var any interface{} = x
    fmt.Println(any)

    g := Gopher{"Go"}
    var c coder = g
    fmt.Println(c)
}

type coder interface {
    code()
    debug()
}

type Gopher struct {
    language string
}

func (p Gopher) code() {
    fmt.Printf("I am coding %s language\n", p.language)
}

func (p Gopher) debug() {
    fmt.Printf("I am debuging %s language\n", p.language)
}
```

执行命令，打印出汇编语言：

```Shell
go tool compile -S ./src/main.go
```

可以看到，main 函数里调用了两个函数：

```Go
func convT2E64(t *_type, elem unsafe.Pointer) (e eface)
func convT2I(tab *itab, elem unsafe.Pointer) (i iface)
```

上面两个函数的参数和 iface 及 eface 结构体的字段是可以联系起来的：两个函数都是将参数组装一下，形成最终的接口。

我们最后再来看下 _type 结构体：

```Go
type _type struct {
    // 类型大小
    size       uintptr
    ptrdata    uintptr
    // 类型的 hash 值
    hash       uint32
    // 类型的 flag，和反射相关
    tflag      tflag
    // 内存对齐相关
    align      uint8
    fieldalign uint8
    // 类型的编号，有bool, slice, struct 等等等等
    kind       uint8
    alg        *typeAlg
    // gc 相关
    gcdata    *byte
    str       nameOff
    ptrToThis typeOff
}
```
Go 语言各种数据类型都是在 _type 字段的基础上，增加一些额外的字段来进行管理的：

```Go
type arraytype struct {
    typ   _type
    elem  *_type
    slice *_type
    len   uintptr
}

type chantype struct {
    typ  _type
    elem *_type
    dir  uintptr
}

type slicetype struct {
    typ  _type
    elem *_type
}

type structtype struct {
    typ     _type
    pkgPath name
    fields  []structfield
}
```

这些数据类型的结构体定义，是反射实现的基础。

### 4. 接口的动态类型和动态值

从源码里可以看到：iface包含两个字段：tab 是接口表指针，指向类型信息；data 是数据指针，则指向具体的数据。它们分别被称为动态类型和动态值。

#### 【引申1】接口类型和 nil 作比较

接口值的零值是指`动态类型`和`动态值`都为 nil。当仅且当这两部分的值都为 nil 的情况下，这个接口值就才会被认为 接口值 == nil。

来看个例子：

```Go
package main

import "fmt"

type Coder interface {
    code()
}

type Gopher struct {
    name string
}

func (g Gopher) code() {
    fmt.Printf("%s is coding\n", g.name)
}

func main() {
    var c Coder
    fmt.Println(c == nil)
    fmt.Printf("c: %T, %v\n", c, c)

    var g *Gopher
    fmt.Println(g == nil)

    c = g
    fmt.Println(c == nil)
    fmt.Printf("c: %T, %v\n", c, c)
}
```

输出：

```Shell
true
c: <nil>, <nil>
true
false
c: *main.Gopher, <nil>
```

一开始，c 的 动态类型和动态值都为 nil，g 也为 nil，当把 g 赋值给 c 后，c 的动态类型变成了 *main.Gopher，仅管 c 的动态值仍为 nil，但是当 c 和 nil 作比较的时候，结果就是 false 了。

#### 【引申2】 来看一个例子，看一下它的输出：

```Go
package main

import "fmt"

type MyError struct {}

func (i MyError) Error() string {
    return "MyError"
}

func main() {
    err := Process()
    fmt.Println(err)

    fmt.Println(err == nil)
}

func Process() error {
    var err *MyError = nil
    return err
}
```

函数运行结果：

```Shell
<nil>
false
```

这里先定义了一个 MyError 结构体，实现了 Error 函数，也就实现了 error 接口。Process 函数返回了一个 error 接口，这块隐含了类型转换。所以，虽然它的值是 nil，其实它的类型是 *MyError，最后和 nil 比较的时候，结果为 false。

#### 【引申3】如何打印出接口的动态类型和值？

```Go
package main

import (
    "unsafe"
    "fmt"
)

type iface struct {
    itab, data uintptr
}

func main() {
    var a interface{} = nil

    var b interface{} = (*int)(nil)

    x := 5
    var c interface{} = (*int)(&x)

    ia := *(*iface)(unsafe.Pointer(&a))
    ib := *(*iface)(unsafe.Pointer(&b))
    ic := *(*iface)(unsafe.Pointer(&c))

    fmt.Println(ia, ib, ic)

    fmt.Println(*(*int)(unsafe.Pointer(ic.data)))
}
```

代码里直接定义了一个 iface 结构体，用两个指针来描述 itab 和 data，之后将 a, b, c 在内存中的内容强制解释成我们自定义的 iface。最后就可以打印出动态类型和动态值的地址。

运行结果如下：

```Shell
{0 0} {17426912 0} {17426912 842350714568}
5
```

a 的动态类型和动态值的地址均为 0，也就是 nil；b 的动态类型和 c 的动态类型一致，都是 *int；最后，c 的动态值为 5。

### 5. 编译器自动检测类型是否实现接口

经常看到一些开源库里会有一些类似下面这种奇怪的用法：

```Go
var _ io.Writer = (*myWriter)(nil)
```

这时候会有点懵，不知道作者想要干什么，实际上这就是此问题的答案。编译器会由此检查 *myWriter 类型是否实现了 io.Writer 接口。

来看一个例子：

```Go
package main

import "io"

type myWriter struct {

}

func (w myWriter) Write(p []byte) (n int, err error) {
    return
}

func main() {
    // 检查 *myWriter 类型是否实现了 io.Writer 接口
    var _ io.Writer = (*myWriter)(nil)

    // 检查 myWriter 类型是否实现了 io.Writer 接口
    var _ io.Writer = myWriter{}
}
```

### 6. 接口的构造过程是怎样的

####【引申1】 如何打印出接口类型的 Hash 值？

```Go
type iface struct {
    tab  *itab
    data unsafe.Pointer
}
type itab struct {
    inter uintptr
    _type uintptr
    link uintptr
    hash  uint32
    _     [4]byte
    fun   [1]uintptr
}

func main() {
    var qcrao = Person(Student{age: 18})

    iface := (*iface)(unsafe.Pointer(&qcrao))
    fmt.Printf("iface.tab.hash = %#x\n", iface.tab.hash)
}
```

定义了一个山寨版的 iface 和 itab，说它山寨是因为 itab 里的一些关键数据结构都不具体展开了，比如 _type，对比一下正宗的定义就可以发现，但是山寨版依然能工作，因为 _type 就是一个指针而已嘛。

在 main 函数里，先构造出一个接口对象 qcrao，然后强制类型转换，最后读取出 hash 值，非常妙!

### 7. 类型转换和断言的区别

我们知道，Go 语言中不允许隐式类型转换，也就是说 = 两边，不允许出现类型不相同的变量。

类型转换、类型断言本质都是把一个类型转换成另外一个类型。不同之处在于，类型断言是对接口变量进行的操作。

#### 类型转换

对于类型转换而言，转换前后的两个类型要相互兼容才行。类型转换的语法为：

> <结果类型> := <目标类型> ( <表达式> )

#### 断言

 前面说过，因为空接口 interface{} 没有定义任何函数，因此 Go 中所有类型都实现了空接口。当一个函数的形参是 interface{}，那么在函数中，需要对形参进行断言，从而得到它的真实类型。

断言的语法为：

> <目标类型的值>，<布尔参数> := <表达式>.( 目标类型 ) // 安全类型断言 
> <目标类型的值> := <表达式>.( 目标类型 )　　//非安全类型断言

类型转换和类型断言有些相似，不同之处，在于类型断言是对接口进行的操作。

断言其实还有另一种形式，就是用在利用 switch 语句判断接口的类型。每一个 case 会被顺序地考虑。当命中一个 case 时，就会执行 case 中的语句，因此 case 语句的顺序是很重要的，因为很有可能会有多个 case 匹配的情况。

```Go
func main() {
    //var i interface{} = new(Student)
    //var i interface{} = (*Student)(nil)
    var i interface{}

    fmt.Printf("%p %v\n", &i, i)

    judge(i)
}

func judge(v interface{}) {
    fmt.Printf("%p %v\n", &v, v)

    switch v := v.(type) {
    case nil:
        fmt.Printf("%p %v\n", &v, v)
        fmt.Printf("nil type[%T] %v\n", v, v)

    case Student:
        fmt.Printf("%p %v\n", &v, v)
        fmt.Printf("Student type[%T] %v\n", v, v)

    case *Student:
        fmt.Printf("%p %v\n", &v, v)
        fmt.Printf("*Student type[%T] %v\n", v, v)

    default:
        fmt.Printf("%p %v\n", &v, v)
        fmt.Printf("unknow\n")
    }
}

type Student struct {
    Name string
    Age int
}
```

main 函数里有三行不同的声明，每次运行一行，注释另外两行，得到三组运行结果：

```Shell
// --- var i interface{} = new(Student)
0xc4200701b0 [Name: ], [Age: 0]
0xc4200701d0 [Name: ], [Age: 0]
0xc420080020 [Name: ], [Age: 0]
*Student type[*main.Student] [Name: ], [Age: 0]

// --- var i interface{} = (*Student)(nil)
0xc42000e1d0 <nil>
0xc42000e1f0 <nil>
0xc42000c030 <nil>
*Student type[*main.Student] <nil>

// --- var i interface{}
0xc42000e1d0 <nil>
0xc42000e1e0 <nil>
0xc42000e1f0 <nil>
nil type[<nil>] <nil>
```

对于第一行语句：

```Go
var i interface{} = new(Student)
```

i 是一个 *Student 类型，匹配上第三个 case，从打印的三个地址来看，这三处的变量实际上都是不一样的。在 main 函数里有一个局部变量 i；调用函数时，实际上是复制了一份参数，因此函数里又有一个变量 v，它是 i 的拷贝；断言之后，又生成了一份新的拷贝。所以最终打印的三个变量的地址都不一样。

对于第二行语句：

```Go
var i interface{} = (*Student)(nil)
```

这里想说明的其实是 i 在这里动态类型是 (*Student), 数据为 nil，它的类型并不是 nil，它与 nil 作比较的时候，得到的结果也是 false。

最后一行语句：

```Go
var i interface{}
```

这回 i 才是 nil 类型。

【引申1】 fmt.Println 函数的参数是 interface。对于内置类型，函数内部会用穷举法，得出它的真实类型，然后转换为字符串打印。而对于自定义类型，首先确定该类型是否实现了 String() 方法，如果实现了，则直接打印输出 String() 方法的结果；否则，会通过反射来遍历对象的成员进行打印。

再来看一个简短的例子，比较简单，不要紧张：

```Go
package main

import "fmt"

type Student struct {
    Name string
    Age int
}

func main() {
    var s = Student{
        Name: "qcrao",
        Age: 18,
    }

    fmt.Println(s)
}
```

因为 Student 结构体没有实现 String() 方法，所以 fmt.Println 会利用反射挨个打印成员变量：

```Shell
{qcrao 18}
```

增加一个 String() 方法的实现：

```Go
func (s Student) String() string {
    return fmt.Sprintf("[Name: %s], [Age: %d]", s.Name, s.Age)
}
```

打印结果：

[Name: qcrao], [Age: 18]
按照我们自定义的方法来打印了。

【引申2】 针对上面的例子，如果改一下：

```Go
func (s *Student) String() string {
    return fmt.Sprintf("[Name: %s], [Age: %d]", s.Name, s.Age)
}
```

注意看两个函数的接受者类型不同，现在 Student 结构体只有一个接受者类型为 指针类型 的 String() 函数，打印结果：

```Shell
{qcrao 18}
```

为什么？

> 类型 T 只有接受者是 T 的方法；而类型 *T 拥有接受者是 T 和 *T 的方法。语法上 T 能直接调 *T 的方法仅仅是 Go 的语法糖。

### 8. 接口转换的原理

通过前面提到的 iface 的源码可以看到，实际上它包含接口的类型 interfacetype 和 实体类型的类型 _type，这两者都是 iface 的字段 itab 的成员。也就是说生成一个 itab 同时需要接口的类型和实体的类型。

> <interface 类型， 实体类型> ->itable

当判定一种类型是否满足某个接口时，Go 使用类型的方法集和接口所需要的方法集进行匹配，如果类型的方法集完全包含接口的方法集，则可认为该类型实现了该接口。

例如某类型有 m 个方法，某接口有 n 个方法，则很容易知道这种判定的时间复杂度为 O(mn)，Go 会对方法集的函数按照函数名的字典序进行排序，所以实际的时间复杂度为 O(m+n)。

这里我们来探索将一个接口转换给另外一个接口背后的原理，当然，能转换的原因必然是类型兼容。

直接来看一个例子：

```Go
package main

import "fmt"

type coder interface {
    code()
    run()
}

type runner interface {
    run()
}

type Gopher struct {
    language string
}

func (g Gopher) code() {
    return
}

func (g Gopher) run() {
    return
}

func main() {
    var c coder = Gopher{}

    var r runner
    r = c
    fmt.Println(c, r)
}
```

简单解释下上述代码：定义了两个 interface: coder 和 runner。定义了一个实体类型 Gopher，类型 Gopher 实现了两个方法，分别是 run() 和 code()。main 函数里定义了一个接口变量 c，绑定了一个 Gopher 对象，之后将 c 赋值给另外一个接口变量 r 。赋值成功的原因是 c 中包含 run() 方法。这样，两个接口变量完成了转换。

执行命令：

```Shell
go tool compile -S ./src/main.go
```

得到 main 函数的汇编命令，可以看到： r = c 这一行语句实际上是调用了 runtime.convI2I(SB)，也就是 convI2I 函数，从函数名来看，就是将一个 interface 转换成另外一个 interface，看下它的源代码：

```Go
func convI2I(inter *interfacetype, i iface) (r iface) {
    tab := i.tab
    if tab == nil {
        return
    }
    if tab.inter == inter {
        r.tab = tab
        r.data = i.data
        return
    }
    r.tab = getitab(inter, tab._type, false)
    r.data = i.data
    return
}
```

代码比较简单，函数参数 inter 表示接口类型，i 表示绑定了实体类型的接口，r 则表示接口转换了之后的新的 iface。通过前面的分析，我们又知道， iface 是由 tab 和 data 两个字段组成。所以，实际上 convI2I 函数真正要做的事，找到新 interface 的 tab 和 data，就大功告成了。

我们还知道，tab 是由接口类型 interfacetype 和 实体类型 _type。所以最关键的语句是 r.tab = getitab(inter, tab._type, false)。

因此，重点来看下 getitab 函数的源码，只看关键的地方：

```Go
func getitab(inter *interfacetype, typ *_type, canfail bool) *itab {
    // ……

    // 根据 inter, typ 计算出 hash 值
    h := itabhash(inter, typ)

    // look twice - once without lock, once with.
    // common case will be no lock contention.
    var m *itab
    var locked int
    for locked = 0; locked < 2; locked++ {
        if locked != 0 {
            lock(&ifaceLock)
        }

        // 遍历哈希表的一个 slot
        for m = (*itab)(atomic.Loadp(unsafe.Pointer(&hash[h]))); m != nil; m = m.link {

            // 如果在 hash 表中已经找到了 itab（inter 和 typ 指针都相同）
            if m.inter == inter && m._type == typ {
                // ……

                if locked != 0 {
                    unlock(&ifaceLock)
                }
                return m
            }
        }
    }

    // 在 hash 表中没有找到 itab，那么新生成一个 itab
    m = (*itab)(persistentalloc(unsafe.Sizeof(itab{})+uintptr(len(inter.mhdr)-1)*sys.PtrSize, 0, &memstats.other_sys))
    m.inter = inter
    m._type = typ

    // 添加到全局的 hash 表中
    additab(m, true, canfail)
    unlock(&ifaceLock)
    if m.bad {
        return nil
    }
    return m
}
```

简单总结一下：getitab 函数会根据 interfacetype 和 _type 去全局的 itab 哈希表中查找，如果能找到，则直接返回；否则，会根据给定的 interfacetype 和 _type 新生成一个 itab，并插入到 itab 哈希表，这样下一次就可以直接拿到 itab。

这里查找了两次，并且第二次上锁了，这是因为如果第一次没找到，在第二次仍然没有找到相应的 itab 的情况下，需要新生成一个，并且写入哈希表，因此需要加锁。这样，其他协程在查找相同的 itab 并且也没有找到时，第二次查找时，会被挂住，之后，就会查到第一个协程写入哈希表的 itab。

再来看一下 additab 函数的代码：

```Go
// 检查 _type 是否符合 interface_type 并且创建对应的 itab 结构体 将其放到 hash 表中
func additab(m *itab, locked, canfail bool) {
    inter := m.inter
    typ := m._type
    x := typ.uncommon()

    // both inter and typ have method sorted by name,
    // and interface names are unique,
    // so can iterate over both in lock step;
    // the loop is O(ni+nt) not O(ni*nt).
    // 
    // inter 和 typ 的方法都按方法名称进行了排序
    // 并且方法名都是唯一的。所以循环的次数是固定的
    // 只用循环 O(ni+nt)，而非 O(ni*nt)
    ni := len(inter.mhdr)
    nt := int(x.mcount)
    xmhdr := (*[1 << 16]method)(add(unsafe.Pointer(x), uintptr(x.moff)))[:nt:nt]
    j := 0
    for k := 0; k < ni; k++ {
        i := &inter.mhdr[k]
        itype := inter.typ.typeOff(i.ityp)
        name := inter.typ.nameOff(i.name)
        iname := name.name()
        ipkg := name.pkgPath()
        if ipkg == "" {
            ipkg = inter.pkgpath.name()
        }
        for ; j < nt; j++ {
            t := &xmhdr[j]
            tname := typ.nameOff(t.name)
            // 检查方法名字是否一致
            if typ.typeOff(t.mtyp) == itype && tname.name() == iname {
                pkgPath := tname.pkgPath()
                if pkgPath == "" {
                    pkgPath = typ.nameOff(x.pkgpath).name()
                }
                if tname.isExported() || pkgPath == ipkg {
                    if m != nil {
                        // 获取函数地址，并加入到itab.fun数组中
                        ifn := typ.textOff(t.ifn)
                        *(*unsafe.Pointer)(add(unsafe.Pointer(&m.fun[0]), uintptr(k)*sys.PtrSize)) = ifn
                    }
                    goto nextimethod
                }
            }
        }
        // ……

        m.bad = true
        break
    nextimethod:
    }
    if !locked {
        throw("invalid itab locking")
    }

    // 计算 hash 值
    h := itabhash(inter, typ)
    // 加到Hash Slot链表中
    m.link = hash[h]
    m.inhash = true
    atomicstorep(unsafe.Pointer(&hash[h]), unsafe.Pointer(m))
}
```

additab 会检查 itab 持有的 interfacetype 和 _type 是否符合，就是看 _type 是否完全实现了 interfacetype 的方法，也就是看两者的方法列表重叠的部分就是 interfacetype 所持有的方法列表。注意到其中有一个双层循环，乍一看，循环次数是 ni * nt，但由于两者的函数列表都按照函数名称进行了排序，因此最终只执行了 ni + nt 次，代码里通过一个小技巧来实现：第二层循环并没有从 0 开始计数，而是从上一次遍历到的位置开始。

### 9. 如何用 interface 实现多态

Go 语言并没有设计诸如虚函数、纯虚函数、继承、多重继承等概念，但它通过接口却非常优雅地支持了面向对象的特性。

多态是一种运行期的行为，它有以下几个特点：

> 一种类型具有多种类型的能力
> 允许不同的对象对同一消息做出灵活的反应
> 以一种通用的方式对待个使用的对象
> 非动态语言必须通过继承和接口的方式来实现

看一个实现了多态的代码例子：

```Go
package main

import "fmt"

func main() {
    qcrao := Student{age: 18}
    whatJob(&qcrao)

    growUp(&qcrao)
    fmt.Println(qcrao)

    stefno := Programmer{age: 100}
    whatJob(stefno)

    growUp(stefno)
    fmt.Println(stefno)
}

func whatJob(p Person) {
    p.job()
}

func growUp(p Person) {
    p.growUp()
}

type Person interface {
    job()
    growUp()
}

type Student struct {
    age int
}

func (p Student) job() {
    fmt.Println("I am a student.")
    return
}

func (p *Student) growUp() {
    p.age += 1
    return
}

type Programmer struct {
    age int
}

func (p Programmer) job() {
    fmt.Println("I am a programmer.")
    return
}

func (p Programmer) growUp() {
    // 程序员老得太快 ^_^
    p.age += 10
    return
}
```

### 参考

- [深度解密Go语言之关于 interface 的10个问题](https://gocn.vip/article/1717)