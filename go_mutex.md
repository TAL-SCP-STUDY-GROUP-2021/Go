[TOC]

# Go 的互斥锁

互斥锁，也称互斥量，是实现同步机制的重要手段之一。在 Go 中，互斥量定义在 `sync` 包中的 `mutex.go` 中。`sync` 包提供了基础的同步原语，它们是相对原始的同步机制，比较适合用于低层级的库进程，多数情况下，高层级的同步可以用抽象层级更高的 channel 和通信更好地完成。

`sync.Mutex` 是一个不可重入锁。如它的名字所表示的一样，「相互独占」（mutual exclusion），是一个排他的作用量。当一个 goroutine 获得了锁的所有权后，其他请求这个锁的 goroutine 就会阻塞在试图获取锁的所有权的 `Lock()` 方法的调用上，直至锁被当前持有者释放。

## 锁的定义

### 数据结构

```go
type Mutex struct {
	state int32
	sema  uint32
}
```

Mutex 的数据结构不大，只占 8 个字节，由两个字段构成：`state` 记录当前锁的状态；`sema` 是 semaphore 的缩写，是用于控制锁的状态的信号量。

-   Mutex 的零值是一个未上锁的互斥量；
-   一旦使用过后 Mutex 就**不应该被复制**；

### 常量和状态机

```go
const (
	mutexLocked = 1 << iota // mutex is locked
	mutexWoken	// 2
	mutexStarving	// 4
	mutexWaiterShift = iota	// 3

	starvationThresholdNs = 1e6
)

// 通过位操作作用于Mutex.state
 +----------------------------------------+---------------+------------+-------------+
 |                                        |               |            |             |
 |             waitersCounter             | mutexStarving | mutexWoken | mutexLocked |
 |                                        |               |            |             |
 ^----------------------------------------^---------------^------------^-------------^
 |                                        |               |            |             |
 32                                       3               2            1             0
```

同名文件中还使用 `iota` 标识符定义了 Mutex 的状态机常量。

这些常量分别对应状态在状态变量 `Mutex.state` 中的位索引，通过位操作作用于 `Mutex.state` 上以相关状态进行读取或修改。这些常量在 `Mutex.state` 中从低位到高位依次为：

-   第 0 位 `mutexLocked`： Mutex 的锁定状态，0 表示未上锁，1表示已经上锁；
-   第 1 位 `mutexWoken` ：表示从正常模式被唤醒；
-   第 2 位 `mutexStarving` ：Mutex 的饥饿状态，0 表示正常模式，1 表示饥饿模式；
-   第 3 位 `mutexWaiterShift`：waitersCount 的起始偏移量。

waitersCount 是当前在 Mutex 上等待着的 goroutine 的个数。

默认情况下，Mutex 的所有状态位都是 0。

除了表示 `Mutex.state` 中的锁定状态位索引外，当 `mutexLocked` 常量出现在 `==` 算符右侧时还用来表示 Mutex 处于锁定状态。

此外还有一个常量 `starvationThresholdNs` ，它定义了饥饿阈值，表示当一个 goroutine 等待超过 1e6 纳秒，即 1 毫秒，会被标记为饥饿状态。

### Mutex 的公平性设计

为了保证锁对 goroutine 的公平性，Mutex 有两种模式：正常(normal)模式和饥饿(starvation)模式。

#### 正常模式

等待者按照 FIFO 的顺序排队获取 Mutex，一个被唤起的等待者不会直接获取锁，而是要与新创建的 goroutine 争夺锁的所有权。后者在这种模式下是更具优势的——它们已经运行在 CPU 上，并且可能有很多个这样的 goroutine，所以一个刚刚被唤起的 goroutine 很有可能竞争不过新到达的 goroutine。在这种情况下，唤醒的 goroutine 会被**置于队列的头部**。为了减少这种情况的出现，**如果一个等待者超过 1 ms 没有获取到锁，那么它会把当前的 Mutex 切换到饥饿模式**。

#### 饥饿模式

饥饿模式下，锁的所有权会被直接从解锁它的 goroutine 转交给等待队列最前面的 goroutine。新创建的 goroutine 既不会试图获取锁的所有权，也不会自旋，而是会把自己添加到队列的尾部。

#### 饥饿模式的解除

当一个等待者获取到锁的所有权且满足下面两个条件中的一个时，它会把 Mutex 切换回正常模式：

1.   它是队列中的最后一个等待者；
2.   它的等待时间不足 1 ms。

正常模式通常具有更好的性能表现，但是饥饿模式也是必要的，它能避免 goroutine 由于陷入等待而无法获取锁造成的病态尾延迟。

>   尾延迟 Tail latency
>
>   Tail latency is the small percentage of response times from a system, out of all of responses to the input/output (I/O) requests it serves, that take the longest in comparison to the bulk of its response times.
>
>   https://www.computerweekly.com/opinion/Storage-How-tail-latency-impacts-customer-facing-applications

## 加锁

加锁动作是依靠 `sync.Mutex.Lock` 方法完成的。方法的主干部分保留了最常见和简单的情形——当 Mutex 的状态是零值时，直接将 `mutexLocked` 状态位置 1。

```go
// Lock 锁住线程
// 如果lock已经生效，那么调用这个方法的goroutine会被阻塞直到Mutex可获取
func (m *Mutex) Lock() {
    // 快速通道：如果Mutex的状态机处于未加锁状态，那么获得锁
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}
	// Slow path (outlined so that the fast path can be inlined)
	m.lockSlow()
}
```

对更复杂的情况的处理放在了 `sync.Mutex.lockSlow` 方法里。调用了该方法的 goroutine 尝试通过自旋的方式等待锁的释放，方法的主体是一个 for 循环，在这个循环中，goroutine 获取锁的过程分为如下几个部分：

1.   判断当前 goroutine 能否进入自旋；
2.   自旋等待
3.   todo
4.   todo

### 自旋准入

```go
func (m *Mutex) lockSlow() {
	var waitStartTime int64	// 当前goroutine的等待时间
	starving := false	// 当前goroutine是否处于饥饿状态
	awoke := false	// 当前goroutine是否已唤醒
	iter := 0	// 自旋次数
	old := m.state	// 读取Mutex当前状态
	for {
        // 自旋准入判断
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true	// 当前goroutine标记自己被唤醒
			}
			runtime_doSpin()	// 自旋
			iter++
			old = m.state
			continue
		}
    // ...
```

自旋是一种多线程同步机制，当前的进程在进入自旋的过程中会一直保持 CPU 的占用，持续检查某个条件是否为真。在多核的 CPU 上，自旋可以避免 goroutine 的切换，使用恰当会对性能带来很大的增益，但是使用的不恰当就会拖慢整个程序，所以 goroutine 进入自旋的条件较为保守，要同时满足以下两个条件：

1.   互斥锁须为普通模式；
2.   `sync_runtime_canSpin` 函数需要返回 `true`。

`sync_runtime_canSpin` 函数位于 src/runtime/proc.go 的 runtime package：

```go
// Active spinning for sync.Mutex.
//go:linkname sync_runtime_canSpin sync.runtime_canSpin
//go:nosplit
func sync_runtime_canSpin(i int) bool {
	// sync.Mutex is cooperative, so we are conservative with spinning.
	// Spin only few times and only if running on a multicore machine and
	// GOMAXPROCS>1 and there is at least one other running P and local runq is empty.
	// As opposed to runtime mutex we don't do passive spinning here,
	// because there can be work on global runq or on other Ps.
	if i >= active_spin || ncpu <= 1 || gomaxprocs <= int32(sched.npidle+sched.nmspinning)+1 {
		return false
	}
	if p := getg().m.p.ptr(); !runqempty(p) {
		return false
	}
	return true
}
```

通过上面的代码可以得知，条件 2 成立的条件是：

1.   机器是多核 CPU，且 GOMAXPROCS>1；
2.   当前 goroutine 为了获取该锁而进入自旋的次数小于 `active_spin` 次，它的默认值是 4；
3.   当前机器上至少存在一个正在运行的处理器 P，且处理的运行队列 `runq` 为空。

在满足上述条件时，goroutine 会调用 `runtime.sync_runtime_doSpin` 函数，执行 30 次 `PAUSE` 指令，该指令只会占用 CPU 并消耗 CPU 时间。

```go
//go:linkname sync_runtime_doSpin sync.runtime_doSpin
//go:nosplit
func sync_runtime_doSpin() {
	procyield(active_spin_cnt)
}
```

在 x86_64 机器上，`procyield` 的汇编代码是：

```assembly
TEXT runtime·procyield(SB),NOSPLIT,$0-0
	MOVL	cycles+0(FP), AX
again:
	PAUSE
	SUBL	$1, AX
	JNZ	again
	RET
```

### 更新锁状态

#### 计算期望状态

自旋相关的逻辑结束后，当前 goroutine 要根据上下文计算它期望 Mutex 将要迁移到的状态 `new`，后者继承自当前的 Mutex 状态 `old`。对 Mutex 的 4 个状态字段 `mutexLocked`、`mutexWaiterShift`、`mutexStarving` 和 `mutexWoken`  的更新通过 4 个逻辑判断来完成。

```go
		new := old
		// Don't try to acquire starving mutex, new arriving goroutines must queue.
		if old&mutexStarving == 0 {
			new |= mutexLocked
		}
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift
		}
		// The current goroutine switches mutex to starvation mode.
		// But if the mutex is currently unlocked, don't do the switch.
		// Unlock expects that starving mutex has waiters, which will not
		// be true in this case.
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving
		}
		if awoke {
			// The goroutine has been woken from sleep,
			// so we need to reset the flag in either case.
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			new &^= mutexWoken
		}
```

1.   如果 Mutex 正处于正常模式，那么当前 goroutine 要参与锁竞争，它期望给 Mutex 加锁；

2.   Mutex 状态中 `mutexLocked` 和 `mutexStarving` 字段有 4 种组合，当它们不同时为 0 时，当前 goroutine 要加入等待队列，因此等待者计数器需要加1；

     | `mutexLocked` | `mutexStarving` |
     | :-----------: | :-------------: |
     |       0       |        0        |
     |       0       |        1        |
     |       1       |        0        |
     |       1       |        1        |

3.   如果当前 goroutine 是饥饿、且 Mutex 是锁定的，那么它期望将 Mutex 切换到饥饿模式；

4.   如果当前 goroutine 是被唤起的（而非新创建的），那么要把 `mutexWoken` 清空。

     >   &^ 是bit clear运算符，对于 z = x &^ y，同一位上，如果y是1，那么结果置零，否则与x保持一致。
     >
     >   https://go.dev/ref/spec#Arithmetic_operators

计算了期望状态后，当前 goroutine 尝试通过 CAS 操作对 Mutex 进行更新。

#### CAS 成功

CAS 操作返回 1 表明当前 goroutine 成功地将 Mutex 的状态修改为了期望值。

如果此前 Mutex 是非饥饿且非锁定的，那么表明 goroutine 获取到了 Mutex，并成功地为其加了锁，不需要做额外的操作，直接退出 for 循环即可（为什么等待者计数器没有减去 1 呢？因为 `mutexLocked` 和 `mutexStarving` 同时为 0 时计数器未曾计入当前 goroutine）；否则（Mutex 此前是饥饿的或者锁定的，或两者同时成立），Mutex 的争夺仍未结束，这一部分也是 `Lock` 中最抽象的环节。

如果当前 goroutine 此前的排队开始时间不为 0（即参与过等待），那么就要把自己加入队列的方式应当是“后进先出”，接着将排队开始时间更新到当前时间。接下来调用 `runtime.sync_runtime_SemacquireMutex` 通过信号量保证 Mutex 资源不会被两个 goroutine 同时获取。`runtime.sync_runtime_SemacquireMutex` 会在方法中不断尝试获取锁并陷入休眠等待信号量的释放，一旦当前 goroutine 可以获取信号量，它就会立刻返回，剩余代码也会继续执行。

当前 goroutine 获取到 Mutex 资源后，需要根据等待时间设置自己的饥饿标记，goroutine 会依据当前 Mutex 的模式而执行不同的操作。

-   如果 Mutex 处于饥饿模式，那么需要对 Mutex 状态不一致问题进行检查以及对状态进行修正；
    1.   如果当前 goroutine 是被唤起的且 Mutex 处于饥饿模式，那么 Mutex 的所有权已经移交给当前 goroutine，但 Mutex 因为某种原因处于一种不一致的状态：`mutexLocked` 未被设定、当前 goroutine 也还被当作等待者；
    2.   在修正值中将等待者数量减去 1（当前 goroutine）；
    3.   如果当前 goroutine 是非饥饿的或者是队列中的最后一个等待者，那么饥饿->正常模式的切换条件便得到了满足，因此修正值中把 `mutexStarving` 字段置零。
    4.   最后使用原子操作将修正值作用在 Mutex 的状态位上，当前 goroutine 获得了 Mutex 的所有权、加锁成功，退出 for 循环。
-   如果 Mutex 处于正常模式，将自己标记为被唤醒并重置自旋次数，重新执行 for 循环以争夺 Mutex。

```go
if atomic.CompareAndSwapInt32(&m.state, old, new) {
			if old&(mutexLocked|mutexStarving) == 0 {
				break // locked the mutex with CAS
			}
			// If we were already waiting before, queue at the front of the queue.
			queueLifo := waitStartTime != 0
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}
			runtime_SemacquireMutex(&m.sema, queueLifo, 1)
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state
			if old&mutexStarving != 0 {
				// If this goroutine was woken and mutex is in starvation mode,
				// ownership was handed off to us but mutex is in somewhat
				// inconsistent state: mutexLocked is not set and we are still
				// accounted as waiter. Fix that.
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				if !starving || old>>mutexWaiterShift == 1 {
					// Exit starvation mode.
					// Critical to do it here and consider wait time.
					// Starvation mode is so inefficient, that two goroutines
					// can go lock-step infinitely once they switch mutex
					// to starvation mode.
					delta -= mutexStarving
				}
				atomic.AddInt32(&m.state, delta)
				break
			}
			awoke = true
			iter = 0
		}
```

#### CAS 失败

CAS 操作返回 0 表明当前 goroutine 所掌握的 Mutex 状态与其最新状态不一致，修改 Mutex 状态的尝试是失败的（更不用提获取它的所有权），重新读取 Mutex 状态并进入到下一个循环。

```go
		} else {
			old = m.state
		}
	}

	if race.Enabled {
		race.Acquire(unsafe.Pointer(m))
	}
}
```

### 总结

1.   当 `break` 出现时才表明 goroutine 获取锁成功，能够退出 `Lock` 的 for 循环；