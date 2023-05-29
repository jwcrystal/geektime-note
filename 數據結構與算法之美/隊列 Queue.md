# 隊列 Queue

## 如何理解隊列？

FIFO(First In First Out)，先進先出，為典型的隊列結構。

跟棧相同，也是一種操作受限的線性表數據結構，而最基本的操作也是兩個：
- **入隊(Enqueue)**：加入數據到隊尾。
- **出隊(Dequque)**：從隊首取出數據。

![](media/16820408628811/16822429548018.jpg)

## 順序隊列和鏈式隊列

下面為 java、go 實現的順序隊列：
```java
// 用數組實現的隊列
public class ArrayQueue {
  // 數組：items，數組大小：n
  private String[] items;
  private int n = 0;
  // head表示隊頭下標，tail表示隊尾下標
  private int head = 0;
  private int tail = 0;

  // 申請一個大小為capacity的數組
  public ArrayQueue(int capacity) {
    items = new String[capacity];
    n = capacity;
  }

  // 入隊
  public boolean enqueue(String item) {
    // 如果tail == n 表示隊列已經滿了
    if (tail == n) return false;
    items[tail] = item;
    ++tail;
    return true;
  }

  // 出隊
  public String dequeue() {
    // 如果head == tail 表示隊列為空
    if (head == tail) return null;
    // 為了讓其他語言的同學看的更加明確，把--操作放到單獨一行來寫了
    String ret = items[head];
    ++head;
    return ret;
  }
}
```
```go
# 基於切片實現順序隊列
type T int

type Queue struct {
    array []T
    sync.RWMutex
}

func NewQueue(){
    return &Queue{
        array = []T{}
    }
}

func (q *Queue) Enqueue(t T) {
    q.Lock()
    q.array = append(q.array, t)
    q.Unlock()
}

func (q *Queue) Dequeue() (*T, error){
    if len(q.array) == 0 {
        return nil, fmt.Errorf("queue empty")
    }
    item := q.array[0]
    q.Lock()
    q.array = q.array[1:len(q.array) - 1]
    q.UnLock()
    return item, nil
}
```

對於棧來說，我們只需要一個棧頂指針就可以了。但是隊列需要兩個指針：一個是 head 指針，指向隊頭；一個是 tail 指針，指向隊尾。

下圖示意：
![](media/16820408628811/16822440839296.jpg)

優化上面的代碼，以 java 為例：
- 頭尾指針只會一直往右，當到最右邊時候，即使數組有空間，也無法操作。
- 因此，需要繼續先進行數據搬移後，才能繼續操作數組，這樣的時間複雜度為 O(n)。
- 優化思想為，在入隊時候進行數據搬移。

```java
   // 入隊操作，將item放入隊尾
  public boolean enqueue(String item) {
    // tail == n表示隊列末尾沒有空間了
    if (tail == n) {
      // tail ==n && head==0，表示整個隊列都佔滿了
      if (head == 0) return false;
      // 數據搬移
      for (int i = head; i < tail; ++i) {
        items[i-head] = items[i];
      }
      // 搬移完之後重新更新head和tail
      tail -= head;
      head = 0;
    }
    
    items[tail] = item;
    ++tail;
    return true;
  }
```
![](media/16820408628811/16822445116947.jpg)

出隊操作的時間複雜度仍然是 O(1)，但入隊操作的時間複雜度也還是 O(1)，因為最差時間複雜度是O(n)，最好是O(1)，均攤後就是O(1)。

### 基於鏈式隊列的實現

如下圖所示，
```java
# 入隊時
tail->next= new_node;
tail = tail->next;
# 出隊時
head = head->next;
```

![](media/16820408628811/16822445970269.jpg)

## 循環隊列

前面可以看到在底層是數組結構時，在 tail == n 時會出現數據搬移。

用循環思想，即可避免掉數據搬移動作，最關鍵的是，**確定好隊空和隊滿的判定條件**。

![](media/16820408628811/16822447246312.jpg)



文章 May Day2 學習筆記，內容來源於極客時間 [《數據結構與算法之美》](https://time.geekbang.org/column/article/41222)
