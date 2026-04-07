## 目录

- [go func()流程](#go-func流程)
- [sync.Mutex](#syncmutex)
- [迭代器 iter.Pull 实现](#迭代器-iterpull-实现)
  - [为什么需要两个栈？](#为什么需要两个栈)
  - [和 goroutine 的关键区别](#和-goroutine-的关键区别)
  - [所以为什么不用一个栈？](#所以为什么不用一个栈)

---

## go func()流程
1. 分配一个G，从gfree取或是创建，分配2kb初始栈，_Gidle → _Grunnable
2. 放到P的本地队列(lokc free)runnext标记，无锁，开销极小
3. M尝试获取P执行G，从g0栈切换到g栈， _Grunnable → _Grunning
5. 执行完g切回g0，g放到gfree以便复用

## sync.Mutex
1. 快速路径：无竞争直接 CAS 拿锁
2. 自旋：正常模式下自旋几次，减少休眠
3. 正常模式：新来 G 不会休眠，自旋过程中更容易获得锁
4. 饥饿模式：等待 > 1ms 自动开启，保证公平，不饿死 G
5. TryLock：纯 CAS，不排队、不阻塞
6. Unlock：释放锁，正常模式：唤醒一个 G；饥饿模式：唤醒队列第一个 G，G 直接获得锁，不进行争抢
7. 休眠/唤醒 通过信号量sema实现


## 迭代器 iter.Pull 实现

这个问题很有深度。`iter.Pull` 用的**不是两个 goroutine，而是两个协程栈（coroutine stack）**，这是本质区别。
### 为什么需要两个栈？
核心矛盾在于：**`yield` 必须能在任意嵌套调用深度处"暂停"**，而暂停就意味着要保存当时完整的调用栈。
举个极端的例子：
```go
func treeSeq(node *TreeNode, yield func(int) bool) {
    if node == nil { return }
    treeSeq(node.Left, yield)   // 递归进去
    yield(node.Value)            // ← 在这里暂停，此时栈上有 N 层递归帧
    treeSeq(node.Right, yield)  // 恢复后继续
}
```
当 `yield(node.Value)` 被调用时，栈上可能有几十层递归帧。如果只有一个栈，"暂停"意味着要把整个调用栈都拆掉，然后恢复时再重建——这就回到了手写状态机的老路，完全失去迭代器的意义。
所以**两个栈的本质原因**是：
```
┌─────────────────┐      ┌─────────────────┐
│  generator 栈   │      │  consumer 栈    │
│                 │      │                 │
│  treeSeq(root)  │      │  next()         │
│  treeSeq(left)  │ ←──► │  c.Stream(...)  │
│  treeSeq(left2) │      │  handler(...)   │
│  yield(v)  ←暂停│      │                 │
└─────────────────┘      └─────────────────┘
```
两个栈交替执行，各自保存自己的调用上下文。
### 和 goroutine 的关键区别
`iter.Pull` 在 Go 1.23 用的是 runtime 内部的 `newcoro` / `coroswitch`，**不走 goroutine 调度器**：
| | goroutine | coroutine（iter.Pull 用的） |
|---|---|---|
| 调度 | Go 调度器，可并行 | 手动切换，永远串行 |
| 切换开销 | 较高（涉及调度） | 极低（等同函数调用） |
| 是否并发 | 可以 | 不可以，这是保证 |
| 栈 | 独立 | 独立，但共享同一 goroutine |
### 所以为什么不用一个栈？
用一个栈只有一种可能：把 `seq` 编译成**显式状态机**——用 `switch/case` + 手动保存所有局部变量。这正是 Python `yield`、C# `yield return` 在编译期做的事（编译器帮你生成状态机）。
Go 没有这个编译器变换，所以用运行时双栈协程来实现同等语义，代价是多一个栈，换来的是写法完全自然。