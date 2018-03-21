# Golang Slice与Map

## Slice
与数组在声明时的区别仅为数组需要指定数目, 而slice不需要. <br>
创建方式:
`var slice []int`  <br>
`slice := array[start:end]` 从原有数组中创建 <br>
`s := []int{0, 1, 2, 3}` <br>

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

#### for range 遍历
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

#### 关于[]byte 与 string
