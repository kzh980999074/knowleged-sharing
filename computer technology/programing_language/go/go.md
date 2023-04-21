### 1. map
#### map
    map 的核心就是数组+链表
    主要结构 hmap
    ```
    type hmap struct {
        count     int    // 元素的个数
        B         uint8  // buckets 数组的长度就是 2^B 个
        overflow uint16 // 溢出桶的数量 ​
        buckets    unsafe.Pointer // 2^B个桶对应的数组指针
        oldbuckets unsafe.Pointer  // 发生扩容时，记录扩容的buckets数组指针
        extra *mapextra //用于保存溢出桶的地址
    }
    // 编译期的新结构体
    type bmap struct {
        tophash [bucketCnt]uint8
        data  [] //数据 编译期生成的结构  
        overflow *bmap// 溢出桶的地址 
    }
    ```
#### sync.map
数据结构
```
    // Map 是一种并发安全的 map[interface{}]interface{}，在多个 goroutine 中没有额外的锁条件
    // 读取、存储和删除操作的时间复杂度平均为常量
    // Map 类型非常特殊，大部分代码应该使用原始的 Go map。它具有单独的锁或协调以获得类型安全且更易维护。
    // Map 类型针对两种常见的用例进行优化：
    // 1. 给定 key 只会产生写一次但是却会多次读，类似乎只增的缓存
    // 2. 多个 goroutine 读、写以及覆盖不同的 key
    // 这两种情况下，与单独使用 Mutex 或 RWMutex 的 map 相比，会显著降低竞争情况
    // 零值 Map 为空且可以直接使用，Map 使用后不能复制
    type Map struct {
        mu Mutex
        // read 包含 map 内容的一部分，这些内容对于并发访问是安全的（有或不使用 mu）。
        // read 字段 load 总是安全的，但是必须使用 mu 进行 store。
        // 存储在 read 中的 entry 可以在没有 mu 的情况下并发更新，
        // 但是更新已经删除的 entry 需要将 entry 复制到 dirty map 中，并使用 mu 进行删除。
        read atomic.Value // 只读

        // dirty 含了需要 mu 的 map 内容的一部分。为了确保将 dirty map 快速地转为 read map，
        // 它还包括了 read map 中所有未删除的 entry。
        // 删除的 entry 不会存储在 dirty map 中。在 clean map 中，被删除的 entry 必须被删除并添加到 dirty 中，
        // 然后才能将新的值存储为它
        // 如果 dirty map 为 nil，则下一次的写行为会通过 clean map 的浅拷贝进行初始化
        dirty map[interface{}]*entry // 存储的是指针的指针

        // misses 计算了从 read map 上一次更新开始的 load 数，需要 lock 以确定 key 是否存在。
        // 一旦发生足够的 misses 足以囊括复制 dirty map 的成本，dirty map 将被提升为 read map（处于未修改状态）
        // 并且 map 的下一次 store 将生成新的 dirty 副本。
        misses int
    }
    type entry struct {
    	p unsafe.Pointer // *interface{}
    }
```
写流程
1.  修改已存在的于readmap中值，通过atomic.LoadPointer和atomic.CompareAndSwapPointer 完成。不需要加锁
2.  加锁
3.  如果在readMap中已经删除的值，那么需要同事更新dirtyM和readM
4.  如果删除在dirtyM中已存在的值，直接更新dirtyM即可
5.  如果dirtyM中不存在，那么先判断是否存在dirtyM，（不存在则创建新的dirtyM,然后数据迁移）然后将新值写入dirtyM中。
6.  解锁


### defer 延迟调用语句
每个goroutinue会维护一个 defer结构的链表
当代码执行中遇到 defer语句，就会新生产一个 _defer结构体并且将延迟语句的参数返回值复制到结构体当中
然后采用头插法插入到当前的goroutinue中，所以满足 LIFO（先进后出）的执行顺序。
_defer数据结构的分配地 1. 栈 2. 堆 （嵌套循环的defer会分配在栈上）
不同的调用函数痛过_defer.sp字段区分
当前函数调用的return语句的执行步骤： 第一步 设置返回值 第二步 调用当前函数的defer语句  第三步 返回上层调用

### panic && recover 恢复内建函数
#### panic
```
// _panic结构体（链表形式与_defer类似）
type _panic struct {
    argp      unsafe.Pointer
    arg       interface{}    // panic 的参数
    link      *_panic        // 链接下一个 panic 结构体
    recovered bool           // 是否恢复，到此为止？
    aborted   bool           // the panic was aborted
}
```
panic本质是对runtime.gopanic的调用,panic内部调用的用户态代码只有defer函数（panic链表中存在多个元素只能是panic函数调用defer函数导致的panic）

panic无法被恢复的情况：
1. 系统栈上的 panic 
2. malloc发生的panic
3. 禁止抢占导致的panic
4. 发生在锁上的panic
其他情况的panic可以被恢复
当panic未能检测到recover函数的时候会调用exit(2)进行退出

#### recover
recover函数内部调用 runtime.gorecover
recover：从当前的goroutine中头部的_panic结构的recovered字段置为 true

#### channel 
channel主要结构体
```
    // src/runtime/chan.go
    type hchan struct {
        qcount   uint           // 队列中的所有数据数
        dataqsiz uint           // 环形队列的大小
        buf      unsafe.Pointer // 指向大小为 dataqsiz 的数组
        elemsize uint16         // 元素大小
        closed   uint32         // 是否关闭
        elemtype *_type         // 元素类型
        sendx    uint           // 发送索引
        recvx    uint           // 接收索引
        recvq    waitq          // recv 等待列表，即（ <-ch ）
        sendq    waitq          // send 等待列表，即（ ch<- ）
        lock mutex
    }
    type waitq struct { // 等待队列 sudog 双向队列
        first *sudog
        last  *sudog
    }
```
channel主要是由一个携带数据的环形队列，发送列表，接受列表和一把锁构成

向channel发送数据的流程：
1.  加锁操作
2.  拷贝要发送的数据：首先如果有等待接受的接受方，则将数据直接发送给接受方，然后返回。其次 如果数据缓存中有多余的空间，将数据拷贝到缓存，然后返回。最后 阻塞当前协程，知道该协程被唤醒
3.  释放锁

从channel中获取数据的流程
1. 加锁操作
2. 拷贝要接受的数据：第一步 如果channel被关闭，且没有数据那么立即返回，第二步 如果存在正在阻塞的发送方，直接从阻塞的发送方接受数据，然后复始该发送协程。第三步 如果缓存中存在数据，从缓存队列头部获取获取，并将缓存的数据量更新。 第四步 没有可供接受的数据，阻塞当前的接受方goroutine
3. 阻塞当前接受方goroutine

### 接口interface
接口是对其他类型行为的概括和抽象。
golang的接口通过interface字段声明，在接口中定义方法集。如果某个类型要实现一个接口，那么只需要实现接口定义的方法集即可。
如果T类型实现了接口I，那么*T类型也自动实现了接口I，此时，即可以把一个T类型的实例赋值给接口I的变量，也可以把一个*T类型的实例指针赋值给接口I的变量。不同之处 1.如果把T类型的实例赋值给接口变量，那么将拷贝该实例的数据结构到接口变量中 2.如果把*T类型的实例指针赋值给接口变量，那么仅拷贝指针值到接口变量中 3.如果将一个接口变量赋值给另一个接口变量，两个接口变量将会引用同一个实例。

接口底层分为 iface 和 eface(其中efece 描述的是不包含方法集的接口)
```
    //
    type iface struct {
        tab  *itab
        data unsafe.Pointer
    }
    // 没有方法的interface
    type eface struct {
        _type *_type
        data  unsafe.Pointer
    }
```


### 互斥锁 mutex
数据结构
```
    type Mutex struct {
        state int32  // 表示 mutex 锁当前的状态
        sema  uint32 // 信号量，用于唤醒 goroutine
    }
```
锁的两种状态： 正常模式和饥饿模式
加锁的流程：
1. 无冲突，通过 CAS 操作把当前状态设置为加锁状态
2. 有冲突，开始自旋，并等待锁释放，如果其他 goroutine 在这段时间内释放该锁，直接获得该锁；如果没有释放则为下一种情况
3. 有冲突，且已经过了自旋阶段，通过调用 semrelease 让 goroutine 进入等待状态

解锁的流程
1. 将锁的状态字段置为0
2. 非饥饿模式，唤醒一个阻塞的 goroutine（runtime_Semrelease）
3. 饥饿模式: 直接将 mutex 所有权交给等待队列最前端的 goroutine

### 原子操作
atomic.CompareAndSwapUintptr 本质上调用的是使用 CPU 的 LOCK`+CMPXCHGQ`

#### 原子值
atomic.Value数据结构
```
    type Value struct {
        v interface{}
    }
```
只是对存储的值进行了一层封装
其中两个方法Load和Store
Load方法只是对存储的数据进行读取
Store方法 1. 还未被写入过任何数据，会先将数据类型写入，数据类型的写入需要现将当前的goroutine设置为不可抢占的状态。然后将类型设置为一个特殊值（该特殊值标志有协程正在在对类型进行写入），然后原子存储值与类型。 2. 如果已经存在类型，那么对比类型 然后原子操作存储数据。

### 同步组 sync.WaitGroup
底层数据结构
```
    type WaitGroup struct {
        // 64 位值: 高 32 位用于计数，低 32 位用于等待计数
        // 64 位的原子操作要求 64 位对齐，但 32 位编译器无法保证这个要求
        // 因此分配 12 字节然后将他们对齐，其中 8 字节作为状态，其他 4 字节用于存储信号量原语
        state1 [3]uint32
    }
```
方法集
wg.Done 等同于 wg.Add(-1)
wg.Add
1. 首先获取状态指针和存储指针
2. 然后通过atomic.AddUint64将当前的值加到状态上
3. 如果 状态值为0，等待者大于0 则 runtime_Semrelease 释放信号量
wg.Wait
1. 首先获取状态指针和存储指针
2. 将等待者加1
3. runtime_Semacquire阻塞当前线程


### sync.Pool 临时对象池
数据结构
```
    type Pool struct {
        local     unsafe.Pointer // local 固定大小 per-P 数组, 实际类型为 [P]poolLocal
        localSize uintptr        // local array 的大小

        victim     unsafe.Pointer // 来自前一个周期的 local （相当于二级缓存）
        victimSize uintptr        // victim 数组的大小
        ...
    }
    type poolLocalInternal struct {
        private interface{} //仅用于当前P读写
        shared  poolChain
    }

    type poolLocal struct {
        poolLocalInternal
        pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
    }
```
#### get流程
1. 先将当前p与g进行绑定
2. 优先从当前P的 private 中选择对象
3. 若取不到，则尝试从当前P的共享队列的队头进行读取（从队列中获取数据是通过cas做到无锁获取数据）
4. 若取不到，则尝试从其他的 P 的共享队列中进行偷取 getSlow
5. 从victim中获取数据（victim是上个周期的local的数据）
6. 若还是取不到，则使用 New 方法新建

#### put流程
1. 绑定当前的g 与 p
2. 优先放入 private
3. 如果 private 已经有值，即不能放入，则尝试放入 shared

#### 缓存的回收
sync.Pool 的垃圾回收发生在运行时 GC 开始之前
1. 删除当前的 victim中的数据
2. 将主缓存移动到 victim 缓存


#### 优点
1. 引入了 victim （二级）缓存，每次 GC 周期不再清理所有的缓存对象，而是将 locals 中的对象暂时放入 victim ，从而延迟到下一个 GC 周期进行回收；
2. 在下一个周期到来前，victim 中的缓存对象可能会被偷取，在 Put 操作后又重新回到 locals 中，这个过程发生在从其他 P 的 shared 队列中偷取不到、以及 New 一个新对象之前，进而是在牺牲了 New 新对象的速度的情况下换取的
3. poolLocal 不再使用 Mutex 这类昂贵的锁来保证并发安全，取而代之的是使用了 CAS 算法优化实现的 poolChain 变长无锁双向链式队列
   

### go 垃圾回收
垃圾回收（GC)就是回收堆上不在使用的物理内存（栈上的内存会随着函数调用的结束自动回收）
当前版本go的垃圾回收算法采用的是并行的三色标记清扫算法
将对象分为三种颜色：分别是 白色，灰色，黑色
白色代表 潜在的垃圾对象
灰色代表 正在扫描的对象
黑色代表 扫描完的被引用的对象

#### 垃圾回收的基本流程
1. 初始时将所有的内存对象都标记为白色，将所有对象加入白色集合中
2. 标记准备阶段：初始化GC任务,开启混合写屏障，并且统计全局数据和goroutine的栈数据 （这个这个阶段会stop the world）
3. 标记阶段：从灰色集合中取出对象，将该对象关联的白色对象放入灰色集合中。 （和用户程序并发执行）
4. 标记结束阶段： 关闭混合写屏障（这个这个阶段会stop the world）。
5. 删除白色集合中剩余的对象 （和用户程序并发执行）

#### 标记阶段的注意事项
标记阶段堆上的数据增删：
三色不变性：
1. 强三色不变性：黑色对象不会指向白色对象，只会指向灰色对象或者黑色对象。
2. 弱三色不变性 ：黑色对象指向的白色对象必须包含一条从灰色对象经由多个白色对象的可达路径。

并发标记阶段需要满足所以引入混合写屏障技术（删除写屏障技术和插入写屏障技术）
插入写屏障：插入屏障拦截将白色指针插入黑色对象的操作，标记其对应对象为灰色
删除写屏障：删除屏障也是拦截写操作的，在将对象删除的时候会将该对象从白色标记为灰色
但是是通过保护灰色对象到白色对象的路径不会断来实现的，满足了弱三色不变性。
（注意：所有的屏障加在堆上）

标级阶段栈上数据的增删
栈上新增的数据直接将其标记为黑色
栈上删除的数据不做处理

### iota关键字
1.iota在const关键字出现时将被重置为0。
2.const中每新增一行常量声明将使iota计数一次(iota可理解为const语句块中的行索引)。
