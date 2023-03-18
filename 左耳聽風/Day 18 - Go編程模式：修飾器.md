# Day 18 - Go編程模式：修飾器
> Ref：
> - [函數式編程](https://coolshell.cn/articles/10822.html)
> - [The Laws of Reflection](https://go.dev/blog/laws-of-reflection)

通過詳細介紹從過程式編程的思維方式過渡到函數式編程的思維方式，帶動更多的人玩函數式編程。

Go 語言是強類型的靜態無虛擬機的語言，所以沒法像 Java 和 Python 那樣寫出優雅的修飾器的代碼。

## 簡單示例

```go
package main

import "fmt"

func decorator(f func(s string)) func(s string) {

    return func(s string) {
        fmt.Println("Started")
        f(s)
        fmt.Println("Done")
    }
}

func Hello(s string) {
    fmt.Println(s)
}

func main() {
        decorator(Hello)("Hello, World!")
}
```

上述例子，跟 python 的用法很雷同，只是 Go 沒支持像 python 一樣的語法糖 @decorator。如果只是想代碼更加好閱讀，可以如下：
```go
hello := decorator(Hello)
hello("Hello")
```

另一個時間計算的例子

```go
package main

import (
  "fmt"
  "reflect"
  "runtime"
  "time"
)

type SumFunc func(int64, int64) int64

func getFunctionName(i interface{}) string {
  return runtime.FuncForPC(reflect.ValueOf(i).Pointer()).Name()
}

func timedSumFunc(f SumFunc) SumFunc {
  return func(start, end int64) int64 {

    defer func(t time.Time) {
      fmt.Printf("--- Time Elapsed (%s): %v ---\n", 
          getFunctionName(f), time.Since(t))
    }(time.Now())

    return f(start, end)
  }
}

func Sum1(start, end int64) int64 {
  var sum int64
  sum = 0
  if start > end {
    start, end = end, start
  }
  for i := start; i <= end; i++ {
    sum += i
  }
  return sum
}

func Sum2(start, end int64) int64 {
  if start > end {
    start, end = end, start
  }
  return (end - start + 1) * (end + start) / 2
}

func main() {

  sum1 := timedSumFunc(Sum1)
  sum2 := timedSumFunc(Sum2)

  fmt.Printf("%d, %d\n", sum1(-10000, 10000000), sum2(-10000, 10000000))
}
---
// output
$ go run time.sum.go
--- Time Elapsed (main.Sum1): 3.557469ms ---
--- Time Elapsed (main.Sum2): 291ns ---
49999954995000, 49999954995000
```
上述代碼，需要留意的是：
- 代碼中使用了 Go 語言的反射機制來獲取函數名
- 修飾器函數是 **timedSumFunc()**

## HTTP 示例

一個簡單的 HTTP Server 代碼
```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "strings"
)

func WithServerHeader(h http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        log.Println("--->WithServerHeader()")
        w.Header().Set("Server", "HelloServer v0.0.1")
        h(w, r)
    }
}

func hello(w http.ResponseWriter, r *http.Request) {
    log.Printf("Recieved Request %s from %s\n", r.URL.Path, r.RemoteAddr)
    fmt.Fprintf(w, "Hello, World! "+r.URL.Path)
}

func main() {
    http.HandleFunc("/v1/hello", WithServerHeader(hello))
    err := http.ListenAndServe(":8080", nil)
    if err != nil {
        log.Fatal("ListenAndServe: ", err)
    }
}
```
這段代碼中使用到了修飾器模式：
- **WithServerHeader() 函數就是一個 Decorator**，它會傳入一個 http.HandlerFunc，然後返回一個改寫的版本

因此，其他相對參數也可以依照同樣邏輯寫出：
- 有寫 HTTP 響應頭的，有寫認證 Cookie 的，有檢查認證 Cookie 的，有打日誌的
```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "strings"
)

func WithServerHeader(h http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        log.Println("--->WithServerHeader()")
        w.Header().Set("Server", "HelloServer v0.0.1")
        h(w, r)
    }
}

func WithAuthCookie(h http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        log.Println("--->WithAuthCookie()")
        cookie := &http.Cookie{Name: "Auth", Value: "Pass", Path: "/"}
        http.SetCookie(w, cookie)
        h(w, r)
    }
}

func WithBasicAuth(h http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        log.Println("--->WithBasicAuth()")
        cookie, err := r.Cookie("Auth")
        if err != nil || cookie.Value != "Pass" {
            w.WriteHeader(http.StatusForbidden)
            return
        }
        h(w, r)
    }
}

func WithDebugLog(h http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        log.Println("--->WithDebugLog")
        r.ParseForm()
        log.Println(r.Form)
        log.Println("path", r.URL.Path)
        log.Println("scheme", r.URL.Scheme)
        log.Println(r.Form["url_long"])
        for k, v := range r.Form {
            log.Println("key:", k)
            log.Println("val:", strings.Join(v, ""))
        }
        h(w, r)
    }
}
func hello(w http.ResponseWriter, r *http.Request) {
    log.Printf("Recieved Request %s from %s\n", r.URL.Path, r.RemoteAddr)
    fmt.Fprintf(w, "Hello, World! "+r.URL.Path)
}

func main() {
    http.HandleFunc("/v1/hello", WithServerHeader(WithAuthCookie(hello)))
    http.HandleFunc("/v2/hello", WithServerHeader(WithBasicAuth(hello)))
    http.HandleFunc("/v3/hello", WithServerHeader(WithBasicAuth(WithDebugLog(hello))))
    err := http.ListenAndServe(":8080", nil)
    if err != nil {
        log.Fatal("ListenAndServe: ", err)
    }
}
```

## 多個修飾器的 Pipeline

修飾器模式在使用上，需要對函數一層層地套起來，不是很好閱讀，如果需要修飾器比較多的話，代碼就會比較醜。

可以重構，如下：
- 先寫一個工具函數，用來遍歷並調用各個修飾器
```go
type HttpHandlerDecorator func(http.HandlerFunc) http.HandlerFunc

func Handler(h http.HandlerFunc, decors ...HttpHandlerDecorator) http.HandlerFunc {
    for i := range decors {
        d := decors[len(decors)-1-i] // iterate in reverse
        h = d(h)
    }
    return h
}
```
操作行為就回變成如下，而 Pipeline 功能也相對出來了。
```go
http.HandleFunc("/v4/hello", Handler(hello,
                WithServerHeader, WithBasicAuth, WithDebugLog))
```

## 泛型的修飾器

用 Reflection 機制寫的一個比較通用的修飾器（為了便於閱讀，刪除了出錯判斷代碼）。
```go
func Decorator(decoPtr, fn interface{}) (err error) {
    var decoratedFunc, targetFunc reflect.Value

    decoratedFunc = reflect.ValueOf(decoPtr).Elem()
    targetFunc = reflect.ValueOf(fn)

    v := reflect.MakeFunc(targetFunc.Type(),
            func(in []reflect.Value) (out []reflect.Value) {
                fmt.Println("before")
                out = targetFunc.Call(in)
                fmt.Println("after")
                return
            })

    decoratedFunc.Set(v)
    return
}
```
上述例子，要表示的是，此函數需要兩個參數：
1. 出參 decoPtr ，就是完成修飾後的函數
2. 入參 fn ，就是需要修飾的函數。


為了驗證，需要假設有兩個需要修飾的函數
```go
func foo(a, b, c int) int {
    fmt.Printf("%d, %d, %d \n", a, b, c)
    return a + b + c
}

func bar(a, b string) string {
    fmt.Printf("%s, %s \n", a, b)
    return a + b
}
```

修改前面例子，如下：
```go
type MyFoo func(int, int, int) int
var myfoo MyFoo
Decorator(&myfoo, foo)
myfoo(1, 2, 3)
---
// 不想聲明函數簽名，就如下
mybar := bar
Decorator(&mybar, bar)
mybar("hello,", "world!")
```


此文章為3月Day18學習筆記，內容來源於極客時間[《左耳聽風》](https://time.geekbang.org/column/article/332608)