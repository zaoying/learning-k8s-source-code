# 基本类型

参考[深入解析go语言](https://www.w3cschool.cn/go_internals/go_internals-tdys282g.html)

以go-1.8.3进行研究

## 在内存中的形式
首先看一下在go中，一些基础类型在内存中是以什么形态存在的，如下图所示：

![基础类型在内存中的存在形式](../images/基础类型在内存中的存在形式.png)

变量j的类型是int32， 而变量i的类型是int，两者不是同一个类型，所以赋值操作`i=j`是一种类型错误`cannot use j (type int32) as type int in assignment`。 

正确的方式应该是
```go
	i := int(7)
	j := int32(7)
	i = int(j)
```

结构体的域在内存中是紧密排列的。

## 静态类型和底层类型
byte是Go的静态类型，uint8是Go的底层类型

rune是int32的别名，用于表示unicode字符。通常在处理中文的时候需要用到它

## string类型
定义在`go-1.8.3/go/src/runtime/string.go`
```go
type stringStruct struct {
        str unsafe.Pointer
        len int
}
```
两个属性：一个指针，一个int型的长度，都是私有成员！

![go对string类型的定义](../images/go对string类型的定义.png)

string类型类型在Go语言的内存模型中用一个2字长的数据结构表示。 
从上图可以看出，其实多个string是共享一个存储的。

`str[i:j]`进行字符串切片操作，会得到一个新的`type stringStruct struct`对象，该对象的指针依然还是指向str的底层存储，长度为`j-i`。 
这说明字符串切分不涉及内存分配或复制操作，其效率等同于传递下标。

内建函数len()对string类型的操作是直接从底层结构中取出len值，而不需要额外的操作

## slice类型
定义在`/go-1.8.3/src/runtime/slice.go`
```go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```
显然，type slice struct和上面的type stringStruct struct很类似，只是多了一个cap属性。

![slice的定义](../images/slice的定义.png)

一个slice是一个底层数组的部分引用。同理，对底层数据进行切片操作也不会涉及到内存分配或复制操作，仅仅是新建了一个`type slice struct`对象而已！

需要注意的是，在上图中，y[0:4]是有效的，打印出来的结果会是`[3，5，7，11]`

由于slice是不同于指针的多字长结构，分割操作并不需要分配内存，甚至没有通常被保存在堆中的slice头部。这种表示方法使slice操作和在C中传递指针、长度对一样廉价。

slice相关的函数有如下几个，是不是感觉很熟悉。
```go
func makeslice(et *_type, len64, cap64 int64) slice
func growslice(et *_type, old slice, cap int) slice 
func slicecopy(to, fm slice, width uintptr) int
func slicestringcopy(to []byte, fm string) int 
```

### slice的扩容
对slice进行append操作时，可能会造成扩容操作。扩容规则如下：
  - 如果新的大小是当前大小2倍以上，则大小增长为新大小
  - 否则循环以下操作：如果当前长度len小于1024，按每次2倍增长，否则每次按当前cap的1/4增长。直到增长的大小超过或等于新大小。

```go
	newcap := old.cap
	doublecap := newcap + newcap
	//和old.cap的double进行比较
	if cap > doublecap {
		newcap = cap
	} else {
		if old.len < 1024 {
			newcap = doublecap
		} else {
			for newcap < cap {
				newcap += newcap / 4
			}
		}
	}
```

### slice与unsafe.Pointer相互转换
1. 利用make+slice弄块内存出来自己管理。。。。这个比较牛逼了
```go
s := make([]byte, 200)
ptr := unsafe.Pointer(&s[0])
```

2. 基于内存指针ptr构造出一个slice
```go
var o []byte
sliceHeader := (*reflect.SliceHeader)((unsafe.Pointer(&o)))
sliceHeader.Cap = length
sliceHeader.Len = length
sliceHeader.Data = uintptr(ptr)
```

## new和make
了解完Go对`type slice struct`的定义之后，再来理解new和make的差异就简单得多了。

- new(T)，仅分配内存，不进行初始化。返回的`＊T`指向一个类型T的零值。
- make(T, args)，分配内存，且进行初始化。返回是`T`本身。因为T本身就是一个引用类型。

以下属声明的类型为例子，分别用new和make的效果如下图：
```go
type Point struct { X, Y int }
type Rect1 struct { Min, Max Point }
type Rect2 struct { Min, Max *Point }
```

![new和make的区别](../images/new和make的区别.png)

## 参考
本系列参考[深入解析go语言](https://www.w3cschool.cn/go_internals/go_internals-tdys282g.html)
