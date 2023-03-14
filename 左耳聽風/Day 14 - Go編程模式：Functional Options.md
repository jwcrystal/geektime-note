# Day 14 - Go編程模式：Functional Options

這是一個函數式編程的應用案例，編程技巧也很好，是 Go 語言中最流行的一種編程模式。

## 解決配置選項問題

在編程中，經常需要對一個對象（或是業務實體）進行相關的配置。

```go
type Server struct {
    Addr     string
    Port     int
    Protocol string
    Timeout  time.Duration
    MaxConns int
    TLS      *tls.Config
}
```
一般配置為

- Addr 和 Port 為必填選項
- Protocal、Timeout、MaxConns 字段，不能為空，但會有默認值
    - 如，協議為 TCP，超時 30 秒，最大連接數 1024 個等
- TLS 安全連結，需要配置相關憑證，可以為空

針對這樣的配置，需要有多種不同的創建不同配置 Server 的函數簽名，如下所示：

```go
func NewDefaultServer(addr string, port int) (*Server, error) {
  return &Server{addr, port, "tcp", 30 * time.Second, 100, nil}, nil
}

func NewTLSServer(addr string, port int, tls *tls.Config) (*Server, error) {
  return &Server{addr, port, "tcp", 30 * time.Second, 100, tls}, nil
}

func NewServerWithTimeout(addr string, port int, timeout time.Duration) (*Server, error) {
  return &Server{addr, port, "tcp", timeout, 100, nil}, nil
}

func NewTLSServerWithMaxConnAndTimeout(addr string, port int, maxconns int, timeout time.Duration, tls *tls.Config) (*Server, error) {
  return &Server{addr, port, "tcp", 30 * time.Second, maxconns, tls}, nil
}
```
> Go 語言不支持重載函數，所以得用不同的函數名來應對不同的配置選項。

## 配置對象方案

要解決前面問題，常見方式為使用一個配置對象：
- 把非必要選項都歸類到一個結構體中 

```go
type Config struct {
    Protocol string
    Timeout  time.Duration
    Maxconns int
    TLS      *tls.Config
}
---
type Server struct {
    Addr string
    Port int
    Conf *Config
}
```
如此一來，只需要一個函數 `NewServer()` 即可，但是需要先構造 `Config` 配置對象
```go
func NewServer(addr string, port int, conf *Config) (*Server, error) {
    //...
}

//Using the default configuratrion
srv1, _ := NewServer("localhost", 9000, nil) 

conf := ServerConfig{Protocol:"tcp", Timeout: 60*time.Duration}
srv2, _ := NewServer("locahost", 9000, &conf)
```

## Builder 模式

熟悉設計模式的用戶，相信會直接採用 builder 模式進行設定配置
```go
Server server = new Server.Builder()
  .addr("localhost")
  .port(80)
  .build();
```

沿著此模式思路，修改前面的例子：
```go
//使用一個 builder 類來做包裝
type ServerBuilder struct {
  Server
}

func (sb *ServerBuilder) Create(addr string, port int) *ServerBuilder {
  sb.Server.Addr = addr
  sb.Server.Port = port
  //其它代碼設置其它成員的默認值
  return sb
}

func (sb *ServerBuilder) WithProtocol(protocol string) *ServerBuilder {
  sb.Server.Protocol = protocol 
  return sb
}

func (sb *ServerBuilder) WithMaxConn( maxconn int) *ServerBuilder {
  sb.Server.MaxConns = maxconn
  return sb
}

func (sb *ServerBuilder) WithTimeOut( timeout time.Duration) *ServerBuilder {
  sb.Server.Timeout = timeout
  return sb
}

func (sb *ServerBuilder) WithTLS( tls *tls.Config) *ServerBuilder {
  sb.Server.TLS = tls
  return sb
}

func (sb *ServerBuilder) Build() (Server) {
  return  sb.Server
}
```
就可以使用這樣方式建立了
```go
sb := ServerBuilder{}
server, err := sb.Create("127.0.0.1", 80).
  WithProtocol("tcp").
  WithMaxConn(1024).
  WithTimeOut(30*time.Second).
  Build()
```
> 不需要額外的 Config 類，**使用鏈式的函數調用的方式來構造一個對象**，只需要多加一個 Builder 類。但在**處理錯誤的時候可能就有點麻煩，不如一個包裝類更好些**。

## Functional Option

先定義一個函數類型，即可使用函數式方式修改前面例子，而使用函數式編程。

```go
type Option func(*Server)

func Protocol(p string) Option {
    return func(s *Server) {
        s.Protocol = p
    }
}
func Timeout(timeout time.Duration) Option {
    return func(s *Server) {
        s.Timeout = timeout
    }
}
func MaxConns(maxconns int) Option {
    return func(s *Server) {
        s.MaxConns = maxconns
    }
}
func TLS(tls *tls.Config) Option {
    return func(s *Server) {
        s.TLS = tls
    }
}
```
> **返回的函數會設置自己的 Server 參數**。如，當調用其中的一個函數 `MaxConns(30)` 時，其返回值是一個 `func(s* Server) { s.MaxConns = 30 } `的函數

Go 語言有可變長參數特性。因此，可以設定一個可變參數 `options`，它可以傳入多個上面的函數，然後使用一個 for-loop 來設置我們的 Server 對象。

```go
func NewServer(addr string, port int, options ...func(*Server)) (*Server, error) {

  srv := Server{
    Addr:     addr,
    Port:     port,
    Protocol: "tcp",
    Timeout:  30 * time.Second,
    MaxConns: 1000,
    TLS:      nil,
  }
  for _, option := range options {
    option(&srv)
  }
  //...
  return &srv, nil
}
```

所以在創建 Server 對象，即可修改為：
```go
s1, _ := NewServer("localhost", 1024)
s2, _ := NewServer("localhost", 2048, Protocol("tcp"))
s3, _ := NewServer("0.0.0.0", 80, Timeout(300 * time.Second), MaxConns(1000))
```
> 整體比前面幾個方案優雅很多。
> 
> 不但解決了使用 **Config 對象方式的需要有一個 config 參數，但在不需要的時候，是放 nil 還是放 Config{}** 的選擇困難問題，**也不需要引用一個 Builder 的控制對象，直接使用函數式編程**，在代碼閱讀上也很優雅。

## 小結

以配置設定為例子，說明為什麼需要 Functional Option 模式，以及各種方案的比較：

- Go 語言不支持重載函數，所以得用不同的函數名來應對不同的配置選項。
- 單獨配置對象：需要在新建函數時，先創建 Config 對象
- Builder 模式：鏈式調用函數，但是需要額外處理錯誤問題
- Functional Option：函數式編程，**返回的函數會設置自己對象的參數**

使用類似前面代碼，建議採用 Fucntional Option 模式，這種方式至少帶來了下面幾種好處：

- 直覺式的編程
- 高度的可配置化
- 很容易維護和擴展
- 新來的人很容易上手
- 不需要煩惱處理 nil 還是 Empty 的問題

## 參考文檔

- [Self referential functions and design](https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html) by Rob Pike

此文章為3月Day14學習筆記，內容來源於極客時間[《左耳聽風》](https://time.geekbang.org/column/article/332603)