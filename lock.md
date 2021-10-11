## 互斥锁

```
传统并发程序对共享资源进行访问控制的主要手段，由标准库代码包中sync中的Mutex结构体表示

// Mutex 是互斥锁，零值是解锁的互斥锁，首次使用后不得复制互斥锁。
type Mutex struct {
   state int32
   sema  uint32
}

//sync.Mutex类型只有两个公开的指针方法
//Locker表示可以锁定和解锁的对象。
type Locker interface {
   Lock()
   Unlock()
}

//锁定当前的互斥量
//如果锁已被使用，则调用goroutine
//阻塞直到互斥锁可用。
func (m *Mutex) Lock() 

//对当前互斥量进行解锁
//如果在进入解锁时未锁定m，则为运行时错误。
//锁定的互斥锁与特定的goroutine无关。
//允许一个goroutine锁定Mutex然后安排另一个goroutine来解锁它。
func (m *Mutex) Unlock()

// 如果对一个已经上锁的对象再次上锁，那么就会导致该锁定操作被阻塞，直到该互斥锁回到被解锁状态
// 互斥锁可以被多个协程共享，但还是建议将对同一个互斥锁的加锁解锁操作放在同一个层次的代码中
```



## 读写锁

```
读写锁是针对读写操作的互斥锁，可以分别针对读操作与写操作进行锁定和解锁操作 。
读写锁的访问控制规则如下：
	1 多个写操作之间是互斥的
	2 写操作与读操作之间也是互斥的
	3 多个读操作之间不是互斥的
在这样的控制规则下，读写锁可以大大降低性能损耗。

由标准库代码包中sync中的RWMutex结构体表示
// RWMutex是一个读/写互斥锁，可以由任意数量的读操作或单个写操作持有。
// RWMutex的零值是未锁定的互斥锁。
//首次使用后，不得复制RWMutex。
//如果goroutine持有RWMutex进行读取而另一个goroutine可能会调用Lock，那么在释放初始读锁之前，goroutine不应该期望能够获取读锁定。 
//特别是，这种禁止递归读锁定。 这是为了确保锁最终变得可用; 阻止的锁定会阻止新读操作获取锁定。
type RWMutex struct {
   w           Mutex  // 如果有待处理的写操作就持有
   writerSem   uint32 // 写操作等待读操作完成的信号量
   readerSem   uint32 // 读操作等待写操作完成的信号量
   readerCount int32  // 待处理的读操作数量
   readerWait  int32  // number of departing readers
}

sync中的RWMutex有以下几种方法：
//对读操作的锁定
func (rw *RWMutex) RLock()
//对读操作的解锁
func (rw *RWMutex) RUnlock()
//对写操作的锁定
func (rw *RWMutex) Lock()
//对写操作的解锁
func (rw *RWMutex) Unlock()

//返回一个实现了sync.Locker接口类型的值，实际上是回调rw.RLock and rw.RUnlock.
func (rw *RWMutex) RLocker() Locker

Unlock会试图唤醒所有因欲进行读锁定而被阻塞的协程，而 RUnlock 只会在已无任何读锁定的情况下，试图唤醒一个因欲进行写锁定而被阻塞的协程。若对一个未被写锁定的读写锁进行写解锁，就会引发一个不可恢复的panic，同理对一个未被读锁定的读写锁进行读写锁也会如此。

由于读写锁控制下的多个读操作之间不是互斥的，因此对于读解锁更容易被忽视。对于同一个读写锁，添加多少个读锁定，就必要有等量的读解锁，这样才能其他协程有机会进行操作。
```

## 自旋锁

```
自旋锁是指当一个线程在获取锁的时候，如果锁已经被其他线程获取，那么该线程将循环等待，然后不断地判断是否能够被成功获取，知直到获取到锁才会退出循环。
 获取锁的线程一直处于活跃状态，但是并没有执行任何有效的任务，使用这种锁会造成busy-waiting。
 它是为实现保护共享资源而提出的一种锁机制。其实，自旋锁与互斥锁比较类似，它们都是为了解决某项资源的互斥使用。无论是互斥锁，还是自旋锁，在任何时刻，最多只能由一个保持者，也就说，在任何时刻最多只能有一个执行单元获得锁。但是两者在调度机制上略有不同。对于互斥锁，如果资源已经被占用，资源申请者只能进入睡眠状态。但是自旋锁不会引起调用者睡眠，如果自旋锁已经被别的执行单元保持，调用者就一直循环在那里看是否该自旋锁的保持者已经释放了锁，“自旋”一词就是因此而得名。
 
type spinLock uint32
func (sl *spinLock) Lock() {
    for !atomic.CompareAndSwapUint32((*uint32)(sl), 0, 1) {
        runtime.Gosched()
    }
}
func (sl *spinLock) Unlock() {
    atomic.StoreUint32((*uint32)(sl), 0)
}
func NewSpinLock() sync.Locker {
    var lock spinLock
    return &lock
}
```

## 可重入的自旋锁

```
上面不支持重入的，即当一个线程第一次已经获取到了该锁，在锁释放之前又一次重新获取该锁，第二次就不能成功获取到。由于不满足CAS，所以第二次获取会进入while循环等待，而如果是可重入锁，第二次也是应该能够成功获取到的。

而且，即使第二次能够成功获取，那么当第一次释放锁的时候，第二次获取到的锁也会被释放，而这是不合理的。

为了实现可重入锁，我们需要引入一个计数器，用来记录获取锁的线程数

type spinLock struct {
      owner int
      count  int
}

func (sl *spinLock) Lock() {
        me := GetGoroutineId()
        if spinLock .owner == me { // 如果当前线程已经获取到了锁，线程数增加一，然后返回
               sl.count++
               return
        }
        // 如果没获取到锁，则通过CAS自旋
    for !atomic.CompareAndSwapUint32((*uint32)(sl), 0, 1) {
        runtime.Gosched()
    }
}
func (sl *spinLock) Unlock() {
      if  rl.owner != GetGoroutineId() {
          panic("illegalMonitorStateError")
      }
      if sl.count >0  { // 如果大于0，表示当前线程多次获取了该锁，释放锁通过count减一来模拟
           sl.count--
       }else { // 如果count==0，可以将锁释放，这样就能保证获取锁的次数与释放锁的次数是一致的了。
           atomic.StoreUint32((*uint32)(sl), 0)
       }
}

func GetGoroutineId() int {
    defer func()  {
        if err := recover(); err != nil {
            fmt.Println("panic recover:panic info:%v", err)     }
    }()

    var buf [64]byte
    n := runtime.Stack(buf[:], false)
    idField := strings.Fields(strings.TrimPrefix(string(buf[:n]), "goroutine "))[0]
    id, err := strconv.Atoi(idField)
    if err != nil {
        panic(fmt.Sprintf("cannot get goroutine id: %v", err))
    }
    return id
}

func NewSpinLock() sync.Locker {
    var lock spinLock
    return &lock
}
```

