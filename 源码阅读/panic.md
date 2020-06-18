# 5.8学习日报

## panic原理

panic 能改变程序的控制流，函数调用panic会立即停止执行当前函数的其他代码，并在结束后再当前go程中递归执行调用方的延迟函数调用defer

#### 数据结构

在runtime2.go包中

~~~go
type _panic struct {
	argp      unsafe.Pointer // pointer to arguments of deferred call run during panic; cannot move - known to liblink
	arg       interface{}    // argument to panic
	link      *_panic        // link to earlier panic
	pc        uintptr        // where to return to in runtime if this panic is bypassed
	sp        unsafe.Pointer // where to return to in runtime if this panic is bypassed
	recovered bool           // whether this panic is over
	aborted   bool           // the panic was aborted
	goexit    bool
}
~~~

argp: 指向defer调用时参数的指针  

arg: 调用panic函数传入的参数 ,通常是字符串

link: 指向前一个调用的_panic，这也代表了panic采用了链表结构

pc: 程序计数器，计组中经常提到的，下一个要执行的指令序号。

sp: 栈指针 stack pointer，指向当前栈位置

还有三个布尔类型的变量分别是 当前panic是否被recover,是否被强行终止，go程是否成功退出。

unsafe.Pointer类型作用是指向一个任意类型的指针

很经典的整数类型（经典吓唬人

~~~go
type Pointer *ArbitraryType
type ArbitraryType int
~~~

#### 运行流程

当程序执行运行到panic时，会调用runtime.gopanic函数，由于太长了只截取部分代码

~~~go
// panic.go文件中
func gopanic(e interface{}) {
	gp := getg()
	...
	var p _panic
	p.arg = e
	p.link = gp._panic
	gp._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

	for {
		d := gp._defer
		if d == nil {
			break
		}

		d._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

		reflectcall(nil, unsafe.Pointer(d.fn), deferArgs(d), uint32(d.siz), uint32(d.siz))

		d._panic = nil
		d.fn = nil
		gp._defer = d.link

		freedefer(d)
		if p.recovered {
			...
		}
	}

	fatalpanic(gp._panic)
	*(*int)(nil) = 0
}
~~~

这里实际上还涉及go程的数据结构，比如第一行就是获取了当前go程，然后接下来初始化\_painc后，把他加在goroutine的\_panic链表上。go程的数据结构就不具体看了。

在for循环中，是不断的从go程的defer链表中，获取defer并反射执行。如果在这期间执行了recover函数，则会恢复程序，由p.recovered标志

最后则会调用fatalpanic函数终止go程，并打印全部panic消息和传入参数，返回错误码2

那么程序通过recover恢复，执行的函数如下：

~~~go
func gorecover(argp uintptr) interface{} {
	// Must be in a function running as part of a deferred call during the panic.
	// Must be called from the topmost function of the call
	// (the function used in the defer statement).
	// p.argp is the argument pointer of that topmost deferred function call.
	// Compare against argp reported by caller.
	// If they match, the caller is the one who can recover.
	gp := getg()
	p := gp._panic
	if p != nil && !p.goexit && !p.recovered && argp == uintptr(p.argp) {
		p.recovered = true
		return p.arg
	}
	return nil
}
~~~

可以看到，recover只是改了_panic的是否恢复标志位，并没有做具体的恢复逻辑。

而在gopanic函数中，当执行完含有recover的defer函数后，检测p.recovered为true，则进行程序恢复。

~~~go
gp._panic = p.link
for gp._panic != nil && gp._panic.aborted {
    gp._panic = gp._panic.link
}
if gp._panic == nil {
    gp.sig = 0
}
gp.sigcode0 = uintptr(sp)
gp.sigcode1 = pc
mcall(recovery)
throw("recovery failed")
~~~

因为可能这个panic并不是第一个被调的panic，所以要把go._panic置为下一个panic，然后将gp中的sp,pc设置为defer处的，紧接着调用recovery函数。

~~~go
func recovery(gp *g) {
	// Info about defer passed in G struct.
	sp := gp.sigcode0
	pc := gp.sigcode1

	// d's arguments need to be in the stack.
	if sp != 0 && (sp < gp.stack.lo || gp.stack.hi < sp) {
		print("recover: ", hex(sp), " not in [", hex(gp.stack.lo), ", ", hex(gp.stack.hi), "]\n")
		throw("bad recovery")
	}

	// Make the deferproc for this d return again,
	// this time returning 1. The calling function will
	// jump to the standard return epilogue.
	gp.sched.sp = sp
	gp.sched.pc = pc
	gp.sched.lr = 0
	gp.sched.ret = 1
	gogo(&gp.sched)
}
~~~

>  这里的 [`runtime.gogo`](https://github.com/golang/go/blob/22d28a24c8b0d99f2ad6da5fe680fa3cfa216651/src/runtime/asm_386.s#L301-L314) 函数会跳回 `defer` 关键字调用的位置。[`runtime.recovery`](https://github.com/golang/go/blob/22d28a24c8b0d99f2ad6da5fe680fa3cfa216651/src/runtime/panic.go#L1132-L1151) 在调度过程中会将函数的返回值设置成 1。从 [`runtime.deferproc`](https://github.com/golang/go/blob/22d28a24c8b0d99f2ad6da5fe680fa3cfa216651/src/runtime/panic.go#L218-L258) 的注释中我们会发现，当 [`runtime.deferproc`](https://github.com/golang/go/blob/22d28a24c8b0d99f2ad6da5fe680fa3cfa216651/src/runtime/panic.go#L218-L258) 函数的返回值是 1 时，编译器生成的代码会直接跳转到调用方函数返回之前并执行 [`runtime.deferreturn`](https://github.com/golang/go/blob/22d28a24c8b0d99f2ad6da5fe680fa3cfa216651/src/runtime/panic.go#L526-L571)
>
> 跳转到 [`runtime.deferreturn`](https://github.com/golang/go/blob/22d28a24c8b0d99f2ad6da5fe680fa3cfa216651/src/runtime/panic.go#L526-L571) 函数之后，程序就已经从 `panic` 中恢复了并执行正常的逻辑，而 [`runtime.gorecover`](https://github.com/golang/go/blob/22d28a24c8b0d99f2ad6da5fe680fa3cfa216651/src/runtime/panic.go#L1080-L1094) 函数也能从 [`runtime._panic`](https://github.com/golang/go/blob/cfe3cd903f018dec3cb5997d53b1744df4e53909/src/runtime/runtime2.go#L891-L900) 结构体中取出了调用 `panic` 时传入的 `arg` 参数并返回给调用方。



最后总结内容（不是我总结的）

> 1. 编译器会负责做转换关键字的工作；
>    1. 将 `panic` 和 `recover` 分别转换成 `runtime.gopanic` 和 `runtime.gorecover`；
>    2. 将 `defer` 转换成 `deferproc` 函数；
>    3. 在调用 `defer` 的函数末尾调用 `deferreturn` 函数；
> 2. 在运行过程中遇到 `gopanic` 方法时，会从 Goroutine 的链表依次取出 `_defer` 结构体并执行；
> 3. 如果调用延迟执行函数时遇到了gorecover就会将_panic.recovered标记成true并返回panic的参数；
>    1. 在这次调用结束之后，`gopanic` 会从 `_defer` 结构体中取出程序计数器 `pc` 和栈指针 `sp` 并调用 `recovery` 函数进行恢复程序；
>    2. `recovery` 会根据传入的 `pc` 和 `sp` 跳转回 `deferproc`；
>    3. 编译器自动生成的代码会发现 `deferproc` 的返回值不为 0，这时会跳回 `deferreturn` 并恢复到正常的执行流程；
> 4. 如果没有遇到 `gorecover` 就会依次遍历所有的 `_defer` 结构，并在最后调用 `fatalpanic` 中止程序、打印 `panic` 的参数并返回错误码 `2`；

## goland快捷键

为了脱离鼠标而做的总结，鼠标点点点真的太累惹

### 查看

CTRL+ALT ←/→ 返回上次编辑的位置

### 代码编辑

CTRL+Z 倒退

CTRL+SHIFT+Z 向前

CTRL+D 复制行
CTRL+X 剪切,删除行



## chrome快捷键

| 打开新的标签页，并跳转到该标签页           | Ctrl + t                          |
| ------------------------------------------ | --------------------------------- |
| 重新打开最后关闭的标签页，并跳转到该标签页 | Ctrl + Shift + t                  |
| 跳转到下一个打开的标签页                   | Ctrl + Tab 或 Ctrl + PgDn         |
| 跳转到上一个打开的标签页                   | Ctrl + Shift + Tab 或 Ctrl + PgUp |
| 跳转到特定标签页                           | Ctrl + 1 到 Ctrl + 8              |
| 跳转到最后一个标签页                       | Ctrl + 9                          |

| 关闭当前标签页               | Ctrl + w 或 Ctrl + F4        |
| ---------------------------- | ---------------------------- |
| 关闭所有打开的标签页和浏览器 | Ctrl + Shift + w             |
| 最小化当前窗口               | Alt + 空格键 + n             |
| 最大化当前窗口               | Alt + 空格键 + x             |
| 退出 Google Chrome           | Ctrl + Shift + q 或 Alt + F4 |

### windows快捷键

alt + tab 切换打开的应用

alt + 空格 锁屏