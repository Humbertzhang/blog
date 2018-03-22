# Golang Slice & Map特性与底层实现

## Table of contents
#### [slice](#slice) <br>
[底层结构](#底层结构) <br>
[append函数](#append函数的运用)  <br>
[for-range遍历](#for-range遍历) <br>
[关于byte与string](#关于byte与string) <br>

#### [map](#map) <br>
[声明](#声明) <br>
[遍历](#遍历) <br>
[判空](#判空) <br>
[使用slice作为map的key](#使用slice作为map的key) <br>
[底层结构](#底层结构) <br>
[扩容](#扩容) <br>
[元素删除机制](#元素删除机制) <br>

## Slice
与数组在声明时的区别仅为数组需要指定数目, 而slice不需要. <br>
创建方式:
`var slice []int`  <br>
`slice := array[start:end]` 从原有数组中创建 <br>
`s := []int{0, 1, 2, 3}` <br>


#### for range遍历
需要index:
```Go
for i, v := range slice {
  fmt.Println("%d %d\n", i, v)
}
```
不需要index:
```Go
for _, v := range slice {
  fmt.Println(v)
}
```

#### 底层结构
Slice在/runtime/slice.go中的声明如下: <br>
```Go
type slice struct {
     array unsafe.Pointer
     len   int
     cap   int
}
```
可以看到，Slice由数组的指针，len，cap组成.其中数组指针为slice初始化时初始化的数组，len为slice的长度，cap为slice的容量.<br>
slice不一定占用整个数组，而可能是底层那个数组的一部分，slice的len即为slice占用该数组的长度，而cap为从slice开始到数组结束的长度 <br>
因为slice记录了len和cap, 因此使用len()内置函数可以获取slice的len，cap()内置函数可以获取slice的容量。<br>
和数组要区分的是，slice不可以直接用 == 来比较，除了[]byte可以用bytes.Equal来比较之外，其他类型slice只能自己写比较函数 <br>



#### append函数的运用
slice的扩容方式与cpp的vector类似,当append的元素的数目超过了slice的容量(cap)时，会为其重新分配一块空间，为了保证追加一个元素所消耗的时间整体上来看为固定的，并尽量减少内存重新分配的次数，故每次都会比原来多分配一倍的空间.我们无法保证append之后的slice与原来的slice在底层上指向的数组是同一个，故一般会运用这样的方式来运用append:`slice = append(slice, s)` <br>

#### 关于byte与string
string在`src/runtime/string.go`的声明如下:
```Go
type stringStruct struct {
    str unsafe.Pointer
    len int
}
```
可以看到，同slice一样，string底层也是一个数组。
string 指向的数组是只读的(`个人猜测是golang中string没有实现像slice一样直接根据向index来更改字符的机制，又在重新赋值时新开一个字符串而不是修改之前的`)，所以string字符串是不可更改的.

[]byte和string是可以相互转换的，但是在相互转换时默认的实现不会使用原有的空间，而都是重新分配内存。因为string是不可以改变的，如果将string转换为[]byte之后且不重新分配，那么此时他们就共用内存，一旦slice修改了底层字符，会导致程序Down掉.

<b> 实际运用中应当如何取舍string与[]byte? </b>

+ string可以直接比较，而[]byte不可以，所以[]byte不可以当map的key值。
+ 因为无法修改string中的某个字符，需要粒度小到操作一个字符时，用[]byte。
+ string值不可为nil，所以如果你想要通过返回nil表达额外的含义，就用[]byte。
+ []byte切片这么灵活，想要用切片的特性就用[]byte。
+ 需要大量字符串处理的时候用[]byte，性能好很多。


## Map
Map底层实现为哈希表。

#### 声明
```
m := make(map[keytype]valuetype)
m[key1] = value1
m[key2] = value2
```
或
```
m := map[string]int {
     "alice":1,
     "belly":2.
} 
```

#### 遍历
```
for key, value := range m {
     fmt.Println(key, ":", value)
}
```


#### 判空
map在获取元素时若其还未曾加入到map中，不会报错而是会返回0（或其他value type的零值）,因此就需要另外的机制来区分是本来值为0还是未曾加入到map中<br>
因此map在获取元素时还会在最后多返回一个参数来标志是否在map中.
```
fakekey, ok := m["fakekey"]
if !ok {
     // handle func
     // fakekey 不在map中
}
或者直接这样写
if fakekey, ok := m["fakekey"]; !ok {
     // handle func
}
```

#### 使用slice作为map的key
Map的Key只能为不可变的元素，因此slice不能直接作为map的key。
常采取的手段是使用一个函数将slice来映射成字符串，并保证slice不重复字符串也就不重复来使用.
```
// 设转换函数为strs()
m := make(map[string]int)
m[strs(slice1)] = 1
m[strs(slice2)] = 2
```


#### 底层结构
HashMap
```
type hmap struct {
	count     int // # 元素个数
	flags     uint8
	B         uint8  // 包含2^B个bucket
	noverflow uint16 // 溢出的bucket的个数
	hash0     uint32 // hash种子

	buckets    unsafe.Pointer // buckets的数组指针
	oldbuckets unsafe.Pointer // 结构扩容的时候用于复制的buckets数组
	nevacuate  uintptr        // 搬迁进度（已经搬迁的buckets数量）

	extra *mapextra
}
```

#### 扩容
随着元素的增加，在一个bucket链中寻找特定的key会变得效率低下，所以在插入的元素个数/bucket个数达到某个阈值（当前设置为6.5）时，map会进行扩容.<br>
扩容的具体方式是，首先创建bucket数组，长度为原长度的两倍然后替换原有的bucket，原有的bucket被移动到oldbucket指针下。但是此时不会直接将元素迁移到新的bucket中,而是当访问到该bucket时，才将oldbucket下的一些元素rehash到新的bucket中。随着访问的进行，所有oldbucket才会被逐渐移动到bucket中。<br>
但是如果当第二次需要扩容时，上次的迁移还没结束，则会暂时不会扩容，直到上次迁移结束后再次扩容。<br>

#### 元素删除机制
在map中删除一个元素后，并不会直接释放掉他的内存，而是仅仅将对应的`tophash[i]`设置为empty。若要释放内存只有指针无引用后被系统gc.<br>
也因此对于map的迭代是安全的，因为并不是删除掉元素后rehash,而是仅仅将元素设为不可访问，因此元素顺序不会收到影响.<br>


***
#### Reference: <br>
https://sheepbao.github.io/post/golang_byte_slice_and_string <br>
https://blog.yiz96.com/golang-map/ <br>




