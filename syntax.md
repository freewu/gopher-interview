## Golang可变参数

```
可变参数本质上是一个数组，所以我们向使用数组一样使用

func print(params ...interface{}) {
	for _,v := range params {
		fmt.Println(v)
	}
}
```

## Golang的参数传递、引用类型

```
Go语言中所有的传参都是值传递（传值），都是一个副本，一个拷贝。因为拷贝的内容有时候是非引用类型（int、string、struct等这些），这样就在函数中就无法修改原内容数据；有的是引用类型（指针、map、slice、chan等这些），这样就可以修改原内容数据。

Golang的引用类型包括 slice、map 和 channel。它们有复杂的内部结构，除了申请内存外，还需要初始化相关属性。内置函数 new 计算类型大小，为其分配零值内存，返回指针。而 make 会被编译器翻译成具体的创建函数，由其分配内存和初始化成员结构，返回对象而非指针。
```

## Map底层实现

```
Golang中map的底层实现是一个散列表，因此实现map的过程实际上就是实现散表的过程。在这个散列表中，主要出现的结构体有两个，一个叫hmap(a header for a go map)，一个叫bmap(a bucket for a Go map，通常叫其bucket)。

hmap:
	元素个数 int
	flags uint8
	扩容常量相关字段 uint8
	溢出的 bucket的个数 uint16
	用于扩容的指针 *mapextra
	buckets的数组指针 unsafe.Pointer
	搬迁进度  uintptr
	结构扩容的时候用于复制的buckets unsafe.Pointer
	hash seed uint32

bmap: (bucket)
	高位Hash值 [8]uint8  // 记录的是当前bucket中key相关的“索引”
	存储key和value的数组 []byte
	指向扩容的bucket的指针 pointer
	
Golang把求得的哈希值按照用途一分为二：高位和低位。低位用于寻找当前key属于hmap中的哪个bucket，而高位用于寻找bucket中的哪个key	
```

## Map的扩容

```
当Go的map长度增长到大于加载因子所需的map长度时，Go语言就会将产生一个新的bucket数组，然后把旧的bucket数组移到一个属性字段oldbucket中。

并不是立刻把旧的数组中的元素转义到新的bucket当中，而是，只有当访问到具体的某个bucket的时候，会把bucket中的数据转移到新的bucket中。
```

## defer

```
向 defer 关键字传入的函数会在函数返回之前运行。假设我们在 for 循环中多次调用 defer 关键字,会倒序执行所有向 defer 关键字中传入的表达式。

defer 传入的函数不是在退出代码块的作用域时执行的，它只会在当前函数和方法返回之前被调用

defer 关键字的实现主要依靠编译器和运行时的协作：

    编译期；
        将 defer 关键字被转换 runtime.deferproc；
        在调用 defer 关键字的函数返回之前插入 runtime.deferreturn；
    运行时：
        runtime.deferproc 会将一个新的 runtime._defer 结构体追加到当前 Goroutine 的链表头；
        runtime.deferreturn 会从 Goroutine 的链表中取出 runtime._defer 结构并依次执行；
        
后调用的 defer 函数会先执行：
    后调用的 defer 函数会被追加到 Goroutine _defer 链表的最前面；
    运行 runtime._defer 时是从前到后依次执行；
函数的参数会被预先计算；
	调用 runtime.deferproc 函数创建新的延迟调用时就会立刻拷贝函数的参数，函数的参数不会等到真正执行时计算；
```

## sync.Once

```
sync.Once 是 Golang package 中使方法只执行一次的对象实现，作用与 init 函数类似。但也有所不同。

init 函数是在文件包首次被加载的时候执行，且只执行一次
sync.Onc 是在代码运行中需要的时候执行，且只执行一次
当一个函数不希望程序在一开始的时候就被执行的时候，我们可以使用 sync.Once
```

## new() 与 make() 的区别

```
new(T)  和 make(T, args)  是Go语言内建函数，用来分配内存，但适用的类型不用。
new(T) 会为了 T 类型的新值分配已置零的内存空间，并返回地址（指针），即类型为 *T 的值。换句话说就是，返回一个指针，该指针指向新分配的、类型为 T 的零值。适用于值类型，如 数组 、 结构体 等。
make(T, args) 返回初始化之后的T类型的值，也不是指针 *T ，是经过初始化之后的T的引用。 make() 只适用于 slice 、 map 和 chennel 。
```



