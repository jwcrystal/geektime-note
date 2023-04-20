# 編程範式遊記（8）- Go 語言的委託模式

## 委託的簡單示例

```go
type Widget struct {
    X, Y int
}

type Label struct {
    Widget        // Embedding (delegation)
    Text   string // Aggregation
    X int         // Override 
}

func (label Label) Paint() {
  // [0xc4200141e0] - Label.Paint("State")
    fmt.Printf("[%p] - Label.Paint(%q)\n", 
      &label, label.Text)
}
```

就可以照下面方式編程
```go
label := Label{Widget{10, 10}, "State", 100}

// X=100, Y=10, Text=State, Widget.X=10
fmt.Printf("X=%d, Y=%d, Text=%s Widget.X=%d\n", 
  label.X, label.Y, label.Text, 
  label.Widget.X)
fmt.Println()
// {Widget:{X:10 Y:10} Text:State X:100} 
// {{10 10} State 100}
fmt.Printf("%+v\n%v\n", label, label)

label.Paint()
```

擴張 struct Label 的用法
```go
type Button struct {
    Label // Embedding (delegation)
}
 
func NewButton(x, y int, text string) Button {
    return Button{Label{Widget{x, y}, text, x}}
}
func (button Button) Paint() { // Override
    fmt.Printf("[%p] - Button.Paint(%q)\n", 
      &button, button.Text)
}
func (button Button) Click() {
    fmt.Printf("[%p] - Button.Click()\n", &button)
}
```
```go
type ListBox struct {
    Widget          // Embedding (delegation)
    Texts  []string // Aggregation
    Index  int      // Aggregation
}
func (listBox ListBox) Paint() {
    fmt.Printf("[%p] - ListBox.Paint(%q)\n", 
      &listBox, listBox.Texts)
}
func (listBox ListBox) Click() {
    fmt.Printf("[%p] - ListBox.Click()\n", &listBox)
}
```

聲明兩個接口用於多型
```go
type Painter interface {
    Paint()
}

type Clicker interface {
    Click()
}
```

就可以完成下面例子
```go
button1 := Button{Label{Widget{10, 70}, "OK", 10}}
button2 := NewButton(50, 70, "Cancel")
listBox := ListBox{Widget{10, 40}, 
    []string{"AL", "AK", "AZ", "AR"}, 0}

fmt.Println()
//[0xc4200142d0] - Label.Paint("State")
//[0xc420014300] - ListBox.Paint(["AL" "AK" "AZ" "AR"])
//[0xc420014330] - Button.Paint("OK")
//[0xc420014360] - Button.Paint("Cancel")
for _, painter := range []Painter{label, listBox, button1, button2} {
  painter.Paint()
}

fmt.Println()
//[0xc420014450] - ListBox.Click()
//[0xc420014480] - Button.Click()
//[0xc4200144b0] - Button.Click()
for _, widget := range []interface{}{label, listBox, button1, button2} {
    if clicker, ok := widget.(Clicker); ok {
      clicker.Click()
    }
}
```

例子可以視為 Go 語言中的委託和接口多態的編程方式，其實是面向對象和原型編程綜合的玩法。

## 一個 Undo 的委託重構

先聲明一個數據容器
```go
type IntSet struct {
    data map[int]bool
}

func NewIntSet() IntSet {
    return IntSet{make(map[int]bool)}
}

func (set *IntSet) Add(x int) {
    set.data[x] = true
}

func (set *IntSet) Delete(x int) {
    delete(set.data, x)
}

func (set *IntSet) Contains(x int) bool {
    return set.data[x]
}

func (set *IntSet) String() string { // Satisfies fmt.Stringer interface
    if len(set.data) == 0 {
        return "{}"
    }
    ints := make([]int, 0, len(set.data))
    for i := range set.data {
        ints = append(ints, i)
    }
    sort.Ints(ints)
    parts := make([]string, 0, len(ints))
    for _, i := range ints {
        parts = append(parts, fmt.Sprint(i))
    }
    return "{" + strings.Join(parts, ",") + "}"
}
```

正常用法：
```go
ints := NewIntSet()
for _, i := range []int{1, 3, 5, 7} {
    ints.Add(i)
    fmt.Println(ints)
}
for _, i := range []int{1, 2, 3, 4, 5, 6, 7} {
    fmt.Print(i, ints.Contains(i), " ")
    ints.Delete(i)
    fmt.Println(ints)
}
```

如果這時候要加上 Undo() 做法，基本做法就是取修改上面做法，並記錄下每次 function 的執行結果，用來作為 Undo 的值。但是這樣的缺點就是需要重寫原有的 IntSet() 全部功能，那如果有要擴增其他功能，不就改不完？

這時候可以使用前面學到的泛型編程、函數式編程、IoC 等範式。

設計思路：

- 我們先聲明一個 Undo[] 的函數切片（其實是一個棧）；
- 並實現一個通用 Add()。其需要一個函數指針，並把這個函數指針存放到 Undo[] 函數數組中。
- 在 Undo() 的函數中，我們會遍歷 Undo[]函數數組，並執行之，執行完後就彈棧

```go
type Undo []func()

func (undo *Undo) Add(function func()) {
    *undo = append(*undo, function)
}

func (undo *Undo) Undo() error {
    functions := *undo
    if len(functions) == 0 {
        return errors.New("No functions to undo")
    }
    index := len(functions) - 1
    if function := functions[index]; function != nil {
        function()
        functions[index] = nil // Free closure for garbage collection
    }
    *undo = functions[:index] // remove object that has been undo
    return nil
}
```

改寫原有的代碼
```go
type IntSet struct {
    data map[int]bool
    undo Undo
}

func NewIntSet() IntSet {
    return IntSet{data: make(map[int]bool)}
}
```

在 Add() 和 Delete() 中實現 Undo()：

- Add 操作時加入 Delete 操作的 Undo。
- Delete 操作時加入 Add 操作的 Undo。

```go
func (set *IntSet) Add(x int) {
    if !set.Contains(x) {
        set.data[x] = true
        set.undo.Add(func() { set.Delete(x) })
    } else {
        set.undo.Add(nil)
    }
}

func (set *IntSet) Delete(x int) {
    if set.Contains(x) {
        delete(set.data, x)
        set.undo.Add(func() { set.Add(x) })
    } else {
        set.undo.Add(nil)
    }
}

func (set *IntSet) Undo() error {
    return set.undo.Undo()
}

func (set *IntSet) Contains(x int) bool {
    return set.data[x]
}
```

## 小結

我們就可以看到，已經把 Undo 接口把 Undo 的流程給抽象出來，而要怎麼 Undo 的事交給了業務代碼來維護（通過註冊一個 Undo 的方法）。這樣在 Undo 的時候，就可以回調方法來做與業務相關的 Undo 操作了。

跟 map、reduce、filter 這樣的**只關心控制流程，不關心業務邏輯**的做法是不是很像。一開始用基本做法在 IntSet 類中加入 Undo() 來包裝IntSet類，到反過來在 IntSet 裡依賴 Undo 類，這就是控制反轉 IoC。

文章 4 月 Day20 學習筆記，內容來源於極客時間 [《左耳聽風》](https://time.geekbang.org/column/article/2748)