# Day 13 - Go編程模式：錯誤處理

錯誤處理一直是編程必須要面對的問題。錯誤處理如果做得好的話，代碼的穩定性會很好。

## C 語言的錯誤處理

通過函數的返回值標識是否有錯，然後通過全局的 `errno` 變量加一個 `errstr` 的數組來告訴用戶為什麼出錯。

如：`read()`、 `write()`、`open()` 這些函數的**返回值其實是返回有業務邏輯的值**，也就是說，這些函數的返回值有兩種語義：

- 一種是**成功的值**，如 `open()` 返回的文件句柄指針 `FILE* `
- 另一種是**錯誤 NULL**。這會導致調用者並不知道是什麼原因出錯了，需要去檢查 `errno` 以獲得出錯的原因，從而正確地處理錯誤

大多數情況，上面說的處理方式即可滿足，但也是有特殊例外，如：
```c
// 字符串轉整數
int atoi(const char *str)
```
但是如果無法轉整數時候，發生錯誤呢？

- 出錯返回任何數值都不合理，會和正常結果混淆
- 如果返回 0 ，又會跟 0 數值混淆

在 C99 的規格說明書中可以看到這樣的描述：

> 7.20.1The functions atof, atoi, atol, and atoll need not affect the value of the integer expression errno on an error. If the value of the result cannot be represented, the behavior is undeﬁned.

也就表示，像 `atoi()`、`atof()`、  `atol()` 或 `atoll()` 這樣的函數，是不會設置 `errno` 的，而且，如果結果無法計算的話，行為是 `undefined`。

後來 libc 新增了一個函數 `strtol()`，其函數會在出錯時候，設置一個全局變量 `errno`

```c
long val = strtol(in_str, &endptr, 10);  //10的意思是10進制

//如果無法轉換
if (endptr == str) {
    fprintf(stderr, "No digits were found\n");
    exit(EXIT_FAILURE);
}

//如果整型溢出了
if ((errno == ERANGE && (val == LONG_MAX || val == LONG_MIN)) {
    fprintf(stderr, "ERROR: number out of range for LONG\n");
    exit(EXIT_FAILURE);
 }

//如果是其它錯誤
if (errno != 0 && val == 0) {
    perror("strtol");
    exit(EXIT_FAILURE);
}
```

但是這種用**返回值 + errno** 的錯誤檢查方式會有一些問題：
- 程序員一不小心就會忘記檢查返回值，從而造成代碼的 Bug
- 函數接口非常不純潔，正常值和錯誤值混淆在一起，導致語義有問題

## Java 的錯誤處理

使用 `try-catch-finally` 通過使用異常的方式來處理錯誤。

使用 拋異常 和 抓異常 的方式可以讓我們的代碼有這樣一些好處：
- **函數接口在 input（參數）和 output（返回值）以及錯誤處理的語義是比較清楚的**
- 正常邏輯的代碼可以跟**錯誤處理和資源清理的代碼分開**，提高了代碼的可讀性
- 異常不能被忽略（如果要忽略也需要 catch 住，這是顯式忽略）
- **在面向對象的語言中（如 Java），異常是個對象，所以，可以實現多態式的 catch**。

與狀態返回碼相比，異常捕捉有一個顯著的好處，那就是函數可以嵌套調用，或是鏈式調用：
```java
int x = add(a, div(b,c));
Pizza p = PizzaBuilder().SetSize(sz).SetPrice(p)...;
```

## Go 語言的錯誤處理

**Go 語言的函數支持多返回值**。因此，可以在返回接口把業務語義（業務返回值）和控制語義（出錯返回值）區分開 。

很多函數都會返回（result，error），整理如下：

- 參數上基本上就是入參，而返回接口把結果和錯誤分離，這樣使得函數的接口語義清晰
- Go 語言中的錯誤參數如果要忽略，需要顯式地忽略，用 `_` 這樣的變量來忽略；
- 因為返回的 **error 是個接口**（其中只有一個方法 Error()，返回一個 string ），所以可以**擴展自定義的錯誤處理**

```go
// bulitin.go
type error interface {
	Error() string
}
```

如果返回多類型錯誤，可以採用`類型斷言`處理
```go
if err != nil {
  switch err.(type) {
    case *json.SyntaxError:
      ...
    case *ZeroDivisionError:
      ...
    case *NullPointerError:
      ...
    default:
      ...
  }
}
```
> Go 語言的錯誤處理的方式，本質上是返回值檢查，但是它也兼顧了異常的一些好處：對錯誤的擴展。

## 資源清理

出錯後是需要做資源清理的。
- C 語言：使用的是 goto fail; 的方式到一個集中的地方進行清理
- C++ 語言：一般來說使用 [RAII 模式](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization)，通過面向對象的代理模式，把需要清理的資源交給一個代理類，然後再解析函數來解決
- Java 語言：可以在 `finally` 語句塊里進行清理
- Go 語言：使用 `defer` 關鍵詞進行清理

## Error Check Hell

Go 語言的 **if err !=nil** 的代碼問題，直接藉由例子說明
```go
func parse(r io.Reader) (*Point, error) {

    var p Point

    if err := binary.Read(r, binary.BigEndian, &p.Longitude); err != nil {
        return nil, err
    }
    if err := binary.Read(r, binary.BigEndian, &p.Latitude); err != nil {
        return nil, err
    }
    if err := binary.Read(r, binary.BigEndian, &p.Distance); err != nil {
        return nil, err
    }
    if err := binary.Read(r, binary.BigEndian, &p.ElevationGain); err != nil {
        return nil, err
    }
    if err := binary.Read(r, binary.BigEndian, &p.ElevationLoss); err != nil {
        return nil, err
    }
}
```

可以使用**函數式編程**修改上述的代碼，如：
```go
func parse(r io.Reader) (*Point, error) {
    var p Point
    var err error
    read := func(data interface{}) {
        if err != nil {
            return
        }
        err = binary.Read(r, binary.BigEndian, data)
    }

    read(&p.Longitude)
    read(&p.Latitude)
    read(&p.Distance)
    read(&p.ElevationGain)
    read(&p.ElevationLoss)

    if err != nil {
        return &p, err
    }
    return &p, nil
}
```
> 覺得 error 變量和一個內部函數，會有些繁瑣

藉由 Go 語言的 **bufio.Scanner()** 中似乎可以學習錯誤的封裝用法

```go
scanner := bufio.NewScanner(input)

for scanner.Scan() {
    token := scanner.Text()
    // process token
}

if err := scanner.Err(); err != nil {
    // process the error
}
```

> scanner 在操作底層的 I/O 的時候，那個 for-loop 中沒有任何的 if err !=nil 的情況，退出循環後有一個 scanner.Err() 的檢查，看來使用了結構體的方式。

接著，改造最一開始的例子，先定義結構體：
```go
type Reader struct {
    r   io.Reader
    err error
}

func (r *Reader) read(data interface{}) {
    if r.err == nil {
        r.err = binary.Read(r.r, binary.BigEndian, data)
    }
}
```
藉由 Reader 結構體，修改為：
```go
func parse(input io.Reader) (*Point, error) {
    var p Point
    r := Reader{r: input}

    r.read(&p.Longitude)
    r.read(&p.Latitude)
    r.read(&p.Distance)
    r.read(&p.ElevationGain)
    r.read(&p.ElevationLoss)

    if r.err != nil {
        return nil, r.err
    }

    return &p, nil
}
```
> 這種使用場景是有局限的，也就**只能在對於同一個業務對象的不斷操作下可以簡化錯誤處理**，如果是多個業務對象，還是得需要各種 if err != nil 的方式。

另一個例子，流式接口[（fluent interface）](https://martinfowler.com/bliki/FluentInterface.html)
```go

package main

import (
  "bytes"
  "encoding/binary"
  "fmt"
)

// 長度不夠，少一個Weight
var b = []byte {0x48, 0x61, 0x6f, 0x20, 0x43, 0x68, 0x65, 0x6e, 0x00, 0x00, 0x2c} 
var r = bytes.NewReader(b)

type Person struct {
  Name [10]byte
  Age uint8
  Weight uint8
  err error
}
func (p *Person) read(data interface{}) {
  if p.err == nil {
    p.err = binary.Read(r, binary.BigEndian, data)
  }
}

func (p *Person) ReadName() *Person {
  p.read(&p.Name) 
  return p
}
func (p *Person) ReadAge() *Person {
  p.read(&p.Age) 
  return p
}
func (p *Person) ReadWeight() *Person {
  p.read(&p.Weight) 
  return p
}
func (p *Person) Print() *Person {
  if p.err == nil {
    fmt.Printf("Name=%s, Age=%d, Weight=%d\n",p.Name, p.Age, p.Weight)
  }
  return p
}

func main() {   
  p := Person{}
  p.ReadName().ReadAge().ReadWeight().Print()
  fmt.Println(p.err)  // EOF 錯誤
}
```

## 包裝錯誤

需要把執行的上下文，一起返回給上層，而不是單純返回錯誤值。

一般用法，`fmt.Errorf()` 封裝錯誤
```go
if err != nil {
   return fmt.Errorf("something failed: %v", err)
}
```
更為普遍的做法是**將錯誤包裝在另一個錯誤中，同時保留原始內容**：

```go

type authorizationError struct {
    operation string
    err error   // original error
}

func (e *authorizationError) Error() string {
    return fmt.Sprintf("authorization failed during %s: %v", e.operation, e.err)
}
```

更好的方式是**通過一種標準的訪問方法**。如同標準庫的錯誤處理用法。

causer 接口中實現 Cause() 方法來暴露原始錯誤，以供進一步檢查：
```go
type causer interface {
    Cause() error
}

func (e *authorizationError) Cause() error {
    return e.err
}
```
```go

import "github.com/pkg/errors"

//错误包装
if err != nil {
    return errors.Wrap(err, "read failed")
}

// Cause接口
switch err := errors.Cause(err).(type) {
case *MyError:
    // handle specifically
default:
    // unknown error
}
```

> 補充：
> 在標準庫中，Go提供了構造錯誤值的兩種基本方法—— `errors.New` 和`fmt.Errorf`
> Go 1.13 版本之前，兩種方法返回的是同一個實現 error 接口的類型實例

```go
err := errors.New("your first demo error")
errWithCtx = fmt.Errorf("index %d is out of bounds", i)
wrapErr = fmt.Errorf("wrap error: %w", err) // 僅Go 1.13及後續版本可用
---
// $GOROOT/src/errors/errors.go
type errorString struct {
    s string
}

func (e *errorString) Error() string {
    return e.s
}
```

> Go 1.13及後續版本，當我們在格式化字符串中使用 `%w` 時，fmt.Errorf 返回的錯誤值的底層類型為 `fmt.wrapError`

```go
// $GOROOT/src/fmt/errors.go (Go 1.13及後續版本)
type wrapError struct {
    msg string
    err error
} 
func (e *wrapError) Error() string {
    return e.msg
} 
func (e *wrapError) Unwrap() error {
    return e.err
}

```

> 與 `errorString` 相比，`wrapError` 多實現了 `Unwrap` 方法，這使得被 `wrapError` 類型包裝的錯誤值在包裝錯誤鏈中被檢視（inspect）

```go
var ErrFoo = errors.New("the underlying error")

err := fmt.Errorf("wrap err: %w", ErrFoo)
errors.Is(err, ErrFoo) // true (僅適用於 Go 1.13 及後續版本)
```


> error 接口是錯誤值提供者與錯誤值檢視者之間的契約
> error 接口的實現者負責提供錯誤上下文供負責錯誤處理的代碼使用
> 這種錯誤上下文與 error 接口類型的分離體現了Go設計哲學中的正交理念



## 參考文章

- [Golang Error Handling lesson by Rob Pike](https://jxck.hatenablog.com/entry/golang-error-handling-lesson-by-rob-pike)
- [Errors are values](https://go.dev/blog/errors-are-values)


此文章為3月Day13學習筆記，內容來源於極客時間[《左耳聽風》](https://time.geekbang.org/column/article/332602)