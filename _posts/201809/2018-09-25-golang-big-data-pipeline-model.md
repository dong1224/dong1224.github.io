# Golang处理大数据时使用高效的Pipeline（流水线）执行模型(转)

Golang被证明非常适合并发编程，goroutine比异步编程更易读、优雅、高效。本文提出一个适合由Golang实现的Pipeline执行模型，适合批量处理大量数据（ETL）的情景。

想象这样的应用情景：
（1）从数据库A（Cassandra）加载用户评论（量巨大，例如10亿条）；
（2）根据每条评论的用户ID、从数据库B（MySQL）关联用户资料；
（3）调用NLP服务（自然语言处理），处理每条评论；
（4）将处理结果写入数据库C（ElasticSearch）。

由于应用中遇到的各种问题，归纳出这些需求：
需求一：应分批处理数据，例如规定每批100条。出现问题时（例如任意一个数据库故障）则中断，下次程序启动时使用checkpoint从中断处恢复。
需求二：每个流程设置合理的并发数、让数据库和NLP服务有合理的负载（不影响其它业务的基础上，尽可能占用更多资源以提高ETL性能）。例如，步骤（1）-（4）分别设置并发数1、8、32、2。

这就是一个典型的Pipeline（流水线）执行模型。把每一批数据（例如100条）看作流水线上的产品，4个步骤对应流水线上4个处理工序，每个工序处理完毕后就把半成品交给下一个工序。每个工序可以同时处理的产品数各不相同。

你可能首先想到启用1+8+32+2个goroutine，使用channel来传递半成品。我也曾经这么干，结论就是这么干会让程序员疯掉：流程并发控制代码非常复杂，特别是你得处理异常、执行时间超出预期、可控中断等问题，你不得不加入一堆channel，直到你自己都不记得有什么用。

为了更高效完成ETL工作，我将Pipeline抽象成模块。我先把代码粘贴出来，再解析含义。模块可以直接使用，主要使用的接口是：NewPipeline、Async、Wait。

<pre><code>package main

import "sync"

func HasClosed(c <-chan struct{}) bool {
    select {
    case <-c: return true
    default: return false
    }
}

type SyncFlag interface{
    Wait()
    Chan() <-chan struct{}
    Done() bool
}

func NewSyncFlag() (done func(), flag SyncFlag) {
    f := &syncFlag{
        c : make(chan struct{}),
    }
    return f.done, f
}

type syncFlag struct {
    once sync.Once
    c chan struct{}
}

func (f *syncFlag) done() {
    f.once.Do(func(){
        close(f.c)
    })
}

func (f *syncFlag) Wait() {
    <-f.c
}

func (f *syncFlag) Chan() <-chan struct{} {
    return f.c
}

func (f *syncFlag) Done() bool {
    return HasClosed(f.c)
}

type pipelineThread struct {
    sigs []chan struct{}
    chanExit chan struct{}
    interrupt SyncFlag
    setInterrupt func()
    err error
}

func newPipelineThread(l int) *pipelineThread {
    p := &pipelineThread{
        sigs : make([]chan struct{}, l),
        chanExit : make(chan struct{}),
    }
    p.setInterrupt, p.interrupt = NewSyncFlag()

    for i := range p.sigs {
        p.sigs[i] = make(chan struct{})
    }
    return p
}

type Pipeline struct {
    mtx sync.Mutex
    workerChans []chan struct{}
    prevThd *pipelineThread
}

//创建流水线，参数个数是每个任务的子过程数，每个参数对应子过程的并发度。
func NewPipeline(workers ...int) *Pipeline {
    if len(workers) < 1 { panic("NewPipeline need aleast one argument") }

    workersChan := make([]chan struct{}, len(workers))
    for i := range workersChan {
        workersChan[i] = make(chan struct{}, workers[i])
    }

    prevThd := newPipelineThread(len(workers))
    for _,sig := range prevThd.sigs {
        close(sig)
    }
    close(prevThd.chanExit)

    return &Pipeline{
        workerChans : workersChan,
        prevThd : prevThd,
    }
}

//往流水线推入一个任务。如果第一个步骤的并发数达到设定上限，这个函数会堵塞等待。
//如果流水线中有其它任务失败（返回非nil），任务不被执行，函数返回false。
func (p *Pipeline) Async(works ...func()error) bool {
    if len(works) != len(p.workerChans) {
        panic("Async: arguments number not matched to NewPipeline(...)")
    }

    p.mtx.Lock()
    if p.prevThd.interrupt.Done() {
        p.mtx.Unlock()
        return false
    }
    prevThd := p.prevThd
    thisThd := newPipelineThread(len(p.workerChans))
    p.prevThd = thisThd
    p.mtx.Unlock()

    lock := func(idx int) bool {
        select {
        case <-prevThd.interrupt.Chan(): return false
        case <-prevThd.sigs[idx]: //wait for signal
        }
        select {
        case <-prevThd.interrupt.Chan(): return false
        case p.workerChans[idx]<-struct{}{}: //get lock
        }
        return true
    }
    if !lock(0) {
        thisThd.setInterrupt()
        <-prevThd.chanExit
        thisThd.err = prevThd.err
        close(thisThd.chanExit)
        return false
    }
    go func() { //watch interrupt of previous thread
        select {
        case <-prevThd.interrupt.Chan():
            thisThd.setInterrupt()
        case <-thisThd.chanExit:
        }
    }()
    go func() {
        var err error
        for i,work := range works {
            close(thisThd.sigs[i]) //signal next thread
            if work != nil {
                err = work()
            }
            if err != nil || (i+1 < len(works) && !lock(i+1)) {
                thisThd.setInterrupt()
                break
            }
            <-p.workerChans[i] //release lock
        }

        <-prevThd.chanExit
        if prevThd.interrupt.Done() {
            thisThd.setInterrupt()
        }
        if prevThd.err != nil {
            thisThd.err = prevThd.err
        } else {
            thisThd.err = err
        }
        close(thisThd.chanExit)
    }()
    return true
}

//等待流水线中所有任务执行完毕或失败，返回第一个错误，如果无错误则返回nil。
func (p *Pipeline) Wait() error {
    p.mtx.Lock()
    lastThd := p.prevThd
    p.mtx.Unlock()
    <-lastThd.chanExit
    return lastThd.err
}</code></pre>

使用这个Pipeline组件，我们的ETL程序将会简单、高效、可靠，让程序员从繁琐的并发流程控制中解放出来：
<pre><code>package main

import "log"

func main() {
    checkpoint := loadCheckpoint()
    
    //工序(1)在pipeline外执行，最后一个工序是保存checkpoint
    pipeline := NewPipeline(8, 32, 2, 1) 
    for {
        //(1)
        //加载100条数据，并修改变量checkpoint
        //data是数组，每个元素是一条评论，之后的联表、NLP都直接修改data里的每条记录。
        data, err := extractReviewsFromA(&checkpoint, 100) 
        if err != nil {
            log.Print(err)
            break
        }
        curCheckpoint := checkpoint
        
        ok := pipeline.Async(func() error {
            //(2)
            return joinUserFromB(data)
        }, func() error {
            //(3)
            return nlp(data)
        }, func() error {
            //(4)
            return loadDataToC(data)
        }, func() error {
            //(5)保存checkpoint
            log.Print("done:", curCheckpoint)
            return saveCheckpoint(curCheckpoint)
        })
        if !ok { break }
        
        if len(data) < 100 { break } //处理完毕
    }
    err := pipeline.Wait()
    if err != nil { log.Print(err) }
}</code></pre>

Pipeline执行模型的特性：

1、Pipeline分别控制每一个工序的并发数，如果(4)的并发数已满，某个线程的(3)即使完成都会堵塞等待，直到(4)有一个线程完成。
2、在上面的情景中，Pipeline最多同时处理1+8+32+2+1=44个线程共4400条记录，内存开销可控。
3、每个线程的每个工序的调度，不早于上一个线程同一个工序的调度。

例如：有两个线程正在执行，<1>先执行、<2>后执行。如果<2>(4)早于<1>(4)完成，那<2>必须堵塞等待，直到<1>(4)完成、<1>(5)开始执行，那<2>(5)才会开始。又因为(5)的最大并发数是1，所以实际上<2>(5)必须等待<1>(5)完成才会开始。这个机制保证checkpoint的执行顺序一定是按照Async的顺序，避免中断、继续时漏处理数据。
4、如果某个线程的某个工序处理失败（例如数据库故障），那之后的线程都会中止执行，下一次调用Async返回false，pipeline.Wait()返回第一个错误，整个流水线作业可控中断。

例如：有三个线程正在执行：<1>、<2>、<3>。如果<2>(4)失败（loadDataToC返回error非nil），那<3>无论正在执行到哪一个工序，都不会进入下一个工序而中断。<1>不会受到影响，会一直执行完毕。Wait()等待<1><2><3>全部完成或中止，返回loadDataToC的错误。

5、无法避免中断过程中有checkpoint后的数据写入。下次重启程序将重新写入、覆盖这些数据。

例如：<2>(4)失败、<3>(4)执行成功（已写入数据），那<2>(5)和<3>(5)都不会被执行，checkpoint的最新状态是<1>写入的，下次重启程序将重新执行<2>和<3>，其中<3>的数据会再次写入，所以写入应该按照记录ID作覆盖写入。

6、你可以随时Ctrl+C、重启程序，所有事情都会继续有序执行。死机？毫无压力。

总结：Pipeline执行模型除了限制并发数，也能限制内存开销，对失败恢复有充足的考虑，让程序员从繁琐的并发编程中解放出来。

吐槽：Python程序员没办法用几百行代码就漂亮地完成这个任务。