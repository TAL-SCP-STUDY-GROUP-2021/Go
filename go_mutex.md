# Go 的互斥锁

互斥锁，也称互斥量，是实现同步机制的重要手段之一。在 Go 中，互斥量定义在 `sync` 包中的 `mutex.go` 中。`sync` 包提供了基础的同步原语，比较适合用于低层级的库进程，高层级的同步可以用通道和通信更好地完成。

`sync.Mutex` 是一个不可重入锁，如它的名字所表示的一样，「相互独占」（mutual exclusion），是一个排他的作用量。当一个 goroutine 获得了锁的所有权后，其他请求这个锁的 goroutine 就会阻塞在试图获取锁的所有权的 `Lock()` 方法的调用上，直至锁被当前持有者释放。

## 锁的定义

### 数据结构

```go
type Mutex struct {
	state int32
	sema  uint32
}
```

Mutex 由两个字段构成：`state` 记录当前锁的状态，`sema` 是 semaphore 的缩写，是用于控制锁的状态的信号量。

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
 |               等待者计数器               | mutexStarving | mutexWoken | mutexLocked |
 |                                        |               |            |             |
 ^----------------------------------------^---------------^------------^-------------^
 |                                        |               |            |             |
 32                                       3               2            1             0
```

同名文件中还使用 `iota` 标识符定义了 Mutex 的状态机常量。

这些常量分别对应状态在状态变量 `Mutex.state` 中的位索引，通过位操作作用于 `Mutex.state` 上以相关状态进行读取或修改。这些常量在 `Mutex.state` 中从低位到高位依次为：

-   第 0 位 `mutexLocked`： Mutex 的上锁状态，0 表示未上锁，1表示已经上锁；
-   第 1 位 `mutexWoken` ：表示从正常模式被唤醒；
-   第 2 位 `mutexStarving` ：Mutex 的饥饿状态，0 表示正常模式，1 表示饥饿模式；
-   第 3 位 `mutexWaiterShift`：等待 Mutex 的 goroutine 的计数器的起始偏移量。

除了表示 `Mutex.state` 中的锁定状态位索引外，当 `mutexLocked` 出现在 `==` 算符右侧时还用来表示 Mutex 的锁定状态。

此外还有一个常量 `starvationThresholdNs` ，它定义了饥饿阈值，当表明一个 goroutine 等待超过 $10^{6}$ 纳秒，即 1 毫秒，会被标记为饥饿状态。

### Mutex 的公平性设计

为了保证锁对 goroutine 的公平性，Mutex 有两种模式：正常(normal)模式和饥饿(starvation)模式。

#### 正常模式

等待者按照 FIFO 的顺序排队，一个唤醒的等待者不会直接获取锁，而是要与新到达的 goroutine 争夺锁的所有权。新到达的 goroutine 在这种模式下是更具优势的——它们已经运行在 CPU 上，并且可能有很多个这样的 goroutine，所以一个刚刚唤醒的 goroutine 很有可能竞争不过新到达的 goroutine。在这种情况下，唤醒的 goroutine 会被**置于队列的头部**。**如果一个等待者在 1 ms 之内都没有获取到锁，那么它会把 Mutex 切换到饥饿模式**。

#### 饥饿模式

饥饿模式下，锁的所有权会被直接从解锁它的 goroutine 转交给队列头部的第一个等待者。新到达的 goroutine 既不会试图获取锁的所有权，也不会自旋，而是会把自己添加到队列的尾部。

#### 饥饿模式的解除

当一个等待者获取到锁的所有权且满足下面两个条件中的一个时，它会把 Mutex 切换回正常模式：

1.   它是队列中的最后一个等待者；
2.   它的等待时间不足 1 ms。

正常模式通常具有更好的性能表现，但是饥饿模式也是必要的。

## 上锁

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

func (m *Mutex) lockSlow() {
	var waitStartTime int64	// 当前goroutine的等待时间
	starving := false	// 当前goroutine是否处于饥饿状态
	awoke := false	// 当前goroutine是否已唤醒
	iter := 0	// 自旋次数
	old := m.state
	for {
        // 如果Mutex处于(锁定,正常模式)态，且runtime支持自旋
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			// Active spinning makes sense.
			// Try to set mutexWoken flag to inform Unlock
			// to not wake other blocked goroutines.
            // 如果Mutex.state还没有设置woken标识则设置它，以防止Unlock激活其他阻塞goroutine
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true	// 当前goroutine标记自己被唤醒
			}
			runtime_doSpin()	// 自旋
			iter++
			old = m.state
			continue
		}
        
        // 程序运行到这里表明Mutex.state处于下面几种状态之一：
        // 1. (锁定,饥饿模式)
        // 2. (未锁定,正常模式)
        // 3. (未锁定,饥饿模式)
        
        // new描述了当前goroutine想要把Mutex迁移到什么状态
        // 它以现有的state为起点
		new := old
        
        // goroutine不会试图取得饥饿模式的Mutex的锁的所有权
        // 但如果Mutex不是饥饿的，它期望新到达的goroutine能够排队（而非抢占）
        // 同时它想锁定Mutex
		if old&mutexStarving == 0 {
			new |= mutexLocked
		}
        
        // 如果当前state满足 1.锁定、2.饥饿模式 的任何一种
        // 那么当前goroutine要为排队计数器加1
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift
		}
        
        // 如果当前goroutine已经饥饿且Mutex是锁定的
        // 那么当前goroutine期望把Mutex切换到饥饿模式
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving
		}
        
        // 如果当前goroutine已经被唤醒
		if awoke {
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
            
            // 把Mutex.state的唤醒位清空
			new &^= mutexWoken
            // &^ 是bit clear运算符
            // z = x &^ y
            // 同一位上，如果y是1，那么结果置零，否则与x保持一致
            // https://go.dev/ref/spec#Arithmetic_operators
		}
        
        // 如果当前goroutine修改Mutex.state成功
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
            
            // 如果此前Mutex是(未锁定,正常模式)状态
			if old&(mutexLocked|mutexStarving) == 0 {
				break // 退出循环，此时Mutex已经锁上了
			}
            
            // 如果当前goroutine此前就已经在等待了，那么插入到队首
            
            // 判断要不要插入到队首
			queueLifo := waitStartTime != 0	// bool
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}
			runtime_SemacquireMutex(&m.sema, queueLifo, 1)
            // 通过等待时间判断当前goroutine是否已经饥饿
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
            // 读取一下新修改的Mutex.state
			old = m.state
            // 如果监测到Mutex仍处于饥饿模式
			if old&mutexStarving != 0 {
                // 此时goroutine已经唤醒了，锁的所有权也递交到当前goroutine了
                // 但是如果Mutex此时处于饥饿模式，那么Mutex就进入了不一致状态：
                // Mutex锁状态未被设定，当前goroutine也仍被视作等待者，这需要修正
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}
                
                // 对Mutex.state的修正值：排队计数器减去1
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
                
                // 如果当前goroutine并非饥饿状态或者Mutex的排队计数器为1
				if !starving || old>>mutexWaiterShift == 1 {
                    // 退出饥饿模式
					delta -= mutexStarving
				}
                
                // 对Mutex状态进行修正
				atomic.AddInt32(&m.state, delta)
				break
			}
			awoke = true
			iter = 0
		} else {	// 如果当前goroutine修改Mutex.state失败
            // 那么就获取一下最新的Mutex.state,不做其他事情
			old = m.state
		}
	}

	if race.Enabled {
		race.Acquire(unsafe.Pointer(m))
	}
}
```



## 解锁

todo