# Go 语言运行时环境变量

[转载](https://dave.cheney.net/2015/11/29/a-whirlwind-tour-of-gos-runtime-environment-variables)

The Go runtime, in addition to providing the usual services of garbage collection, goroutine scheduling, timers, network polling and so forth, contains facilities to enable extra debugging output and even alter the behaviour of the runtime itself.

These facilities are controlled by environment variables passed to the Go program. This post describes the function of the major environment variables supported by the runtime.

Go Runtime除了提供:GC, goroutine调度， 定时器，network polling等服务外， 还提供其它一些工具设施，用于开启额外的调试输出， 

或是改变Go Runtime自身的一些行为。这些工具设施由传给Go program的一些环境变量控制， 本文主要讲述它们。

## GOGC

GOGC is one of the oldest environment variable supported by the Go runtime. It’s possibly older than GOROOT, but nowhere near as well known.

GOGC controls the aggressiveness of the garbage collector. By default this value is assumed to be 100, which means garbage collection will not be triggered until the heap has grown by 100% since the previous collection. Effectively GOGC=100 (the default) means the garbage collector will run each time the live heap doubles.

Setting this value higher, say GOGC=200, will delay the start of a garbage collection cycle until the live heap has grown to 200% of the previous size. Setting the value lower, say GOGC=20 will cause the garbage collector to be triggered more often as less new data can be allocated on the heap before triggering a collection.

Setting GOGC=off will disable garbage collection entirely.

With the introduction of the low latency collector in Go 1.5, phrases like “trigger a garbage collection cycle” become more fluid, but the underlying message that values of GOGC greater than 100 mean the garbage collector will run less often, and for values of GOGC less than 100, more often, remains the same.

GOGC 用于控制GC的处发频率， 其值默认为100, 意为直到自上次垃圾回收后heap size已经增长了100%时GC才触发运行。即是GOGC=100意味着live heap size 每增长一倍，GC触发运行一次。

## GOTRACEBACK

GOTRACEBACK controls the level of detail when a panic hits the top of your program. In Go 1.5 GOTRACEBACK has four valid values.

GOTRACEBACK=0 will suppress all tracebacks, you only get the panic message.
GOTRACEBACK=1 is the default behaviour, stack traces for all goroutines are shown, but stack frames related to the runtime are suppressed.
GOTRACEBACK=2 is the same as the previous value, but frames related to the runtime are also shown, this will reveal goroutines started by the runtime itself.
GOTRACEBACK=crash is the same as the previous value, but rather than calling os.Exit, the runtime will cause the process to segfault, triggering a core dump if permitted by the operating system.
The effect of GOTRACEBACK can be seen with a simple program.

	package main

	func main() {
			panic("kerboom")
	}

Compiling and running this program with GOTRACEBACK=0 shows the suppression of all goroutine stack traces.

	% env GOTRACEBACK=0 ./crash 
	panic: kerboom
	% echo $?
	2
	
Experimentation with the other possible values of GOTRACEBACK is left as an exercise to the reader.

Changes to GOTRACEBACK coming in Go 1.6

For Go 1.6 the interpretation of GOTRACEBACK is changing. The new values of GOTRACEBACK will be:

GOTRACEBACK=none will suppress all tracebacks, you only get the panic message.
GOTRACEBACK=single is the new default behaviour that prints only the goroutine believed to have caused the panic.
GOTRACEBACK=all causes stack traces for all goroutines to be shown, but stack frames related to the runtime are suppressed.
GOTRACEBACK=system is the same as the previous value, but frames related to the runtime are also shown, this will reveal goroutines started by the runtime itself.
GOTRACEBACK=crash is unchanged from Go 1.5.
For compatibility with Go 1.5, a value of 0 maps to none, 1 maps to all, and 2 maps to system.

The major take away from this change is, by default in Go 1.6, panic messages will only print the stack trace for the faulting goroutine. 

GOTRACEBACK 在go 1.6中的变化


GOTRACEBACK=none 只输出panic异常信息。
GOTRACEBACK=single 只输出被认为引发panic异常的那个goroutine的相关信息。
GOTRACEBACK=all 输出所有goroutines的相关信息，除去与go runtime相关的stack frames.
GOTRACEBACK=system 输出所有goroutines的相关信息，包括与go runtime相关的stack frames,从而得知哪些goroutine是go runtime启动运行的。
GOTRACEBACK=crash 与go 1.5相同， 未变化。

注意： 在go 1.6中，默认，只输出引发panci异常的goroutine的stack trace.

## GOMAXPROCS

GOMAXPROCS is the well known (and cargo culted via its runtime.GOMAXPROCS counterpart), value that controls the number of operating system threads allocated to goroutines in your program.

As of Go 1.5, the default value of GOMAXPROCS is the number of CPUs (whatever your operating system considers to be a CPU) visible to the program at startup.

note: the number of operating system threads in use by a Go program includes threads servicing cgo calls, thread blocked on operating system calls, and may be larger than the value of GOMAXPROCS.

GOMAXPROCS 大家比较熟悉， 用于控制操作系统的线程数量， 这些线程用于运行go程序中的goroutines.
到go 1.5的时候， GOMAXPROCS的默认值就是我们的go程序启动时可见的操作系统认为的CPU个数。


注意： 在我们的go程序中使用的操作系统线程数量，也包括：正服务于cgo calls的线程, 阻塞于操作系统calls的线程，
所以go 程序中使用的操作系统线程数量可能大于GOMAXPROCS的值。

## GODEBUG

Saving the best for last is GODEBUG. The contents of GODEBUG are interpreted as a list of name=value pairs separated by commas, where each name is a runtime debugging facility. Here is an example invoking godoc with garbage collection and schedule tracing enabled:
本文其余篇幅主要讲讲GODEBUG. GODEBUG的值被解释为一个个的
name=value对， 每一对间由逗号分割，每一对用于控制go runtime 调试工具设施， 例如

	% env GODEBUG=gctrace=1,schedtrace=1000 godoc -http=:8080

The remainder of this post will discuss the GODEBUG debugging facilities that I find useful to diagnosing Go programs.
上面这条命令用于运行godoc程序时开启 GC tracing and schedule tracing.

### gctrace

Of all the GODEBUG facilities, gctrace is the one I find most useful. Here is the output of the first few milliseconds of a godoc -http server with gctrace debugging enabled:

	% env GODEBUG=gctrace=1 godoc -http=:8080 -index
	gc #1 @0.042s 4%: 0.051+1.1+0.026+16+0.43 ms clock, 0.10+1.1+0+2.0/6.7/0+0.86 ms cpu, 4->32->10 MB, 4 MB goal, 4 P
	gc #2 @0.062s 5%: 0.044+1.0+0.017+2.3+0.23 ms clock, 0.044+1.0+0+0.46/2.0/0+0.23 ms cpu, 4->12->3 MB, 8 MB goal, 4 P
	gc #3 @0.067s 6%: 0.041+1.1+0.078+4.0+0.31 ms clock, 0.082+1.1+0+0/2.8/0+0.62 ms cpu, 4->6->4 MB, 8 MB goal, 4 P
	gc #4 @0.073s 7%: 0.044+1.3+0.018+3.1+0.27 ms clock, 0.089+1.3+0+0/2.9/0+0.54 ms cpu, 4->7->4 MB, 6 MB goal, 4 P

The format of this output changes with every version of Go, but you will always find commonalities like the amount of time of the various gc phases; 0.051+1.1+0.026+16+0.43 ms clock, and the various heap sizes during garbage collection cycle; 4->6->4 MB. This trace also includes the timestamp the gc cycle completed, relative to the start time of the program, however older versions of Go omit this information.

The individual output lines may be useful for analysis, but I find it more useful to view them in aggregate. For example, if you enable gc tracing and the output is continuous, it’s a clear sign that the program is allocation bound. Likewise if the reported size of the heap continues to grow over time, that is a clear sign of a memory leak where references that are expected to be freed are being retained in some global structure.

The overhead of enabling gctrace is effectively zero for production deployments as these statistics are always being collected, but are normally suppressed. I recommend that you enable it at least for some representative sample of your application’s production deployment.

note:setting gctrace to values larger than 1 causes each garbage collection cycle to be run twice. This exercises some aspects of finalisation that require two garbage collection cycles to complete. You should not use this as a mechanism to alter finalisation performance in your programs because you should not write programs whose correctness depends on finalisation.

此信息的输出格式随着go的每一不同的版本发生变化，但总是能发现共性的东西， 如： 每一GC 阶段所花费的时间量， heap size 的变化量， 
也包括每一GC阶段完成时间，相对于程序启动时的时间，当然老版本go可能省略一些信息。


每一行信息都很有用， 不过我认为综合分析这些信息则更有用，比如， 不断输出的gc tracing,可以清楚在表明程序的内存分配情况， 

持续不断增长的heap size 则表明可能有内存泄露，也许一些被引用的东西没有被释放。



开启gctrace的代价是很小的，不过其通常是关闭的， 不过我推荐在一些产品环境中，抽取一些
样本产品，开启这个调试工具。

### The heap scavenger

By far the most useful piece of output enabled by gctrace=1 is the output of the heap scavenger.

	scvg143: inuse: 8, idle: 104, sys: 113, released: 104, consumed: 8 (MB)
	
The scavenger’s job is to periodically sweep the heap looking for unused operating system pages. The scavenger then releases them by notifying the operating system that these memory pages from the heap that are not in use. There is no facility to force the operating system to take back the page and many operating systems choose to ignore this advice, or at least defer taking any action until the a time when the machine is starved for free memory.

The output from the scavenger is the best way I know of to tell how much virtual address space is in use by your Go program. It is expected that these values will vary significantly from what tools like free(1) and top(1) report. You should trust the values reported by the scavenger.

scavenger 的工作就是周期性地打扫heap中无用的操作系统内存分页， 它会向操作系统发出建义，请操作系统回收无用内存页，

当然并不能强迫操作系统立刻就去做回收处理，操作系统可以忽略此建义，或是延迟回收，比如直到可分配的空闲内存不够的时候。

scavenger输出的信息是我们了解go程序虚拟内存空间使用情况的最好方式， 当然你也可以通过其它工具，如free, top来获到这些信息，
不过你应用信任scavenger.

### schedtrace

Because the Go runtime manages the allocation of a large set of goroutines onto a smaller set of operating system threads, observing your program externally may not give sufficient detail to understand its performance. You may need to investigate the operation of the runtime scheduler directly.  This output is controlled with the schedtrace value:

因为go runtime管理着大量的goroutine, 并调度goroutine在操作系统线程集上运行，
这个操作系统线程集，其实是就是线程池， 所以从外部考察go程序的性能我们不能获取足够的细节信息，
更谈不上准确分析程序性能。故此我们需要直接了解go runtime scheduler的每一个操作，其输出如下：

	% env GODEBUG=schedtrace=1000 godoc -http=:8080 -index
	SCHED 0ms: gomaxprocs=4 idleprocs=2 threads=4 spinningthreads=1 idlethreads=0 runqueue=0 [0 0 0 0]
	SCHED 1001ms: gomaxprocs=4 idleprocs=0 threads=8 spinningthreads=0 idlethreads=2 runqueue=0 [189 197 231 142]
	SCHED 2004ms: gomaxprocs=4 idleprocs=0 threads=9 spinningthreads=0 idlethreads=1 runqueue=0 [54 45 38 86]
	SCHED 3011ms: gomaxprocs=4 idleprocs=0 threads=9 spinningthreads=0 idlethreads=2 runqueue=2 [85 0 67 111]
	SCHED 4018ms: gomaxprocs=4 idleprocs=3 threads=9 spinningthreads=0 idlethreads=4 runqueue=0 [0 0 0 0]
	
Append scheddetail=1 will cause the runtime to output the state of each individual goroutine in addition to the summary, producing very verbose output.

设定scheddetail=1将使go runtime输出总结性信息时， 一并输出每一个goroutine的状态信息，如：

	% env GODEBUG=scheddetail=1,schedtrace=1000 godoc -http=:8080 -index
	SCHED 0ms: gomaxprocs=4 idleprocs=3 threads=3 spinningthreads=0 idlethreads=0 runqueue=0 gcwaiting=0 nmidlelocked=0 stopwait=0 sysmonwait=0
	  P0: status=1 schedtick=0 syscalltick=0 m=0 runqsize=0 gfreecnt=0
	  P1: status=0 schedtick=0 syscalltick=0 m=-1 runqsize=0 gfreecnt=0
	  P2: status=0 schedtick=0 syscalltick=0 m=-1 runqsize=0 gfreecnt=0
	  P3: status=0 schedtick=0 syscalltick=0 m=-1 runqsize=0 gfreecnt=0
	  M2: p=-1 curg=-1 mallocing=0 throwing=0 preemptoff= locks=1 dying=0 helpgc=0 spinning=false blocked=false lockedg=-1
	  M1: p=-1 curg=17 mallocing=0 throwing=0 preemptoff= locks=0 dying=0 helpgc=0 spinning=false blocked=false lockedg=17
	  M0: p=0 curg=1 mallocing=0 throwing=0 preemptoff= locks=2 dying=0 helpgc=0 spinning=false blocked=false lockedg=1
	  G1: status=2(stack growth) m=0 lockedm=0
	  G17: status=3() m=1 lockedm=1
	  G2: status=1() m=-1 lockedm=-1
	  
This output may be useful for debugging leaking goroutines, but other facilities like net/http/pprof are likely to be more useful.

这个输出对于调试goroutines leaking很有帮助， 不过其它工具， 诸如：net/http/pprof 
 好像更有用一些。