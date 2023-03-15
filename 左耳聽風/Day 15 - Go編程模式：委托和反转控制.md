# Day 15 - Go編程模式：委托和反转控制

[控制反轉（Inversion of control，IOC）](https://en.wikipedia.org/wiki/Inversion_of_control)是一種軟件設計方法，主要思想為把控制邏輯和業務邏輯分開。
> 不要在業務邏輯裡面寫控製邏輯，要讓業務邏輯依賴控製邏輯

舉個例子說明，**開關和電燈**，這邊提到的開關為控制邏輯，電器為業務邏輯（內部流程運作讓電燈發光）。

我們不要在電器中實現開關，而是要把**開關抽象成一種規範或是協議**，讓電器都依賴這個協議。這樣不只降低複雜度，也提高了代碼複用程度，也就是說只要控製裝置實現了開關協議，都可以控制電燈。

## 嵌入 & 委託

### 結構體嵌入

在 Go 語言中，透過組合方式可以輕鬆把結構體嵌入到另一個結構體中

```go
type Widget struct {
    X, Y int
}
type Label struct {
    Widget        // Embedding (delegation)
    Text   string // Aggregation
}
```
便可以直接使用組合的結構體字段
```go
label := Label{Widget{10, 10}, "State:"}

label.X = 11
label.Y = 12
```
> 如果在Label 結構體里出現了重名，就需要解決重名問題。
> 要表明是哪個結構體的字段，如 `label.Wedget.X` 表明是嵌入過來的

如此一來，就可以在結構設計上進行層層分解。
```go
type Button struct {
    Label // Embedding (delegation)
}

type ListBox struct {
    Widget          // Embedding (delegation)
    Texts  []string // Aggregation
    Index  int      // Aggregation
}
```

### 方法重寫

延續上面提到的思路，需要兩個接口：

- Painter 把組件畫出來
- Clicker 表示點擊事件

```go
type Painter interface {
    Paint()
}
 
type Clicker interface {
    Click()
}
```
Go 語言特性，隱形接口
實現如下：
```go
func (label Label) Paint() {
  fmt.Printf("%p:Label.Paint(%q)\n", &label, label.Text)
}

//因為這個接口可以通過 Label 的嵌入帶到新的結構體，
//所以，可以在 Button 中重載這個接口方法
func (button Button) Paint() { // Override
    fmt.Printf("Button.Paint(%s)\n", button.Text)
}
func (button Button) Click() {
    fmt.Printf("Button.Click(%s)\n", button.Text)
}


func (listBox ListBox) Paint() {
    fmt.Printf("ListBox.Paint(%q)\n", listBox.Texts)
}
func (listBox ListBox) Click() {
    fmt.Printf("ListBox.Click(%q)\n", listBox.Texts)
}
```
> 重點提醒，Button.Paint() 接口可以通過 Label 的嵌入帶到新的結構體，如果 Button.Paint() 不實現的話，會調用 Label.Paint() ，所以，在 Button 中聲明 Paint() 方法，相當於 Override。

### 嵌入結構多態

可以使用接口多態，或是使用泛型的 `Any（interface{}`） 來多態，但需要類型斷言

```go
button1 := Button{Label{Widget{10, 70}, "OK"}}
button2 := NewButton(50, 70, "Cancel")
listBox := ListBox{Widget{10, 40}, 
    []string{"AL", "AK", "AZ", "AR"}, 0}

for _, painter := range []Painter{label, listBox, button1, button2} {
    painter.Paint()
}
 
for _, widget := range []interface{}{label, listBox, button1, button2} {
  widget.(Painter).Paint() // .() 做了類型斷言，判斷 Pianter
  if clicker, ok := widget.(Clicker); ok {
    clicker.Click()
  }
  fmt.Println() // print a empty line 
}
```

## 反轉控制

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
```

### 實現 Undo 功能

透過封裝 IntSet 成 UndoableIntSet，代碼如下：
```go
type UndoableIntSet struct { // Poor style
    IntSet    // Embedding (delegation)
    functions []func()
}
 
func NewUndoableIntSet() UndoableIntSet {
    return UndoableIntSet{NewIntSet(), nil}
}
 

func (set *UndoableIntSet) Add(x int) { // Override
    if !set.Contains(x) {
        set.data[x] = true
        set.functions = append(set.functions, func() { set.Delete(x) })
    } else {
        set.functions = append(set.functions, nil)
    }
}


func (set *UndoableIntSet) Delete(x int) { // Override
    if set.Contains(x) {
        delete(set.data, x)
        set.functions = append(set.functions, func() { set.Add(x) })
    } else {
        set.functions = append(set.functions, nil)
    }
}

func (set *UndoableIntSet) Undo() error {
    if len(set.functions) == 0 {
        return errors.New("No functions to undo")
    }
    index := len(set.functions) - 1
    if function := set.functions[index]; function != nil {
        function()
        set.functions[index] = nil // For garbage collection
    }
    set.functions = set.functions[:index]
    return nil
}
```
> 此方式缺點為，**Undo 操作是一種控製邏輯**。所以複用功能時候，需要修改很多地方，因為**跟 IntSet 業務邏輯高耦合**

### 反轉依賴

另一個方式，先聲明一個函數接口，表示 Undo 控制可以接受的**函數簽名**是什麼樣的
```go
type Undo []func()
```

接著，可以修改 Undo 控制邏輯為：
```go
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
    functions[index] = nil // For garbage collection
  }
  *undo = functions[:index]
  return nil
}
```
> Undo 就是一個類型，所以可以為一個結構體，或是一個函數數組

然後，在 IntSet 中嵌入 Undo 功能
```go
type IntSet struct {
    data map[int]bool
    undo Undo
}
 
func NewIntSet() IntSet {
    return IntSet{data: make(map[int]bool)}
}

func (set *IntSet) Undo() error {
    return set.undo.Undo()
}
 
func (set *IntSet) Contains(x int) bool {
    return set.data[x]
}

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
```
> 控製反轉，IOC，為業務邏輯 IntSet 依賴 Undo，依賴一個協議。
> 
> **這個協議可以為一個沒有參數的函數數組**。如此一來，Undo 代碼就可以重用了


此文章為3月Day15學習筆記，內容來源於極客時間[《左耳聽風》](https://time.geekbang.org/column/article/332605)