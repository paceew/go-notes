# 从源码角度看 Golang 的 net 包补充
> 本文是对《从源码角度看Golang的TCP Socket(epoll)实现》的勘误与补充，结合当前 Go 版本（go1.23+）源码进行验证。
---
## 目录
* [一、Listen 流程勘误：缺少 sys_listen](#一listen-流程勘误缺少-sys_listen)
* [二、Accept 流程补充：goroutine 阻塞 ≠ 系统调用阻塞](#二accept-流程补充goroutine-阻塞--系统调用阻塞)
  * [第一层：非阻塞的 accept4](#第一层非阻塞的-accept4)
  * [第二层：EAGAIN 触发 goroutine 挂起](#第二层eagain-触发-goroutine-挂起不是线程阻塞)
  * [第三层：epoll_wait 驱动唤醒](#第三层epoll_wait-驱动唤醒)
  * [第四层：accept 成功后，新 fd 立即注册 epoll](#第四层accept-成功后新-fd-立即注册-epoll)
  * [Accept 完整调用链](#accept-完整调用链)
* [三、epoll 唤醒 goroutine 的完整机制](#三epoll-唤醒-goroutine-的完整机制)
  * [步骤 1：epoll_ctl ADD 时，把 pd 指针存入 event.Data](#步骤-1epoll_ctl-add-时把-pd-指针存入-eventdata)
  * [步骤 2：goroutine 阻塞时，把 *g 存入 pd.rg 或 pd.wg](#步骤-2goroutine-阻塞时把-g-存入-pdrg-或-pdwg)
  * [步骤 3：epoll_wait 返回事件，取出 pd → 取出 *g → 唤醒](#步骤-3epoll_wait-返回事件取出-pd--取出-g--唤醒)
* [四、fd 复用的 ABA 问题与 fdseq](#四fd-复用的-aba-问题与-fdseq)
* [五、并发读写同一 fd 的保护机制](#五并发读写同一-fd-的保护机制)
  * [读写分离：rg 和 wg 是独立槽位](#读写分离rg-和-wg-是独立槽位)
  * [同向并发：fdMutex 序列化](#同向并发多个-goroutine-同时读fdmutex-序列化)
  * [两层保护结构](#两层保护结构)
* [六、epoll 为何保持全局单份（vs per-P timer）](#六epoll-为何保持全局单份vs-per-p-timer)
* [七、timer 与 goroutine 不在同一 P 的情况](#七timer-与-goroutine-不在同一-p-的情况)
---
## 一、Listen 流程勘误：缺少 `sys_listen`
原文描述的 Listen 流程：
> socket → bind → 初始化 epoll → 注册 fd 到 epoll
**实际源码顺序（`net/sock_posix.go: listenStream()`）：**
```
socket → bind → listen → fd.init()（epoll 初始化 + 注册 fd）
```
关键在于 `bind` 之后必须调用 `listenFunc`（即 `syscall.Listen`），让内核将 socket 转入 `LISTEN` 状态，建立半连接队列（SYN queue）和全连接队列（accept queue）。**没有 listen，accept 无法工作。**
```go
// net/sock_posix.go
func (fd *netFD) listenStream(...) error {
    syscall.Bind(fd.pfd.Sysfd, lsa)      // 1. 绑定地址
    listenFunc(fd.pfd.Sysfd, backlog)    // 2. sys_listen（原文缺失！）
    fd.init()                             // 3. epoll 初始化 + 注册 fd
}
```
`fd.init()` → `pollDesc.init()` 内部通过 `sync.Once` 懒初始化 epoll（`epoll_create1`），然后立即将 listener fd 通过 `epoll_ctl(EPOLL_CTL_ADD)` 注册进去。**这一步在 `net.Listen()` 返回之前就完成了，与 `Accept()` 调用无关。**
---
## 二、Accept 流程补充：goroutine 阻塞 ≠ 系统调用阻塞
原文说"先去系统层 `sys_accept` 等到一个连接请求"，暗示 accept 是阻塞的系统调用。实际上 Go 的 socket 是非阻塞的，整个机制分四层：
### 第一层：非阻塞的 accept4
Linux 下 Go 调用的是 `accept4`，直接把新 fd 设为 `SOCK_NONBLOCK | SOCK_CLOEXEC`：
```go
// internal/poll/sock_cloexec.go
func accept(s int) (int, syscall.Sockaddr, string, error) {
    ns, sa, err := Accept4Func(s, syscall.SOCK_NONBLOCK|syscall.SOCK_CLOEXEC)
    // ...
}
```
### 第二层：EAGAIN 触发 goroutine 挂起（不是线程阻塞）
```go
// internal/poll/fd_unix.go
func (fd *FD) Accept() (int, syscall.Sockaddr, string, error) {
    fd.pd.prepareRead(fd.isFile)   // 重置 pollDesc 读状态
    for {
        s, rsa, _, err := accept(fd.Sysfd)  // 非阻塞调用
        if err == nil {
            return s, rsa, "", nil
        }
        switch err {
        case syscall.EAGAIN:
            fd.pd.waitRead(fd.isFile)   // goroutine 挂起，M 继续跑其他 G
            continue
        case syscall.EINTR:
            continue
        }
    }
}
```
`waitRead()` → `runtime_pollWait()` → `netpollblock()` → **`gopark()`**，当前 goroutine 进入 `Gwaiting` 状态，让出 P，线程（M）继续运行其他 goroutine。
### 第三层：epoll_wait 驱动唤醒
scheduler 的 `findrunnable()` 在无工作可做时调用 `netpoll(delay)`，内部是 `epoll_wait`。有新连接到达时，内核触发 `EPOLLIN`，runtime 将等待的 goroutine 重新放入运行队列：
```
epoll_wait → 取事件 → 找到 pd → netpollready → goready(g) → 重回运行队列
```
### 第四层：accept 成功后，新 fd 立即注册 epoll
`accept4` 返回新 fd 后，`netFD.accept()` **立即**调用 `netfd.init()` 将新 fd 注册到 epoll，不等用户层使用：
```go
// net/fd_unix.go
func (fd *netFD) accept() (netfd *netFD, err error) {
    d, rsa, _, err := fd.pfd.Accept()   // accept4
    // ...
    newFD(d, ...)                        // 包装成 netFD
    netfd.init()                         // 立即 epoll_ctl ADD，用户感知不到
    // ...
    return netfd, nil                    // 这里才返回给用户
}
```
注册时使用**边缘触发（ET）模式**，同时监听读写：
```go
// runtime/netpoll_epoll.go
ev.Events = syscall.EPOLLIN | syscall.EPOLLOUT | syscall.EPOLLRDHUP | syscall.EPOLLET
```
### Accept 完整调用链
```
ln.Accept()
│
├─ TCPListener.accept()
│   └─ netFD.accept()
│       └─ poll.FD.Accept()
│           ├─ [loop] accept4(SOCK_NONBLOCK)
│           │   ├─ 成功 → 返回新 fd
│           │   └─ EAGAIN → pd.waitRead()
│           │                └─ netpollblock() → gopark()  ← goroutine 挂起
│           │
│           └─ [epoll 触发，goroutine 被唤醒] ← scheduler: epoll_wait → netpollready → goready
│
└─ accept4 成功后：
    ├─ newFD()          包装成 netFD
    ├─ netfd.init()  →  epoll_ctl ADD 新 fd（ET 模式）
    └─ setAddr()        记录本端/对端地址
```
---
## 三、epoll 唤醒 goroutine 的完整机制
整条链路分三步：**存指针 → 挂 goroutine → 唤醒 goroutine**。
### 步骤 1：`epoll_ctl ADD` 时，把 `pd` 指针存入 `event.Data`
```go
// runtime/netpoll_epoll.go
func netpollopen(fd uintptr, pd *pollDesc) uintptr {
    var ev syscall.EpollEvent
    ev.Events = syscall.EPOLLIN | syscall.EPOLLOUT | syscall.EPOLLRDHUP | syscall.EPOLLET
    tp := taggedPointerPack(unsafe.Pointer(pd), pd.fdseq.Load())  // pd + 版本号
    *(*taggedPointer)(unsafe.Pointer(&ev.Data)) = tp              // 写入 event.Data
    return syscall.EpollCtl(epfd, syscall.EPOLL_CTL_ADD, int32(fd), &ev)
}
```
### 步骤 2：goroutine 阻塞时，把 `*g` 存入 `pd.rg` 或 `pd.wg`
`pollDesc` 有两个独立槽位：
```go
// runtime/netpoll.go
rg atomic.Uintptr  // pdReady / pdWait / *g（等待读的 goroutine）
wg atomic.Uintptr  // pdReady / pdWait / *g（等待写的 goroutine）
```
`gopark` 真正挂起前回调 `netpollblockcommit`，用 CAS 把 `*g` 写进槽位：
```go
func netpollblockcommit(gp *g, gpp unsafe.Pointer) bool {
    r := atomic.Casuintptr((*uintptr)(gpp), pdWait, uintptr(unsafe.Pointer(gp)))
    // pdWait → *g，原子操作
    return r
}
```
`rg`/`wg` 的状态机：`pdNil(0)` → `pdWait(2)` → `*g（真实 goroutine 指针）`
### 步骤 3：`epoll_wait` 返回事件，取出 `pd` → 取出 `*g` → 唤醒
```go
// runtime/netpoll_epoll.go（epoll_wait 事件处理循环）
tp := *(*taggedPointer)(unsafe.Pointer(&ev.Data))   // 从 event.Data 取出 tagged pointer
pd := (*pollDesc)(tp.pointer())                      // 解出 pd
if pd.fdseq.Load() == tp.tag() {                    // 版本号校验（防 fd 复用幽灵事件）
    delta += netpollready(&toRun, pd, mode)          // 从 pd.rg/wg 取 *g，放入待运行列表
}
```
`netpollready` → `netpollunblock`（CAS 取出 `*g`）→ `netpollgoready` → `goready` → `runqput`。
---
## 四、fd 复用的 ABA 问题与 fdseq
### 问题场景
```
1. fd=5 注册到 epoll，event.Data = (pd_X, tag=3)
2. conn.Close() → pd_X 归还 pollcache，fdseq 递增为 4
3. 新建连接恰好复用 fd=5，pd_X 被重新分配，fdseq=5
4. epoll 内核队列里仍有 fd=5 的旧事件（tag=3）
5. epoll_wait 返回此旧事件 → 取出 (pd_X, tag=3)
6. 检查：pd_X.fdseq.Load()=5 ≠ tag=3 → 丢弃，不错误唤醒新连接的 goroutine
```
### 源码：fd 关闭时递增 fdseq
```go
// runtime/netpoll.go
func (c *pollCache) free(pd *pollDesc) {
    // Increment the fdseq field, so that any currently
    // running netpoll calls will not mark pd as ready.
    fdseq := pd.fdseq.Load()
    fdseq = (fdseq + 1) & (1<<taggedPointerBits - 1)
    pd.fdseq.Store(fdseq)
    // ...
}
```
---
## 五、并发读写同一 fd 的保护机制
### 读写分离：rg 和 wg 是独立槽位
一个 goroutine 挂在 `rg`（等读），另一个挂在 `wg`（等写），互不干扰。
### 同向并发（多个 goroutine 同时读）：fdMutex 序列化
`pd.rg` 只能存一个 goroutine 指针，`netpollblock` 注释明确禁止并发：
```go
// Concurrent calls to netpollblock in the same mode are forbidden, as pollDesc
// can hold only a single waiting goroutine for each mode.
```
在更上层，`fdMutex` 用一个 `uint64` 的 `state` 字段同时管理关闭状态、读锁、写锁、引用计数、读等待数、写等待数：
```go
// internal/poll/fd_mutex.go
// fdMutex.state is organized as follows:
// 1 bit  - whether FD is closed
// 1 bit  - lock for read operations
// 1 bit  - lock for write operations
// 20 bits - total number of references
// 20 bits - number of outstanding read waiters
// 20 bits - number of outstanding write waiters
```
并发 `conn.Read()` 时，只有一个 goroutine 能拿到 `mutexRLock`，其余在 `rsema` 信号量上排队挂起，`rwunlock` 时通过 `runtime_Semrelease` 唤醒下一个。
### 两层保护结构
```
多 goroutine 并发 conn.Read()
        │
        ▼
  fdMutex.readLock()     ← 第一道：同向并发序列化，只放一个进去
        │
        ▼
  accept4/read（非阻塞）
        │ EAGAIN
        ▼
  netpollblock()
    rg CAS: pdNil→pdWait→*g  ← pd.rg 只存一个 g，受 fdMutex 保护
        │
        ▼
  epoll_wait → fdseq 校验 ← 第二道：过滤 fd 复用的幽灵事件
        │ 匹配
        ▼
  netpollunblock → goready  ← 唤醒正确的 goroutine
```
---
## 六、epoll 为何保持全局单份（vs per-P timer）
### timer 改 per-P 解决的是锁热点
timer 操作（add/del/run）在高并发下极其频繁，全局堆的锁是明显热点。timer 是纯用户空间内存操作，可以完全分片到各 P，互不干扰。
### epoll 不适合 per-P 的原因
| | timer | epoll |
|--|--|--|
| 本质 | 用户空间内存堆 | 内核对象（fd） |
| 操作频率 | 极高（微秒级） | 低（连接级别） |
| 与 P 的绑定 | 可绑定，创建时固定到当前 P | 无绑定，goroutine 可跨 P 迁移 |
| 瓶颈 | 全局锁竞争 | 无明显锁热点 |
| per-P 收益 | 消除锁竞争 | fd 迁移代价（epoll_ctl DEL+ADD）抵消收益 |
**核心原因**：fd 注册 epoll 时不属于任何 P。goroutine 随时可以被其他 P steal，如果 fd 绑定在 P1 的 epoll 上，P2 steal 了这个 goroutine 后收不到事件，必须重新 `epoll_ctl DEL + ADD`，代价极高。
`sched.lastpoll` 通过 CAS 保证同一时刻只有一个 M 在做阻塞的 `epoll_wait`：
```go
// runtime/proc.go
if netpollinited() && ... && sched.lastpoll.Swap(0) != 0 {
    list, delta := netpoll(delay) // block until new work is available
    sched.lastpoll.Store(now)
```
唤醒的 goroutine 通过 `injectglist` 注入全局队列，所有 P 均可 steal。
---
## 七、timer 与 goroutine 不在同一 P 的情况
### 全局 timer 堆不存在
只有 per-P 的堆，`timers` 字段只在 `p` struct 中，没有全局堆。
### timer 和 goroutine 确实可能在不同 P
**这是完全正常的设计**，时序如下：
```
G 在 P1 上                  P1 的 timer 堆           pd.rg
──────────────              ──────────────           ──────
SetReadDeadline() ───────── timer 加入 P1 堆
conn.Read()
  EAGAIN → gopark ──────────────────────────────────  *G (G 悬空)
  G 离开 P1，不属于任何 P
               P2 work-stealing
               检查 P1 的 timer 堆，发现有到期 timer
               执行 netpollReadDeadline
                 netpollunblock ────────────────────── 取出 *G
                 goready(G)
                   runqput(P2, G)  ← G 放入 P2 队列
G 在 P2 上继续执行，收到 ErrDeadlineExceeded
```
`maybeAdd()` 使用 `acquirem()` 固定到当前 M，timer 加入调用时所在 P 的堆。但 timer 的回调 `netpolldeadlineimpl` 只需要从 `pd.rg` 取出 `*g`，调 `goready` 将其放入**执行回调的那个 P** 的本地队列，与 timer 创建时的 P 无关。
**timer 和 goroutine 之间的联系只有 `pd.rg`/`pd.wg` 这个指针，不依赖任何 P 的本地状态。**
同时 `pd.rseq`/`pd.wseq` 作为版本号，防止 timer 重置后旧 timer 回调错误唤醒 goroutine（与 epoll 的 `fdseq` 机制一致）。