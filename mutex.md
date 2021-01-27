# sync.Mutex深入解析
> 本文代码基于go 1.14
## 引言
golang中sync.Mutex包是非常常用的一个包，经常用来做互斥锁，保护临界区代码访问。本文就来解析下sync.Mutex锁的实现，本文绝对不是一个流水账似的代码注释型的文章，本文力争让读者知其然而且知其所以然，阅读全文大概15分钟左右。
## 是什么
> 该部分参考了 https://blog.csdn.net/u010853261/article/details/106293258

源码位置为: src/sync/mutex.go, 代码非常短小精悍，总共二百多行，除去注释也就五十多行，我们先把代码加上注释放到下面:
```
type Mutex struct {
	state int32
	sema  uint32
}

//下文代码中用到的几个常量
const (
	mutexLocked = 1 << iota // 1
	mutexWoken    // 2
	mutexStarving  // 4
	mutexWaiterShift = iota //3
	starvationThresholdNs = 1e6
)
```

首先Mutex就只有两个变量，一个表示状态，一个是信号量，非常简洁。
mutex的状态机比较复杂，使用一个int32来表示：
```
32                                               3             2             1             0 
 |                                               |             |             |             | 
 |                                               |             |             |             | 
 v-----------------------------------------------v-------------v-------------v-------------+ 
 |                                               |             |             |             v 
 |                 waitersCount                  |mutexStarving| mutexWoken  | mutexLocked | 
 |                                               |             |             |             | 
 +-----------------------------------------------+-------------+-------------+-------------+                                                                                                              

```
最低三位分别表示 mutexLocked、mutexWoken 和 mutexStarving，剩下的位置用来表示当前有多少个 Goroutine 等待互斥锁的释放：

在默认情况下，互斥锁的所有状态位都是 0，int32 中的不同位分别表示了不同的状态：

mutexLocked — 表示互斥锁的锁定状态；
mutexWoken — 表示从正常模式被从唤醒；
mutexStarving — 当前的互斥锁进入饥饿状态；
waitersCount — 当前互斥锁上等待的 goroutine 个数；
为了保证锁的公平性，设计上互斥锁有两种状态：正常状态和饥饿状态。

正常模式下，所有等待锁的goroutine按照FIFO顺序等待。唤醒的goroutine不会直接拥有锁，而是会和新请求锁的goroutine竞争锁的拥有。新请求锁的goroutine具有优势：它正在CPU上执行，而且可能有好几个，所以刚刚唤醒的goroutine有很大可能在锁竞争中失败。在这种情况下，这个被唤醒的goroutine会加入到等待队列的前面。 如果一个等待的goroutine超过1ms没有获取锁，那么它将会把锁转变为饥饿模式。

饥饿模式下，锁的所有权将从unlock的gorutine直接交给交给等待队列中的第一个。新来的goroutine将不会尝试去获得锁，即使锁看起来是unlock状态, 也不会去尝试自旋操作，而是放在等待队列的尾部。

如果一个等待的goroutine获取了锁，并且满足一以下其中的任何一个条件：(1)它是队列中的最后一个；(2)它等待的时候小于1ms。它会将锁的状态转换为正常状态。

正常状态有很好的性能表现，饥饿模式也是非常重要的，因为它能阻止尾部延迟的现象。

我们再看下加锁相关的代码如下, 其中增加了详细的注释:
```
func (m *Mutex) Lock() {
    // 如果mutex的state没有被锁，也没有等待/唤醒的goroutine, 锁处于正常状态，那么获得锁，返回.
    // 比如锁第一次被goroutine请求时，就是这种状态。或者锁处于空闲的时候，也是这种状态。
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        return
    }
    // Slow path (outlined so that the fast path can be inlined)
    m.lockSlow()
}

func (m *Mutex) lockSlow() {
    // 标记本goroutine的等待时间
    var waitStartTime int64
    // 本goroutine是否已经处于饥饿状态
    starving := false
    // 本goroutine是否已唤醒
    awoke := false
    // 自旋次数
    iter := 0
    old := m.state
    for {
        // 第一个条件：1.mutex已经被锁了；2.不处于饥饿模式(如果时饥饿状态，自旋时没有用的，锁的拥有权直接交给了等待队列的第一个。)
        // 尝试自旋的条件：参考runtime_canSpin函数
        if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
            // 进入这里肯定是普通模式
            // 自旋的过程中如果发现state还没有设置woken标识，则设置它的woken标识， 并标记自己为被唤醒。
            if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                awoke = true
            }
            runtime_doSpin()
            iter++
            old = m.state
            continue
        }
        
        // 到了这一步， state的状态可能是：
        // 1. 锁还没有被释放，锁处于正常状态
        // 2. 锁还没有被释放， 锁处于饥饿状态
        // 3. 锁已经被释放， 锁处于正常状态
        // 4. 锁已经被释放， 锁处于饥饿状态
        // 并且本gorutine的 awoke可能是true, 也可能是false (其它goutine已经设置了state的woken标识)
        
        // new 复制 state的当前状态， 用来设置新的状态
        // old 是锁当前的状态
        new := old
        
        // 如果old state状态不是饥饿状态, new state 设置锁， 尝试通过CAS获取锁,
        // 如果old state状态是饥饿状态, 则不设置new state的锁，因为饥饿状态下锁直接转给等待队列的第一个.
        if old&mutexStarving == 0 {
            new |= mutexLocked
        }
        // 将等待队列的等待者的数量加1
        if old&(mutexLocked|mutexStarving) != 0 {
            new += 1 << mutexWaiterShift
        }
        
        // 如果当前goroutine已经处于饥饿状态， 并且old state的已被加锁,
        // 将new state的状态标记为饥饿状态, 将锁转变为饥饿状态.
        if starving && old&mutexLocked != 0 {
            new |= mutexStarving
        }
        
         // 如果本goroutine已经设置为唤醒状态, 需要清除new state的唤醒标记, 因为本goroutine要么获得了锁，要么进入休眠，
        // 总之state的新状态不再是woken状态.
        if awoke {
            // The goroutine has been woken from sleep,
            // so we need to reset the flag in either case.
            if new&mutexWoken == 0 {
                throw("sync: inconsistent mutex state")
            }
            new &^= mutexWoken
        }

        // 通过CAS设置new state值.
        // 注意new的锁标记不一定是true, 也可能只是标记一下锁的state是饥饿状态.
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            
            // 如果old state的状态是未被锁状态，并且锁不处于饥饿状态,
            // 那么当前goroutine已经获取了锁的拥有权，返回
            if old&(mutexLocked|mutexStarving) == 0 {
                break // locked the mutex with CAS
            }
            // If we were already waiting before, queue at the front of the queue.
            // 设置并计算本goroutine的等待时间
            queueLifo := waitStartTime != 0
            if waitStartTime == 0 {
                waitStartTime = runtime_nanotime()
            }
            // 既然未能获取到锁， 那么就使用sleep原语阻塞本goroutine
            // 如果是新来的goroutine,queueLifo=false, 加入到等待队列的尾部，耐心等待
            // 如果是唤醒的goroutine, queueLifo=true, 加入到等待队列的头部
            runtime_SemacquireMutex(&m.sema, queueLifo, 1)

            // sleep之后，此goroutine被唤醒
            // 计算当前goroutine是否已经处于饥饿状态.
            starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
            // 得到当前的锁状态
            old = m.state

            // 如果当前的state已经是饥饿状态
            // 那么锁应该处于Unlock状态，那么应该是锁被直接交给了本goroutine
            if old&mutexStarving != 0 {
                // If this goroutine was woken and mutex is in starvation mode,
                // ownership was handed off to us but mutex is in somewhat
                // inconsistent state: mutexLocked is not set and we are still
                // accounted as waiter. Fix that.
                if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
                    throw("sync: inconsistent mutex state")
                }
                // 当前goroutine用来设置锁，并将等待的goroutine数减1.
                delta := int32(mutexLocked - 1<<mutexWaiterShift)
                // 如果本goroutine是最后一个等待者，或者它并不处于饥饿状态，
                // 那么我们需要把锁的state状态设置为正常模式.
                if !starving || old>>mutexWaiterShift == 1 {
                     // 退出饥饿模式
                    delta -= mutexStarving
                }
                // 设置新state, 因为已经获得了锁，退出、返回
                atomic.AddInt32(&m.state, delta)
                break
            }
            awoke = true
            iter = 0
        } else {
            old = m.state
        }
    }
}
```
整个过程比较复杂，这里总结一下一些重点：

如果锁处于初始状态(unlock, 正常模式)，则通过CAS(0 -> Locked)获取锁；如果获取失败，那么就进入slowLock的流程：

slowLock的获取锁流程有两种模式： 饥饿模式 和 正常模式。

### (1)正常模式
mutex已经被locked了，处于正常模式下；
前 Goroutine 为了获取该锁进入自旋的次数小于四次；
当前机器CPU核数大于1；
当前机器上至少存在一个正在运行的处理器 P 并且处理的运行队列为空；
满足上面四个条件的goroutine才可以做自旋。自旋就会调用sync.runtime_doSpin 和 runtime.procyield 并执行 30 次的 PAUSE 指令，该指令只会占用 CPU 并消耗 CPU 时间。

处理了自旋相关的特殊逻辑之后，互斥锁会根据上下文计算当前互斥锁最新的状态new。几个不同的条件分别会更新 state 字段中存储的不同信息 — mutexLocked、mutexStarving、mutexWoken 和 mutexWaiterShift：

计算最新的new之后，CAS更新，如果更新成功且old状态是未被锁状态，并且锁不处于饥饿状态，就代表当前goroutine竞争成功并获取到了锁返回。(这也就是当前goroutine在正常模式下竞争时更容易获得锁的原因)

如果当前goroutine竞争失败，会调用 sync.runtime_SemacquireMutex 使用信号量保证资源不会被两个 Goroutine 获取。sync.runtime_SemacquireMutex 会在方法中不断调用尝试获取锁并休眠当前 Goroutine 等待信号量的释放，一旦当前 Goroutine 可以获取信号量，它就会立刻返回，sync.Mutex.Lock 方法的剩余代码也会继续执行。

### (2) 饥饿模式
饥饿模式本身是为了一定程度保证公平性而设计的模式。所以饥饿模式不会有自旋的操作，新的 Goroutine 在该状态下不能获取锁、也不会进入自旋状态，它们只会在队列的末尾等待。

不管是正常模式还是饥饿模式，获取信号量，它就会从阻塞中立刻返回，并执行剩下代码：

在正常模式下，这段代码会设置唤醒和饥饿标记、重置迭代次数并重新执行获取锁的循环；
在饥饿模式下，当前 Goroutine 会获得互斥锁，如果等待队列中只存在当前 Goroutine，互斥锁还会从饥饿模式中退出；

```
func (m *Mutex) Unlock() {
    // state-1标识解锁
    new := atomic.AddInt32(&m.state, -mutexLocked)
    if new != 0 {
        // Outlined slow path to allow inlining the fast path.
        // To hide unlockSlow during tracing we skip one extra frame when tracing GoUnblock.
        m.unlockSlow(new)
    }
}

func (m *Mutex) unlockSlow(new int32) {
    // 验证锁状态是否符合
    if (new+mutexLocked)&mutexLocked == 0 {
        throw("sync: unlock of unlocked mutex")
    }
    // xxxx...x0xx & 0100 = 0 ;判断是否处于正常模式
    if new&mutexStarving == 0 {
        old := new
        for {
            // 如果没有等待的goroutine或goroutine已经解锁完成
            if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
                return
            }
            // Grab the right to wake someone.
            new = (old - 1<<mutexWaiterShift) | mutexWoken
            if atomic.CompareAndSwapInt32(&m.state, old, new) {
                runtime_Semrelease(&m.sema, false, 1)
                return
            }
            old = m.state
        }
    } else {
        // 饥饿模式:将mutex所有权移交给下一个等待的goroutine
        // 注意:mutexlock没有设置，goroutine会在唤醒后设置。
        // 但是互斥锁仍然被认为是锁定的，如果互斥对象被设置，所以新来的goroutines不会得到它
        runtime_Semrelease(&m.sema, true, 1)
    }
}

```
互斥锁的解锁过程 sync.Mutex.Unlock 与加锁过程相比就很简单，该过程会先使用 AddInt32 函数快速解锁，这时会发生下面的两种情况：

如果该函数返回的新状态等于 0，当前 Goroutine 就成功解锁了互斥锁；
如果该函数返回的新状态不等于 0，这段代码会调用 sync.Mutex.unlockSlow 方法开始慢速解锁：
sync.Mutex.unlockSlow 方法首先会校验锁状态的合法性 — 如果当前互斥锁已经被解锁过了就会直接抛出异常 sync: unlock of unlocked mutex 中止当前程序。

在正常情况下会根据当前互斥锁的状态，分别处理正常模式和饥饿模式下的互斥锁：

在正常模式下，这段代码会分别处理以下两种情况处理；
如果互斥锁不存在等待者或者互斥锁的 mutexLocked、mutexStarving、mutexWoken 状态不都为 0，那么当前方法就可以直接返回，不需要唤醒其他等待者；
如果互斥锁存在等待者，会通过 sync.runtime_Semrelease 唤醒等待者并移交锁的所有权；
在饥饿模式下，上述代码会直接调用 sync.runtime_Semrelease 方法将当前锁交给下一个正在尝试获取锁的等待者，等待者被唤醒后会得到锁，在这时互斥锁还不会退出饥饿状态；

## 为什么是这样
上文流水账似的注释了相关的代码，以及摘抄了一些网上的注释，下文就重点讲解下为什么。我先提几个问题，如果你能回答上来那么可以不用读下面的文章，否则相信我，读完你对mutex的理解会上一个台阶。

1. 为什么有mutexWoken, 没有不行吗?
1. atomic.CompareAndSwapInt32是怎么实现原子性的?
1. 为什么只对mutex的变量做些原子性的修改，函数中还有很多其它变量为什么不用原子性改动?

下面让我们挨个回答下上面三个问题

### 为什么需要有mutexWoken
我们首先看lock中下面的代码
```
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
```
这段代码的作用就是在spin的过程中尝试设置mutexWoken位，这样下面unlock的时候有如下代码
```
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
```
代码中可以看到如果state状态中mutexWoken是置位的，那么直接返回了，没有调用runtime_Semrelease信号量操作去唤醒goroutine，这样做的好处是已经有唤醒的goroutine了没必要再多唤醒gorotine增加竞争。并且在lock的时候
```
		if awoke {
			// The goroutine has been woken from sleep,
			// so we need to reset the flag in either case.
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			new &^= mutexWoken
		}

		if atomic.CompareAndSwapInt32(&m.state, old, new) {
```
这样可以在设置新状态成功时，将woken置为0，这样如果获得了锁则将woken恢复，否则如果陷入了休眠woken状态也得到了维护，其它线程unlock的时候可以唤醒一个goroutine。

总结一下就是用来在某个协程spin之后设置该位可以减少其它gouroutine的唤醒，以及增加自己获得该锁的概率。

### atomic.CompareAndSwapInt32是怎么实现原子性的
首先我们先找到这个函数对应的代码如下:
src/runtime/internal/atomic/asm_amd64.s
```
// bool Cas(int32 *val, int32 old, int32 new)
// Atomically:
//	if(*val == old){
//		*val = new;
//		return 1;
//	} else
//		return 0;
TEXT runtime∕internal∕atomic·Cas(SB),NOSPLIT,$0-17 
    MOVQ    ptr+0(FP), BX
    // AX要用来存参数old
    MOVL    old+8(FP), AX
    // 把new中的数存到寄存器CX中
    MOVL    new+12(FP), CX
    // 注意这里了，这里使用了LOCK前缀，所以保证操作是原子的
    LOCK
    // 0(BX) 可以理解为 *val
    // 把 AX中的数 和 第二个操作数 0(BX)——也就是BX寄存器所指向的地址中存的值 进行比较
    // 如果相等，就把 第一个操作数 CX寄存器中存的值 赋给 第二个操作数 BX寄存器所指向的地址
    // 并将标志寄存器ZF设为1
    // 否则将标志寄存器ZF清零
    CMPXCHGL    CX, 0(BX)
    // SETE的作用是：
    // 如果Zero Flag标志寄存器为1，那么就把操作数设为1
    // 否则把操作数设为0
    // 也就是说，如果上面的比较相等了，就返回true，否则为false
    // ret+16(FP)代表了返回值的地址
    SETEQ    ret+16(FP)
    RET

```
首先Cas指令就是如果addr指向的地址中存的值和old一样，那么就把addr中的值改为new并返回true；否则什么都不做，返回false。但是注意CMPXCHGL指令需要去读取BX寄存器中所指向的内存值，如果在多核中其它核可能有对该内存的读写，首先CMPXCHGL指令本身不是原子的它需要先去内存中读取然后再比对，因为拆分成了两步，那么在这期间内存可能有了修改，比对结果有问题，所以前面添加了LOCK指令，该指令主要用来锁住总线，避免其它指令对内存的读写，这样就保证了该指令的原子性。

### 为什么只对mutex的变量做些原子性的修改，函数中还有很多其它变量为什么不用原子性改动?
因为函数中的变量在不同的协程中是分配到协程自己的栈的，不存在竞争，都私有属于协程本身，所以跟其它协程不存在竞争关系，不用原子性读写。但是mutex中的state和sem是所有协程共有的，故需要控制原子性读写。

## 总结
自此我们讲完了mutex的整个实现原理，整个代码非常简洁，但涉及到的知识点和原理很多，希望你至此已经收获满满, 知其然而且知其所以然~


