# 探究 Go 语言中的切片


<!--more-->

## 切片概述

切片 slice 是 Golang 里经典的数据结构，功能与其他语言的数组类似。切片中的元素存放在内存地址连续的区域，使用索引可以快速检索到指定位置的元素。

## 数据结构

```go
type slice struct {
	array unsafe.Pointer // 底层数组的指针，指向起点的地址
	len   int            // 切片的长度
	cap   int            // 切片的容量
}
```

切片结构的定义如上，位于 `runtime/slice.go` 文件中，切片结构包括：地址、长度和容量。

- `array`：指向切片第一个元素的地址，也是内存空间中地址的起点。
- `len`：切片的长度，指的是逻辑上 slice 中存放了多少元素，索引的越界是针对长度而言的。
- `cap`：切片的容量，指的是物理上 slice 分配了用于存放多少个元素的空间大小。

## 生成切片

切片的生成方式主要可以归为三种：

1.  直接生成一个新的切片
2. 从数组生成一个新的切片
3. 从切片生成一个新的切片

### 初始化

通过声明、初始化的操作直接生成新的切片.

- 声明

下面的例子声明了 slice 的类型，但没有进行初始化操作，此时 s 为 一个空指针 `nil`，slice 的长度和容量都为 0，并没有进行内存分配操作：

```go
var s []int
```

- 声明的同时初始化

可以在声明切片的同时进行初始化赋值：

```go
s := []int{2, 3}
```

此时会将切片的长度和容量都设置为 2。

- 基于 make 函数的初始化

```go
s := make([]int, 3, 6)
```

分别指定切片的长度为 3，容量为 6，当切片长度被指定后，对应位置会被分配元素，为相应元素类型的零值。需要注意容量需要永远大于等于长度。

也可以缺省一个参数：

```go
s := make([]int, 3)
```

此时切片的长度和容量均为 3。

切片初始化的源码位于 `runtime/slice.go` 文件中定义的 `makeslice` 方法：

```go
func makeslice(et *_type, len, cap int) unsafe.Pointer {
	// 根据容量和元素类型的大小计算消耗的空间
    mem, overflow := math.MulUintptr(et.Size_, uintptr(cap))
    // 如果出问题直接 panic
    if overflow || mem > maxAlloc || len < 0 || len > cap {
        mem, overflow := math.MulUintptr(et.Size_, uintptr(len))
        if overflow || mem > maxAlloc || len < 0 {
            panicmakeslicelen()
        }
        panicmakeslicecap()
    }
    // 通过 mallocgc 分配内存，初始化切片
    return mallocgc(mem, et, true)
}
```

`makeslice` 方法的核心步骤如下：

- 调用 `runtime/internal/math/math.go` 文件里的 `MulUintprt` 方法，结合切片容量和每个元素的大小，计算出切片需要的内存空间大小，并且返回布尔值表示是否溢出
- 如果出错，则直接 `panic`
- 调用 `runtime/malloc.go` 文件里的 `mallocgc` 方法，为切片分配内存

### 内容截取

slice 可以通过截取操作生成，使用 `s[a:b]` 的格式，其中 a 和 b 代表索引，取值范围是左闭右开，当 a 缺省时默认为 0，当 b 缺省时默认为 `len`。

- 从数组截取生成新的切片

```go
var a = [...]int{1, 2, 3, 4}
var s = a[1:3] // s: [2, 3]
```

- 从切片截取生成新的切片

```go
s := []int{1, 2, 3, 4, 5}
s1 := s[1:]         // s1: [2, 3, 4, 5]
s2 := s[:3]         // 32: [1, 2, 3]
s3 := s[1:len(s)-1] // s3: [2, 3, 4]
```

在进行截取操作时，会创建一个新的切片结构实例，`array`、`len`、`cap` 三个字段可能会发生变化，但是底层使用的是同一块内存空间的数据，本质上相当于进行了一次引用传递。

## 元素追加

通过 `append` 函数来进行元素追加操作：

```go
func TestSlice(t *testing.T) {
	s := []int{2, 3, 4}
	s = append(s, 5) // s: [2, 3, 4, 5]
}
```

`append` 操作会在长度的末尾追加元素，如果追加的内容会导致 `len` 超过 `cap` 的话会引发扩容。

## 切片扩容

当切片进行追加操作（如使用`append`函数添加元素）时，如果追加的元素数量超过了当前切片的容量，就会触发切片的扩容过程。

切片扩容过程的源码位于 `runtime/slice.go` 文件中的 `growslice` 方法和 `nextslicecap` 方法中

`growslice` 方法返回值为新的切片的指针、长度、容量，其入参如下：

- `oldPtr`：指向原切片的指针
- `newLen`：新切片的长度（`newLen = oldLen + num`）
- `oldCap`：原切片的容量
- `num`：添加的元素数量
- `et`：元素类型

```go
func growslice(oldPtr unsafe.Pointer, newLen, oldCap, num int, et *_type) slice {
	oldLen := newLen - num
	// ...
	if newLen < 0 {
		panic(errorString("growslice: len out of range"))
	}

	if et.Size_ == 0 {
		return slice{unsafe.Pointer(&zerobase), newLen, newLen}
	}

	// 计算扩容后数组的容量
	newcap := nextslicecap(newLen, oldCap)

	var overflow bool
	var lenmem, newlenmem, capmem uintptr

	noscan := et.PtrBytes == 0
    switch {
    case et.Size_ == 1:
        lenmem = uintptr(oldLen)
        newlenmem = uintptr(newLen)
        capmem = roundupsize(uintptr(newcap), noscan)
        overflow = uintptr(newcap) > maxAlloc
        newcap = int(capmem)
    case et.Size_ == goarch.PtrSize:
        lenmem = uintptr(oldLen) * goarch.PtrSize
        newlenmem = uintptr(newLen) * goarch.PtrSize
        capmem = roundupsize(uintptr(newcap)*goarch.PtrSize, noscan)
        overflow = uintptr(newcap) > maxAlloc/goarch.PtrSize
        newcap = int(capmem / goarch.PtrSize)
    case isPowerOfTwo(et.Size_):
        var shift uintptr
        if goarch.PtrSize == 8 {
            // Mask shift for better code generation.
            shift = uintptr(sys.TrailingZeros64(uint64(et.Size_))) & 63
        } else {
            shift = uintptr(sys.TrailingZeros32(uint32(et.Size_))) & 31
        }
        lenmem = uintptr(oldLen) << shift
        newlenmem = uintptr(newLen) << shift
        capmem = roundupsize(uintptr(newcap)<<shift, noscan)
        overflow = uintptr(newcap) > (maxAlloc >> shift)
        newcap = int(capmem >> shift)
        capmem = uintptr(newcap) << shift
    default:
        lenmem = uintptr(oldLen) * et.Size_
        newlenmem = uintptr(newLen) * et.Size_
        capmem, overflow = math.MulUintptr(et.Size_, uintptr(newcap))
        capmem = roundupsize(capmem, noscan)
        newcap = int(capmem / et.Size_)
        capmem = uintptr(newcap) * et.Size_
    }

	if overflow || capmem > maxAlloc {
		panic(errorString("growslice: len out of range"))
	}

	// 声明指向新的切片的指针
	var p unsafe.Pointer
    if et.PtrBytes == 0 {
        p = mallocgc(capmem, nil, false)
        memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
    } else {
        p = mallocgc(capmem, et, true)
        if lenmem > 0 && writeBarrier.enabled {
            bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(oldPtr), lenmem-et.Size_+et.PtrBytes, et)
        }
    }
    // 将原切片内容拷贝到扩容后的位置 p
    memmove(p, oldPtr, lenmem)

    return slice{p, newLen, newcap}
}
```

在 `nextslice` 方法中计算了新切片容量的估计值，算法逻辑如下：

- 如果新的长度超过原切片容量的两倍，则直接将新的长度作为新容量
- 如果原容量小于 256，则直接用原容量的两倍作为新容量
- 如果原容量不低于 256，则将原容量按 1.25 的倍率扩容并加上 192，重复该操作直到新容量超过了预期的新长度

```go
// nextslicecap computes the next appropriate slice length.
func nextslicecap(newLen, oldCap int) int {
	// 计算扩容后的新容量
    newcap := oldCap
    // 取原容量的两倍
    doublecap := newcap + newcap
    // 如果新长度大于原容量的两倍，则直接将新长度作为新容量
    if newLen > doublecap {
        return newLen
    }

    const threshold = 256
    // 如果原容量小于 256，则直接将原容量的两倍作为新容量
    // 较小的切片以 2x 的倍率扩容
    if oldCap < threshold {
        return doublecap
    }
    for {
        // 较大的切片以 1.25x 的倍率扩容
        // 新容量 = 原容量 * 1.25 + 192
        newcap += (newcap + 3*threshold) >> 2

		// 当扩容的新容量大于等于预期新长度时退出循环
        if uint(newcap) >= uint(newLen) {
            break
        }
    }
    // 如果出现溢出则将新长度作为新容量
    if newcap <= 0 {
        return newLen
    }
    return newcap
}
```

## 元素删除

Go 语言没有为删除切片元素提供方法，需要手动将删除前后的元素连接起来实现删除功能。

```go
func TestSlice(t *testing.T) {
	s := []int{0, 1, 2, 3, 4}
	// 删除下标为 2 的元素
	s = append(s[:2], s[3:]...)
	// s: [0,1,3,4], len = 4, cap = 5
	t.Logf("s: %v, len: %d, cap %d", s, len(s), cap(s))
}
```

如果需要删除所有元素：

```go
func TestSlice(t *testing.T) {
	s := []int{0, 1, 2, 3, 4}
	// 删除下标为 2 的元素
	s = append(s[:2], s[3:]...)
	// s: [0,1,3,4], len = 4, cap = 5
	t.Logf("s: %v, len: %d, cap %d", s, len(s), cap(s))
}
```

```go
func TestSlice(t *testing.T) {
	s := []int{0, 1, 2, 3, 4}
	// 删除下标为 2 的元素
	s = s[:0]
	// s: [], len = o, cap = 5
	t.Logf("s: %v, len: %d, cap %d", s, len(s), cap(s))
}
```

## 切片拷贝

切片拷贝包括简单拷贝和完整拷贝，简单拷贝得到的只有一个新的 slice struct，但是指向的是同一个内存地址空间，内容截取也属于简单拷贝；完整拷贝会创建一个独立的内存空间，内容与原切片相同，完整拷贝调用 `copy` 函数：

```go
func TestSlice(t *testing.T) {
	s :[]int{1, 2, 3, 4, 5}
	s1 := make([]int, 5)
	copy(s1, s)
	t.Logf("address of s: %p", s)
	t.Logf("address of s1: %p", s1)
}
```

## 切片遍历

通过 `range` 来遍历切片：

```go
func TestSlice(t *testing.T) {
	s :[]int{1, 2, 3, 4, 5}
	for i, v := range s {
		t.Logf("index: %d, value: %d", i, v)
	}
}
```

## 参考资料

[go/src/runtime/slice.go at master · golang/go (github.com)](https://github.com/golang/go/blob/master/src/runtime/slice.go)

[你真的了解go语言中的切片吗？ (qq.com)](https://mp.weixin.qq.com/s/uNajVcWr4mZpof1eNemfmQ)

