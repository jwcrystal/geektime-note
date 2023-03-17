# Day 17 - Go編程模式：Go Generation

Go 語言的代碼生成主要還是用來解決編程泛型的問題。

**泛型編程是靜態語言中非常非常重要的特徵**，如果沒有泛型，我們就很難做到多態，也很難完成抽象，這就會導致我們的代碼冗餘量很大。

## Go 的類型檢查

Go 語言的類型檢查有兩種技術
- Type Assert (類型斷言)
- Reflection (反射)

### Type Assert 

對某個變量進行 `.(type)` 的轉型操作，它會返回兩個值，分別是 **variable** 和 **error**

```go
//Container is a generic container, accepting anything.
type Container []interface{}

//Put adds an element to the container.
func (c *Container) Put(elem interface{}) {
    *c = append(*c, elem)
}
//Get gets an element from the container.
func (c *Container) Get() interface{} {
    elem := (*c)[0]
    *c = (*c)[1:]
    return elem
}
---
// 因為類型是 interface{}, 所以可以有下面的用法
intContainer := &Container{}
intContainer.Put(7)
intContainer.Put(42)
```

Type Assert 的例子
- 類型斷言為 int，不然失敗
```go
// assert that the actual type is int
elem, ok := intContainer.Get().(int)
if !ok {
    fmt.Println("Unable to read an int from intContainer")
}

fmt.Printf("assertExample: %d (%T)\n", elem, elem)
```

### Reflection

以延續上面例子，修改成如下：
```go
type Container struct {
    s reflect.Value
}
func NewContainer(t reflect.Type, size int) *Container {
    if size <=0  { size=64 }
    return &Container{
        s: reflect.MakeSlice(reflect.SliceOf(t), 0, size), 
    }
}
func (c *Container) Put(val interface{})  error {
    if reflect.ValueOf(val).Type() != c.s.Type().Elem() {
        return fmt.Errorf(“Put: cannot put a %T into a slice of %s", 
            val, c.s.Type().Elem()))
    }
    c.s = reflect.Append(c.s, reflect.ValueOf(val))
    return nil
}
func (c *Container) Get(refval interface{}) error {
    if reflect.ValueOf(refval).Kind() != reflect.Ptr ||
        reflect.ValueOf(refval).Elem().Type() != c.s.Type().Elem() {
        return fmt.Errorf("Get: needs *%s but got %T", c.s.Type().Elem(), refval)
    }
    reflect.ValueOf(refval).Elem().Set( c.s.Index(0) )
    c.s = c.s.Slice(1, c.s.Len())
    return nil
}
```
上面代碼完全使用 Reflection 的玩法：
- 在 NewContainer()時，會根據參數的類型初始化一個 Slice
- 在 Put()時，會檢查 val 是否和 Slice 的類型一致
- 在 Get()時，我們需要用一個入參的方式，因為我們沒有辦法返回 reflect.Value 或 interface{}，不然還要做 Type Assert
- 不過有類型檢查，所以，必然會有檢查不對的時候，因此，需要返回 error


使用上面反射例子，如下：
```go
f1 := 3.1415926
f2 := 1.41421356237

c := NewMyContainer(reflect.TypeOf(f1), 16)

if err := c.Put(f1); err != nil {
  panic(err)
}
if err := c.Put(f2); err != nil {
  panic(err)
}

g := 0.0

if err := c.Get(&g); err != nil {
  panic(err)
}
fmt.Printf("%v (%T)\n", g, g) //3.1415926 (float64)
fmt.Println(c.s.Index(0)) //1.4142135623
```

## Template 

對於泛型編程最牛的語言 C++ 來說，這類問題都是使用 `Template` 解決的。
```go
//用<class T>來描述泛型
template <class T> 
T GetMax (T a, T b)  { 
    T result; 
    result = (a>b)? a : b; 
    return (result); 
} 
---
int i=5, j=6, k; 
//生成int類型的函數
k=GetMax<int>(i,j);
 
long l=10, m=5, n; 
//生成long類型的函數
n=GetMax<long>(l,m); s
```

C++ 的編譯器會在**編譯時**分析代碼，**根據不同的變量類型來自動化生成相關類型的函數或類**，在 C++ 里，叫模板的具體化。

## Go Generator

要玩 Go 的代碼生成，需要三個東西：
- 一個函數模板，在裡面設置好相應的佔位符
- 一個腳本，用於按規則來替換文本並生成新的代碼
- 一行注釋代碼

### 函數模版

把之前的示例改成模板，取名为 **container.tmp.go** 放在 `./template/`下：
```go
package PACKAGE_NAME
type GENERIC_NAMEContainer struct {
    s []GENERIC_TYPE
}
func NewGENERIC_NAMEContainer() *GENERIC_NAMEContainer {
    return &GENERIC_NAMEContainer{s: []GENERIC_TYPE{}}
}
func (c *GENERIC_NAMEContainer) Put(val GENERIC_TYPE) {
    c.s = append(c.s, val)
}
func (c *GENERIC_NAMEContainer) Get() GENERIC_TYPE {
    r := c.s[0]
    c.s = c.s[1:]
    return r
}
```

### 函數生成腳本
> Ref:
> - [sed 简明教程](https://coolshell.cn/articles/9104.html)

這裡需要 4 個參數：
- 模板源文件
- 包名
- 實際需要具體化的類型
- 用於構造目標文件名的後綴。

```sh
#!/bin/bash

set -e

SRC_FILE=${1}
PACKAGE=${2}
TYPE=${3}
DES=${4}
#uppcase the first char
PREFIX="$(tr '[:lower:]' '[:upper:]' <<< ${TYPE:0:1})${TYPE:1}"

DES_FILE=$(echo ${TYPE}| tr '[:upper:]' '[:lower:]')_${DES}.go

sed 's/PACKAGE_NAME/'"${PACKAGE}"'/g' ${SRC_FILE} | \
    sed 's/GENERIC_TYPE/'"${TYPE}"'/g' | \
    sed 's/GENERIC_NAME/'"${PREFIX}"'/g' > ${DES_FILE}
```

### 生成代碼

此時，需要在 go 代碼打上一個特殊注釋 `go:generate`
- 下面例子，在工程目錄上執行 `go generate` 命令，就會生成兩份代碼
```go
//go:generate ./gen.sh ./template/container.tmp.go gen uint32 container
func generateUint32Example() {
    var u uint32 = 42
    c := NewUint32Container()
    c.Put(u)
    v := c.Get()
    fmt.Printf("generateExample: %d (%T)\n", v, v)
}

//go:generate ./gen.sh ./template/container.tmp.go gen string container
func generateStringExample() {
    var s string = "Hello"
    c := NewStringContainer()
    c.Put(s)
    v := c.Get()
    fmt.Printf("generateExample: %s (%T)\n", v, v)
}
```

生成代碼，示意如下：
- 生成 uint32_container.go
```go
package gen

type Uint32Container struct {
    s []uint32
}
func NewUint32Container() *Uint32Container {
    return &Uint32Container{s: []uint32{}}
}
func (c *Uint32Container) Put(val uint32) {
    c.s = append(c.s, val)
}
func (c *Uint32Container) Get() uint32 {
    r := c.s[0]
    c.s = c.s[1:]
    return r
}
```
- 生成 string_container.go
```go
package gen

type StringContainer struct {
    s []string
}
func NewStringContainer() *StringContainer {
    return &StringContainer{s: []string{}}
}
func (c *StringContainer) Put(val string) {
    c.s = append(c.s, val)
}
func (c *StringContainer) Get() string {
    r := c.s[0]
    c.s = c.s[1:]
    return r
}
```

## 新版 Filter

Fitler 的模板文件 filter.tmp.go，如下：
```go
package PACKAGE_NAME

type GENERIC_NAMEList []GENERIC_TYPE

type GENERIC_NAMEToBool func(*GENERIC_TYPE) bool

func (al GENERIC_NAMEList) Filter(f GENERIC_NAMEToBool) GENERIC_NAMEList {
    var ret GENERIC_NAMEList
    for _, a := range al {
        if f(&a) {
            ret = append(ret, a)
        }
    }
    return ret
}
```

實際操作，我們可以在需要地方，加上相關的 Go Generate 注釋：
```go
type Employee struct {
  Name     string
  Age      int
  Vacation int
  Salary   int
}

//go:generate ./gen.sh ./template/filter.tmp.go gen Employee filter
func filterEmployeeExample() {

  var list = EmployeeList{
    {"Hao", 44, 0, 8000},
    {"Bob", 34, 10, 5000},
    {"Alice", 23, 5, 9000},
    {"Jack", 26, 0, 4000},
    {"Tom", 48, 9, 7500},
  }

  var filter EmployeeList
  filter = list.Filter(func(e *Employee) bool {
    return e.Age > 40
  })

  fmt.Println("----- Employee.Age > 40 ------")
  for _, e := range filter {
    fmt.Println(e)
  }

  filter = list.Filter(func(e *Employee) bool {
    return e.Salary <= 5000
  })

  fmt.Println("----- Employee.Salary <= 5000 ------")
  for _, e := range filter {
    fmt.Println(e)
  }
}
```

## 第三方工具列表

可以直接使用第三方已經寫好的工具，如下：

- [Genny](https://github.com/cheekybits/genny)
- [Generic](https://github.com/taylorchu/generic)
- [GenGen](https://github.com/joeshaw/gengen)
- [Gen](https://github.com/clipperhouse/gen)

此文章為3月Day17學習筆記，內容來源於極客時間[《左耳聽風》](https://time.geekbang.org/column/article/332607)