## Panic and Recover in Go

### 关于panic和recover的问题：

1. panic时发生了什么，程序为什么会退出？
2. recover为什么可以捕获到panic?
3. 为什么defer只可以捕获本goroutine的panic?
4. recover必须写到defer中吗？
5. recover后程序从哪里继续执行，为什么？

带着这些问题，一起来看看Golang中panic及recover的处理流程；

先看下面最简的跑出panic代码:

```Go
package main

func main() {
    panic("throw a panic")
}
```

我们用panic()方法抛出了一个panic，这段代码的运行结果:

```Shell
panic: throw a panic

goroutine 1 [running]:
main.main()
	/Users/zhaochenyang3/CodeSpace/GoSource/JvirtGoPath/src/chy.space/tests/test_panic_recover.go:4 +0x39
exit status 2
```

#### panic具体发生了什么？

执行 `go tool compile -S main.go` 看一下结果

```Shell
"".main STEXT size=66 args=0x0 locals=0x18
	0x0000 00000 (test_panic_recover.go:3)	TEXT	"".main(SB), $24-0
	0x0000 00000 (test_panic_recover.go:3)	MOVQ	(TLS), CX
	0x0009 00009 (test_panic_recover.go:3)	CMPQ	SP, 16(CX)
	0x000d 00013 (test_panic_recover.go:3)	JLS	59
	0x000f 00015 (test_panic_recover.go:3)	SUBQ	$24, SP
	0x0013 00019 (test_panic_recover.go:3)	MOVQ	BP, 16(SP)
	0x0018 00024 (test_panic_recover.go:3)	LEAQ	16(SP), BP
	0x001d 00029 (test_panic_recover.go:3)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x001d 00029 (test_panic_recover.go:3)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x001d 00029 (test_panic_recover.go:3)	FUNCDATA	$3, gclocals·9fb7f0986f647f17cb53dda1484e0f7a(SB)
	0x001d 00029 (test_panic_recover.go:4)	PCDATA	$2, $1
	0x001d 00029 (test_panic_recover.go:4)	PCDATA	$0, $0
	0x001d 00029 (test_panic_recover.go:4)	LEAQ	type.string(SB), AX
	0x0024 00036 (test_panic_recover.go:4)	PCDATA	$2, $0
	0x0024 00036 (test_panic_recover.go:4)	MOVQ	AX, (SP)
	0x0028 00040 (test_panic_recover.go:4)	PCDATA	$2, $1
	0x0028 00040 (test_panic_recover.go:4)	LEAQ	"".statictmp_0(SB), AX
	0x002f 00047 (test_panic_recover.go:4)	PCDATA	$2, $0
	0x002f 00047 (test_panic_recover.go:4)	MOVQ	AX, 8(SP)
	0x0034 00052 (test_panic_recover.go:4)	CALL	runtime.gopanic(SB) //调用gopanic
	0x0039 00057 (test_panic_recover.go:4)	UNDEF
	0x003b 00059 (test_panic_recover.go:4)	NOP
	0x003b 00059 (test_panic_recover.go:3)	PCDATA	$0, $-1
	0x003b 00059 (test_panic_recover.go:3)	PCDATA	$2, $-1
	0x003b 00059 (test_panic_recover.go:3)	CALL	runtime.morestack_noctxt(SB)
	0x0040 00064 (test_panic_recover.go:3)	JMP	0
```

这里看到，`panic`关键词最终执行了`runtime.gopanic`方法，下面是这个func简化后的代码：

```Go
func gopanic(e interface{}){ //e是panic时传进来的参数
    gp := getg() //获取当前Goroutine

    //初始化一个_panic 对象，加入到当前goroutine的panic队列的最前面
    var p _panic
    p.arg = e
    p.link = gp._panic
    gp._panic = (*_panic)noescape((unsafe.Pointer(&p)))

    for {
        d := gp._defer //获取goroutine的defer队列
        if d == nil{
            break
        }
        if d.started {
            gp._defer = d.link
            freedefer(d)
            continue
        }
        d.started = true
        d._panic =  (*_panic)noescape((unsafe.Pointer(&p)))
        reflectcall(nil, unsafe.Pointer(d.fn), deferArgs(d), uint32(d.siz), uint32(d.siz)) //调用defer中的方法
        d._panic = nil
        d.fn = nil
        gp._defer = d.link
        pc := d.pc //从defer对象中获取伪寄存器的pc和方法栈帧，用户recover时恢复
        sp := unsafe.Pointer(d.sp)
        freedefer(p)

        //如果当前panic被recover，调用下面的代码
        if p.recoverd {
            gp._panic = p.link
            for gp._panic != nil && gp.panic.aborted{
                gp._panic = gp._panic.link
            }
            if gp._panic == nil{
                gp.sig = 0
            }
            gp.sigcode0 = uintptr(sp)//恢复伪寄存器的地址及栈帧，跳转到recover后要执行的下一行代码
            gp.sigcode1 = pc

            mcall(recovery) //从panic中恢复，继续执行后面的代码,这个方法不会return
        }
    }
    preprintpanics(gp._panic) //准备panci要打印的错误信息
    fatalpanic(gp._panic) //打印panic错误并退出程序
}
```
从上面的代码可以看出，如果panic没有被捕获，最终调用了fatalpanic()方法; 那程序为什么会退出呢，我们来看一下fatalpanic的代码：

```Go
func fatalpanic(msgs *_panic) {
    sp := getcallersp()
    pc := getcallerpc()
    gp := getg()
    systemstack(func{
        if startpanic_m() && msgs != nil { //startpanic_m 产生一个无法恢复的panic
            printpanics(msgs) //打印panic信息
        }
        docrash := dopanic_m(gp,pc, sp)
    }
    systemstack(func(){
        exit(2) //退出程序
    })
}
```

可以看到fatalpanic打印了panic的详细信息后，调用了`exit`退出了程序；panic后程序退出的问题到这里就解答了

#### recover如何恢复了程序？

修改一下上面panic的代码，加入recover：

```Go
package main

func main() {
	defer func() {
		if r := recover(); r != nil {
			print("panic be recoved")
		}
	}()
	panic("throw a panic")
}
```

执行一下，输出如下：

```Shell
panic be recoved
```

其实，上面的代码写成下面这样就可以:

```Go
package main

func main() {
	defer func() {
		recover()
		print("panic be recoved")
	}()
	panic("throw a panic")
}

那么，recover是如何恢复程序，让程序继续执行的呢?
同样的执行`go tool compile -S main.go` 看一下：

```Shell
"".main STEXT size=110 args=0x0 locals=0x18
	0x0000 00000 (test_panic_recover.go:3)	TEXT	"".main(SB), $24-0
	0x0000 00000 (test_panic_recover.go:3)	MOVQ	(TLS), CX
	0x0009 00009 (test_panic_recover.go:3)	CMPQ	SP, 16(CX)
	0x000d 00013 (test_panic_recover.go:3)	JLS	103
	0x000f 00015 (test_panic_recover.go:3)	SUBQ	$24, SP
	0x0013 00019 (test_panic_recover.go:3)	MOVQ	BP, 16(SP)
	0x0018 00024 (test_panic_recover.go:3)	LEAQ	16(SP), BP
	0x001d 00029 (test_panic_recover.go:3)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x001d 00029 (test_panic_recover.go:3)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x001d 00029 (test_panic_recover.go:3)	FUNCDATA	$3, gclocals·9fb7f0986f647f17cb53dda1484e0f7a(SB)
	0x001d 00029 (test_panic_recover.go:4)	PCDATA	$2, $0
	0x001d 00029 (test_panic_recover.go:4)	PCDATA	$0, $0
	0x001d 00029 (test_panic_recover.go:4)	MOVL	$0, (SP)
	0x0024 00036 (test_panic_recover.go:4)	PCDATA	$2, $1
	0x0024 00036 (test_panic_recover.go:4)	LEAQ	"".main.func1·f(SB), AX
	0x002b 00043 (test_panic_recover.go:4)	PCDATA	$2, $0
	0x002b 00043 (test_panic_recover.go:4)	MOVQ	AX, 8(SP)
	0x0030 00048 (test_panic_recover.go:4)	CALL	runtime.deferproc(SB)
	0x0035 00053 (test_panic_recover.go:4)	TESTL	AX, AX
	0x0037 00055 (test_panic_recover.go:4)	JNE	87
	0x0039 00057 (test_panic_recover.go:8)	PCDATA	$2, $1
	0x0039 00057 (test_panic_recover.go:8)	LEAQ	type.string(SB), AX
	0x0040 00064 (test_panic_recover.go:8)	PCDATA	$2, $0
	0x0040 00064 (test_panic_recover.go:8)	MOVQ	AX, (SP)
	0x0044 00068 (test_panic_recover.go:8)	PCDATA	$2, $1
	0x0044 00068 (test_panic_recover.go:8)	LEAQ	"".statictmp_0(SB), AX
	0x004b 00075 (test_panic_recover.go:8)	PCDATA	$2, $0
	0x004b 00075 (test_panic_recover.go:8)	MOVQ	AX, 8(SP)
	0x0050 00080 (test_panic_recover.go:8)	CALL	runtime.gopanic(SB)
	0x0055 00085 (test_panic_recover.go:8)	UNDEF
	0x0057 00087 (test_panic_recover.go:4)	XCHGL	AX, AX
	0x0058 00088 (test_panic_recover.go:4)	CALL	runtime.deferreturn(SB)
	0x005d 00093 (test_panic_recover.go:4)	MOVQ	16(SP), BP
	0x0062 00098 (test_panic_recover.go:4)	ADDQ	$24, SP
	0x0066 00102 (test_panic_recover.go:4)	RET
	0x0067 00103 (test_panic_recover.go:4)	NOP
	0x0067 00103 (test_panic_recover.go:3)	PCDATA	$0, $-1
	0x0067 00103 (test_panic_recover.go:3)	PCDATA	$2, $-1
	0x0067 00103 (test_panic_recover.go:3)	CALL	runtime.morestack_noctxt(SB)
	0x006c 00108 (test_panic_recover.go:3)	JMP	0
```
可以看到，主要调用了以下几个方法：

```Go
runtime.deferproc()
runtime.gopanic()
runtime.deferreturn()
runtime.gorecover()
```

从gopanic的代码中可以看到，程序panic触发时会调用当前goroutine注册的defer方法链，并使用refectcall调用defer中的传进的方法，如果其中有recover，则调用recover方法；gopanic的代码也可以看出，panic时只有defer的方法才会被调用，否则程序就会退出；所以，只有在defer中的recover才可以捕获panic；

首先看一下deferproc的方法，看完我们就知道goroutine的_defer链是怎么来的了

```Go
func deferproc(siz int32, fn *funcval) {
    sp := getcallersp()
    argp := uintptr(unsafe.Pointer(&fn)) + unsafe.Sizeof(fn)
    callerpc := getcallerpc()
    d := newdefer(siz) //舒适化一个_defer对象，初始化时会自动将qi添加到当前goroutine的_defer链表头部
    f.fn = fn
    d.pc = callerpc
    d.sp = sp
    
    return0()
}
```

可以看出，golang中的每个`defer`关键字将会产生一个_defer对象，设置它的一些字段后放到当前goroutine的头部，gopanic触发时就会调用这些defer中的fn方法；

再来看一下将当前goroutine从panic中恢复的gorecover方法：

```Go
func gorecover(argp uintptr) interface{} {
    gp := getg()
    p := gp._panic
    if p != nil && !p.recoverd && argp == uintptr(p.argp){
        p.recoverd = true
        return p.arg
    }
    return nil
}
```

可以看到这里只是将当前goroutine中最新一次panic的recovered字段设置为了true，并没有任何跳转流程；那么，goroutine是如何从panic中恢复的呢？

再次看一下gopanic的方法可以看到，panic时执行defer中的fn方法，执行完毕后检查当前panic的recoverd字段，如果true，表示当前panic被defer中的recover方法捕获了，恢复goroutine执行的代码就在`mcall(recovery)`中，来看看它的逻辑：

```Go
func recovery(gp *g) {
    sp := gp.sigcode0
    pc := gp.sigcode1

    gp.sched.sp = sp
    gp.sched.pc = pc
    gp.sched.lr = 0
    gp.sched.ret = 1
    gogo(&gp.sched) //将goroutine重定向到sp和pc指向的位置
}
```

这里的代码设置了编译器中的伪寄存器的值，将程序重定向到sp、pc位置，其中sp表示调用栈的栈帧，pc表示要执行的指令；也就是说，recover后程序恢复到了sp和pc的位置继续执行，这两个参数怎么来的呢？程序最终从哪里继续执行呢？

看这两行：

```Go
func recovery(gp *g) {
    sp := gp.sigcode0
    pc := gp.sigcode1
    ...
}
```

gp.sigcode0和sigcode1从哪里来呢？再看gopanic方法：

```Go
func gopanic() {
    ...
    sp := unsafe.Pointer(d.sp)
    pc := d.pc
    ...
    gp.sigcode0 = uintptr(sp)
    gp.sigcode1 = pc
    ...
}

```

程序recover后，要重新开始执行的位置由_defer结构获取，向goroutine添加_defer时的代码中，有如下两行：

```Go
func deferproc() {
    ...
     sp := getcallersp()
     callerpc := getcallerpc()
     ...
     d.sp = sp
     d.pc = callerpc
     ...
}
```

由此，我们已经明白了，defer中的recover恢复panic后，继续执行的代码位置最终来自getcallersp、getcallerpc两个方法。这两个方法的返回值指向了recover后的代码，所以，recover执行完毕后，goroutine又跳转到了deferproc方法中；还有一个无法忽视的细节需要注意：recovery方法通过`gp.sched.ret = 1`将deferproc方法的返回值直接置为1，再看deferfunc最后一句的注释：

```Go
func deferproc() {
    // deferproc returns 0 normally.
    // a deferred func that stops a panic
    // makes the deferproc return 1.
    // the code the compiler generates always
    // checks the return value and jumps to the
    // end of the function if deferproc returns != 0.
    return0()
}
```
正常情况下，deferproc方法将返回0，但是如果defer方法通过recover从panic状态恢复，则返回1，编译器检查其返回值，如果返回值!=0，则不执行deferproc，而是直接跳到该方法后的下一条指令;

看完上面的代码，基本了解了golang中panic和recover的原理；还有一个问题我们没有直接回答：

> 3. 为什么defer只可以捕获本goroutine的panic?

那是因为，程序panic时，执行的defer中的方法，都是当前giroutine的，所以recover只能捕获当前goroutine的panic；

### 参考

- [深入理解 Go panic and recover](https://github.com/EDDYCJY/blog/blob/master/golang/pkg/2019-05-18-%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Go-panic-and-recover.md)