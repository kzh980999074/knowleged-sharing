# 问题题集

## 1 make 和 new 的区别

共同点：
    给变量申请内存，并且都是在堆上

区别：
返回值的不同：new 返回结构体指针 主要针对结构体类型基本数据类型, make 返回变量本身。
针对类型的不同 new主要为基本的数据类型和自定义结构体分配内存空间，对于new出的make和chanel不能直接进行操作，make主要针对slice,channel,map结构初始化

## 2 数组和切片的区别
共同点：
1. 都是存储一组某种类型的数据
2. 都可以通过len函数访问其长度

区别：
1. 数组是的长度是固定的，在创建的时候分配固定大小的内存，数组在定义的时候其长度是一个常量值
2. 切片是一个内置的复合的结构体，底层包含数组，长度和容量信息，并且切片长度可变，随着长度的增长会实现自动扩容，初始化的时候可以以变量值作为初始的长度
   
## 3 for range 

for a,b := range bs 中a,b 只会被创建一次 之后重复利用a,b

## 4 defer panic recover 

## 4.1 defer
defer是延迟调用函数，调用顺序 LIFO

## 4.2 panic

## 4.3 recover

## 5 rune

rune 类型是 int32 的别称，常用来处理string 或者 utf-8

## 6 反射

reflect包实现了运行时反射，允许程序操作任意类型的对象，典型的用法是interface{}保存的值 通过反射解析出来
反射的功能:
1. 在运行时获取数据的类型，结构体中中定义的变量名称 (typeof)
2. 可以获取任意变量的值（valueof）   

## 7 golang函数参数传递
golang的函数参数传递方式都是按值传递（pass by value）

## 8 常用的数据结构
1. slice
    golang的slice结构由三个字段组成，分别是data,len,cap. data表示slice底层使用的数组，len表示当前切片的长度，cap 代表当前切片的容量也就是data数组的长度，当容量等于长度的时候，往切片中增加数据触发底层数据的扩容操纵。
2. map


## 9 golang select case default
select 在golang中实现了io多路复用机制，select语句用来监听和chanel有关的io操作。当有io事件发生时，会触发相应的case动作，其中每个case都必须是一个通道。如果 select中存在default 那么当select没有事件发生的时候会去处理默认的default流程。如果不存在default，没有事件发生时，select语句会阻塞。
对于for select 不建议添加default，会造成cpu空耗。
当同时有多个事件发生 select的执行顺序是随机的。
select的特性
1. 至少要有一个case ，当出现读写nil的channel时，相应分支会忽略该事件
2. select仅支持管道
3. 每个case处理一个管道 或读或写
4. 多个事件同时发生，执行顺序随机
5. 存在default语句将不会造成select阻塞

## 10 单引号，双引号，反引号的区别
1. 单引号一般表示byte形的值
2. 双引号表示字符串,可以使用len函数获取字符串所占用的字节数
3. 表示字符串字面常量，里面的字符不支持转义

## 11 context
    context 上下文，负责在gorontinue之间传递过期时间，取消信号，和值

context由四个接口构成分别是:
1. Deadline() (deadline time.Time, ok bool)  返回超时时间，过期则取消这个context
2. Done() <-chan struct{} 返回一个channel，用于接受context的取消信号，或者deadline信号。监听done信号的函数会立即放弃当前正在执行的任务并返回。
3. Err() error 从error中可以知道context被取消的原因
4. Value(key interface{}) interface{}  goroutinue之间共享数据，需要保证数据的并发安全问题

context通过构建groutinue之间的树状关系的context来完成上层对下层的控制。


## 12 channel
    channel golang中的通道,用于多个协程之间的通信，（go的csp模型）
根据变量的定义方式可以分为 双向channel 只读channel 只写channel
channel需要使用make函数进行初始化
根据在声明时，分配的缓存数量可以分为 无缓冲channel带缓冲channel
```go
    //无缓冲channel 缓冲区是0
    ch1 := make(chan int)
    //带缓冲channel 缓冲区是10
    ch2 := make(chan int,10)
```



    

## 13 map



