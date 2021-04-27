---
title: 【golang】 slice
date: 2021-04-08 16:25:24
tags: Golang
categories: Golang
---

使用golang已经快2年了，一直没有好好整理过基础知识，这次也好好整理一下，从最基本的slice开始，也把之前的一些疑惑进行自我解答 :)

<!--more-->

## 数组
``` go
func TestArray(t *testing.T) {
	s := [2]int{1, 2}
	fmt.Printf("type is %T, value is %v\n", s, s)
}

type is [2]int, value is [1 2]

```

数组是不可变的,在go中基本上用的很少

## 切片 Slice
### 切片创建
``` go
func TestSliceNew(t *testing.T) {
	//No.1
	s1 := []int{1, 2, 3}
	fmt.Printf("len is %d, cap is %d , is nil %t\n", len(s1), cap(s1), s1 == nil)

	//No.2
	s2 := make([]int, 0, 10)
	fmt.Printf("len is %d, cap is %d , is nil %t\n", len(s2), cap(s2), s2 == nil)

	//No.3
	var s3 []int
	fmt.Printf("len is %d, cap is %d , is nil %t\n", len(s3), cap(s3), s3 == nil)

	//No.4
	// new 函数返回是指针类型，所以需要使用 * 号来解引用
	s4 := *new([]int)
	fmt.Printf("len is %d, cap is %d , is nil %t\n", len(s3), cap(s3), s4 == nil)
}

=== RUN   TestSliceNew
len is 3, cap is 3 , is nil false
len is 0, cap is 10 , is nil false
len is 0, cap is 0 , is nil true
len is 0, cap is 0 , is nil true
--- PASS: TestSliceNew (0.00s)
PASS

``` 
总共常用的是3中方式，其中第三种，第四种就不要用了，这里有2中类型，1个叫"nil切片",1个叫"empty切片"，官方推荐使用nil切片，也就是`var s []int`这种方式

    The former declares a nil slice value, while the latter is non-nil but zero-length. They are functionally equivalent—their len and cap are both zero—but the nil slice is the preferred style.

那么这两种有什么区别呢？其实很简单，empty切片在创建时，都会使用同一内存地址来标识，`runtime/malloc.go`下边就这么写的

``` go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	if gcphase == _GCmarktermination {
		throw("mallocgc called with gcphase == _GCmarktermination")
	}

	if size == 0 {
		return unsafe.Pointer(&zerobase)
	}
    //xxxx
}


// base address for all 0-byte allocations
var zerobase uintptr
```

### 下标操作
``` go
s[start:end]
start 不能超过s的len，起始位0
end 不能超过cap


func TestIndex(t *testing.T) {
	s := make([]int, 0, 10)
	s = append(s, []int{1, 2, 3, 4, 5}...)
	//获取所有元素
	fmt.Println(s[:])
	//从第一个元素开始（包含第一个），等同于[:]
	fmt.Println(s[1:])

	fmt.Println(s[:cap(s)])
}

[1 2 3 4 5]
[2 3 4 5]
[1 2 3 4 5 0 0 0 0 0]

```

### insert delete copy ...

https://ueokande.github.io/go-slice-tricks/
详见这里了，很全

### append
append是最常用的方式，先来看这段代码

``` go
func TestSlice1(t *testing.T) {
	s := make([]int, 0)
	fmt.Printf("len is %d, cap is %d ", len(s), cap(s))
	hdr := (*reflect.SliceHeader)(unsafe.Pointer(&s))
	fmt.Printf("ptr is %x \n", hdr.Data)

	s = append(s, 1)
	fmt.Printf("len is %d, cap is %d ", len(s), cap(s))
	hdr = (*reflect.SliceHeader)(unsafe.Pointer(&s))
	fmt.Printf("ptr is %x \n", hdr.Data)

	s = append(s, 2)
	fmt.Printf("len is %d, cap is %d ", len(s), cap(s))
	hdr = (*reflect.SliceHeader)(unsafe.Pointer(&s))
	fmt.Printf("ptr is %x \n", hdr.Data)

	s = append(s, 3)
	fmt.Printf("len is %d, cap is %d ", len(s), cap(s))
	hdr = (*reflect.SliceHeader)(unsafe.Pointer(&s))
	fmt.Printf("ptr is %x \n", hdr.Data)

	s = append(s, 4)
	fmt.Printf("len is %d, cap is %d ", len(s), cap(s))
	hdr = (*reflect.SliceHeader)(unsafe.Pointer(&s))
	fmt.Printf("ptr is %x \n", hdr.Data)
}

=== RUN   TestSlice1
len is 0, cap is 0 ptr is c00006ae58 
len is 1, cap is 1 ptr is c0000162c0 
len is 2, cap is 2 ptr is c0000162d0 
len is 3, cap is 4 ptr is c00001e100 
len is 4, cap is 4 ptr is c00001e100 
--- PASS: TestSlice1 (0.00s)
PASS
```

从输出结果看，在append后，前3次的slice指针都变了，说明产生了新的slice，在slice的源码中可以看到如下代码`runtime/slice.go growslice`
``` go
newcap := old.cap
doublecap := newcap + newcap
if cap > doublecap {
    newcap = cap
} else {
    if old.cap < 1024 {
        newcap = doublecap
    } else {
        // Check 0 < newcap to detect overflow
        // and prevent an infinite loop.
        for 0 < newcap && newcap < cap {
            newcap += newcap / 4
        }
        // Set newcap to the requested cap when
        // the newcap calculation overflowed.
        if newcap <= 0 {
            newcap = cap
        }
    }
}
```
可以这么解读，如果cap在1024下，每次slice的膨胀会乘以2，当超过1024后，会变为1.25（实际上不准确，后边有内存补齐的处理，基本维持在1.3左右）

那么不难看出，创建一个空slice后，append操作的cap为`0 -> 1 -> 2 -> 4 -> 8`，也就意味着第1、2、3次append都会产生新的slice，4、5次就会使用同一slice

### slice函数传递
之前这个是我最不清楚的，也是一知半解，直接看一下代码

``` go
func TestSlice2(t *testing.T) {
	s := make([]int, 0, 8)
	s = append(s, []int{1, 2, 3, 4, 5}...)

	fmt.Printf("len is %d, cap is %d \n", len(s), cap(s))
	modify(s)
	fmt.Printf("修改后的值 %v\n", s)
	appendSlice(s)
	fmt.Printf("append后的值 %v\n", s)
}

func modify(s []int) {
	s[0] = 9
}

func appendSlice(s []int) {
	s = append(s, 6)
}


=== RUN   TestSlice2
len is 5, cap is 8 
修改后的值 [9 2 3 4 5]
append后的值 [9 2 3 4 5]
--- PASS: TestSlice2 (0.00s)
PASS
```

在modify函数内，改变了s[0]为9，输出后确实是，变化了，说明在传递时是"引用传递"(这里先用这个词来描述),但appendSlice操作后，并没有发生变化，难道不是"引用传递"吗？为什么会没有append成功？


首先这里先直接说结论，`go中没有引用传递，只有值传递` https://golang.org/ref/spec#Calls

    In a function call, the function value and arguments are evaluated in the usual order. After they are evaluated, the parameters of the call are passed by value to the function and the called function begins execution. The return parameters of the function are passed by value back to the caller when the function returns.

那么在go中有3个数据结构是特殊的，那就是`map、slice、chan`他们的类型叫做`reference types`,他们是"引用类型"，但在函数传递过程中，并不是"引用类型"，这块之前比较迷茫，现在就清晰了

    Map types are reference types, like pointers or slices, and so the value of m above is nil; it doesn't point to an initialized map. A nil map behaves like an empty map when reading, but attempts to write to a nil map will cause a runtime panic; don't do that. To initialize a map, use the built in make function:

https://blog.golang.org/maps

先看下slice的结构定义，这里有2个，具体什么区别暂时还不清楚，但原理基本一样

``` go
// runtime/slice.go

type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}

// reflect/value.go
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}
//
```

基本上都是1个指针，指着数组第一位内存地址

基于上述，修改一下之前的测试代码，就可以清楚的看出来参数传递的端倪

``` go
func TestSlice2(t *testing.T) {
	s := make([]int, 0, 8)
	s = append(s, []int{1, 2, 3, 4, 5}...)

	hdr := (*reflect.SliceHeader)(unsafe.Pointer(&s))
	fmt.Printf("slice A len is %d, cap is %d , point is %p, 头指针为 %d\n", len(s), cap(s), &s, hdr.Data)

	appendSlice(s)

	//获取s的SliceHeader指针
	hdr = (*reflect.SliceHeader)(unsafe.Pointer(&s))
	//将Data转为int类型的数组，长度为8
	newData := *(*[8]int)(unsafe.Pointer(hdr.Data))

	fmt.Printf("输出s的第六位值为 %d\n", newData[5])
}

func appendSlice(s []int) {
	hdr := (*reflect.SliceHeader)(unsafe.Pointer(&s))
	fmt.Printf("slice B len is %d, cap is %d , point is %p, 头指针为 %d\n", len(s), cap(s), &s, hdr.Data)
	s = append(s, 6)
}

=== RUN   TestSlice2
slice A len is 5, cap is 8 , point is 0xc0000ba030, 头指针为 824634409344
slice B len is 5, cap is 8 , point is 0xc0000ba048, 头指针为 824634409344
输出s的第六位值为 6
--- PASS: TestSlice2 (0.00s)
PASS
```
可以看出来，入参后，slice的指针为`0xc00000c060`，与`0xc00000c048`是不同的，说明传递的是值，也就是s的一份拷贝,然后slice下的Data指针却是一致的`824633852544`，说明指向的都是同一份内存地址

在appendSlice函数内进行了append,只是对`slice B`进行了append，并没有影响`slice A`,但是通过反射，将Data指针强转为长度为8的数组后，输出第5位值后发现，其实append是成功的，只是`slice A`并没有更新他的len,其实还可以再用如下方法进行修改
``` go
	hdr.Len = 6
	fmt.Println(s[5])

6
```
原理是一致的


## 总结

1.  slice分为"nil切片"、"empty切片"，建议使用"nil切片"
2.  slice扩容在1024前是2倍，1024后是1.25倍，为了减少copy尽量指定slice长度
3.  slice作为参数传递时，传递的是原slice的一份copy，只是data数据指针为同一数组指针