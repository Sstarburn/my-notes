# 异步回调（Asynchronous Callback）原理与 C# 实现

# 异步回调（Asynchronous Callback）原理与 C# 实现

## 一、什么是异步回调

### 1.1 同步 vs 异步

```
同步调用：
  ┌─────┐  调用  ┌──────────┐  返回结果  ┌─────┐
  │ 主线程 │──────→│ DoWork() │──────────→│ 主线程 │
  └─────┘  阻塞  └──────────┘           └─────┘
  (等待期间什么都不能做)

异步调用（回调）：
  ┌─────┐  调用     ┌──────────┐
  │ 主线程 │──────→  │ DoWork() │
  └─────┘           └──────────┘
      │                 │ 完成时
      │  ←──────────────┘
      │  回调通知 (Callback)
  (不等，继续做其他事)
```

**同步**：调用后阻塞等待结果返回，期间不能做其他事。

**异步**：调用后立即返回，不阻塞主线程。操作完成后通过 **回调函数** 通知结果。

### 1.2 回调（Callback）的本质

> **回调** = 将一个函数作为参数传递给另一个函数，当某个操作完成时由被调用方执行这个函数。

```
public void DoWork(Action onComplete)  // onComplete = 回调函数
{
    // 做耗时操作...
    onComplete();  // 操作完成，调用回调通知调用方
}
```

---

## 二、为什么需要异步回调

### 2.1 不阻塞 UI/主线程

```csharp
// ❌ 同步：界面卡死
void Button_Click()
{
    Thread.Sleep(5000);  // UI 冻结 5 秒
    label.Text = "完成";
}

// ✅ 异步：界面流畅
void Button_Click()
{
    DoWorkAsync(() => label.Text = "完成");  // 立即返回，不卡UI
}
```

### 2.2 提高并发吞吐

- 单线程等待 I/O（文件、网络、数据库）时浪费 CPU
- 异步让线程在等待期间处理其他任务
- 避免线程池耗尽

### 2.3 回调地狱（Callback Hell）

异步回调嵌套过多导致代码难以阅读：

```csharp
// 😱 回调地狱
LoginAsync(user, token =>
{
    GetUserDataAsync(token, data =>
    {
        ProcessDataAsync(data, result =>
        {
            SaveResultAsync(result, () =>
            {
                Console.WriteLine("全部完成");
            });
        });
    });
});
```

---

## 三、C# 中异步回调的实现方式

### 3.1 委托（Delegate）— 最朴素的回调

**定义**：委托是 C# 中 **类型安全的函数指针**。

```csharp
// 声明委托类型
public delegate void CallbackDelegate(string result);

// 接受委托作为参数
public void DoWorkAsync(string input, CallbackDelegate callback)
{
    ThreadPool.QueueUserWorkItem(_ =>
    {
        string result = input.ToUpper();
        Thread.Sleep(1000);        // 模拟耗时
        callback(result);          // 完成时调用回调
    });
}

// 使用
DoWorkAsync("hello", result =>
{
    Console.WriteLine($"回调结果: {result}");
});
Console.WriteLine("这句先执行");
```

**输出**：
```
这句先执行
回调结果: HELLO
```

### 3.2 内置委托类型：Action / Func

C# 提供了内置委托，无需自定义：

| 类型 | 签名 | 用途 |
|------|------|------|
| `Action` | `void()` | 无参无返回值回调 |
| `Action<T>` | `void(T)` | 1参无返回值回调 |
| `Action<T1,T2>` | `void(T1,T2)` | 2参无返回值回调 |
| `Func<TResult>` | `TResult()` | 无参有返回值 |
| `Func<T, TResult>` | `TResult(T)` | 1参有返回值 |

```csharp
public void DoWorkAsync(string input, Action<string> onComplete)
{
    Task.Run(() =>
    {
        string result = input.ToUpper();
        onComplete(result);  // Action<string> 回调
    });
}

DoWorkAsync("hello", r => Console.WriteLine(r));
```

### 3.3 事件（Event）— 多播回调

**事件** = 委托的扩展，支持 **多个回调订阅**。

```csharp
public class Downloader
{
    // 声明事件
    public event Action<string>? OnDownloadComplete;

    public void DownloadAsync(string url)
    {
        Task.Run(() =>
        {
            // 模拟下载...
            Thread.Sleep(1000);
            OnDownloadComplete?.Invoke(url);  // 触发事件，通知所有订阅者
        });
    }
}

// 使用
var dl = new Downloader();
dl.OnDownloadComplete += url => Console.WriteLine($"{url} 下载完成");   // 订阅者1
dl.OnDownloadComplete += url => File.WriteAllText("log.txt", url);      // 订阅者2
dl.DownloadAsync("https://example.com/file.zip");
```

### 3.4 APM 模式（Asynchronous Programming Model）

.NET 1.0 时代，使用 `BeginXxx` / `EndXxx` 模式：

```csharp
// APM：Begin + AsyncCallback
FileStream fs = new FileStream("data.txt", FileMode.Open);
byte[] buffer = new byte[1024];

IAsyncResult ar = fs.BeginRead(buffer, 0, buffer.Length, asyncResult =>
{
    int bytesRead = fs.EndRead(asyncResult);
    Console.WriteLine($"读取了 {bytesRead} 字节");
}, null);
```

`BeginRead` 的最后一个参数就是 `AsyncCallback` 回调委托。

### 3.5 EAP 模式（Event-based Asynchronous Pattern）

.NET 2.0 时代，使用事件 + `AsyncCompletedEventArgs`：

```csharp
WebClient client = new WebClient();
client.DownloadStringCompleted += (sender, e) =>
{
    if (e.Error == null)
        Console.WriteLine($"下载完成，长度：{e.Result.Length}");
};
client.DownloadStringAsync(new Uri("https://example.com"));
```

### 3.6 TAP 模式（Task-based Asynchronous Pattern）— 现代方式

.NET 4.0+ 引入 `Task` 和 `Task<TResult>`，配合 **延续（Continuation）** 实现回调：

```csharp
public Task<string> DoWorkAsync(string input)
{
    return Task.Run(() =>
    {
        return input.ToUpper();
    });
}

// 延续 = 结构化回调
DoWorkAsync("hello").ContinueWith(task =>
{
    Console.WriteLine($"结果: {task.Result}");
});
```

### 3.7 async / await — 最终解决方案

.NET 4.5+，用同步写法写异步代码，**编译器在幕后生成回调状态机**：

```csharp
public async Task<string> DoWorkAsync(string input)
{
    await Task.Delay(1000);  // 不阻塞线程
    return input.ToUpper();
}

// 调用方
async void Button_Click()
{
    string result = await DoWorkAsync("hello");
    label.Text = result;  // 自动回到 UI 线程
}
```

**编译器生成的状态机**（伪代码）：

```
DoWorkAsync 状态机：
  ┌─────────────────────────────┐
  │ 状态: 0(初始)               │
  │ 输入: "hello"               │
  ├─────────────────────────────┤
  │ await Task.Delay(1000)      │
  │   → 挂起，返回 Task 给调用方 │
  │   → 1秒后回调，继续执行      │
  ├─────────────────────────────┤
  │ 状态: 1                     │
  │ result = "hello".ToUpper()  │
  │ return "HELLO"              │
  └─────────────────────────────┘
```

**async/await 解决回调地狱**：

```csharp
// ✅ 优雅：像同步一样顺序书写
async Task ProcessAllAsync()
{
    var token = await LoginAsync(user);
    var data = await GetUserDataAsync(token);
    var result = await ProcessDataAsync(data);
    await SaveResultAsync(result);
    Console.WriteLine("全部完成");
}
```

---

## 四、ProfControl 中的回调模式分析

### 4.1 GoTo 回调（最常见）

来自 GYSY_Simulation 代码：

```csharp
agv.GoTo(sta.Mark, ret =>
{
    // 回调：AGV 到达目标站台后执行
    ChargeNum++;
    agv.SetValue("车体状况", "正在充电");
    RemoveHash(agv.Name);
});
```

**解读**：
- `GoTo` 是异步方法，立即返回
- 第二个参数是 `Action` 回调，AGV 到达站台时触发
- 回调中执行到达后的业务逻辑

### 4.2 API.GetShortest 回调

```csharp
API.GetShortest(agv, stas, sta =>
{
    if (AddHash(agv.Name, sta.Mark))
    {
        agv.GoTo(sta.Mark, ret =>
        {
            // 嵌套回调：到达后才执行
        });
    }
});
```

### 4.3 定时器回调

```csharp
API.SetInterval(tc =>
{
    // 每秒执行一次的回调
    SimulateCharge();
}, 1000);
```

### 4.4 事件回调

```csharp
agv.GetNewRoutesEvent += Agv_GetNewRoutesEvent;
```

### 4.5 为什么 ProfControl 使用回调而非 async/await

| 原因 | 说明 |
|------|------|
| **平台限制** | ProfControl API 是旧版设计，GoTo 等方法是基于回调的异步模式 |
| **持续轮询** | 主循环是 `API.SetInterval` 定时器，非 await 驱动 |
| **兼容性** | 脚本运行在宿主平台上，可能不支持最新 C# 特性 |
| **代码直观** | 状态机 + 定时器轮询的模式更容易调试和追踪 |

---

## 五、回调实现的核心机制

### 5.1 状态机模式（编译器背后）

```csharp
// async/await 背后编译器生成了一个状态机类
[AsyncStateMachine(typeof(MyStateMachine))]
public Task MyMethodAsync()
{
    MyStateMachine stateMachine = new MyStateMachine();
    stateMachine.builder = AsyncTaskMethodBuilder.Create();
    stateMachine.state = -1;
    stateMachine.builder.Start(ref stateMachine);
    return stateMachine.builder.Task;
}
```

### 5.2 同步上下文（SynchronizationContext）

**问题**：回调在哪个线程执行？UI 控件只能在 UI 线程更新。

```csharp
// await 自动捕获 SynchronizationContext，回到原线程
async void Button_Click()
{
    await Task.Run(() => Compute());  // 后台线程
    label.Text = "完成";               // 自动回到 UI 线程
}

// 回调方式：需要手动封送
void Button_Click()
{
    Task.Run(() =>
    {
        string result = Compute();
        this.Invoke(() => label.Text = result);  // 手动切回 UI 线程
    });
}
```

### 5.3 回调的执行时机

```
主线程：────A────B────────────────C───────────→
                   ↘              ↗
工作线程：           └──耗时操作──┘
                      完成后回调

A: 发起异步操作
B: 立即返回，主线程继续
C: 回调在某个线程上执行（线程池/完成端口）
```

---

## 六、常见问题与最佳实践

### 6.1 回调中的异常处理

```csharp
// ❌ 回调外的 try-catch 抓不到回调内的异常
try
{
    DoWorkAsync(() => { throw new Exception("崩溃"); });
}
catch { /* 抓不到！*/ }

// ✅ 回调内自己处理
DoWorkAsync(() =>
{
    try { /* 业务 */ }
    catch (Exception ex) { /* 处理异常 */ }
});

// ✅ async/await 方式
try
{
    await DoWorkAsync();
}
catch { /* 能抓到 */ }
```

### 6.2 回调中的 this 引用（闭包陷阱）

```csharp
// ⚠️ 回调捕获了外部变量
for (int i = 0; i < 10; i++)
{
    DoWorkAsync(i, result =>
    {
        Console.WriteLine(i);  // 可能输出全是一样的值！
    });
}

// ✅ 使用局部副本
for (int i = 0; i < 10; i++)
{
    int captured = i;  // 创建副本
    DoWorkAsync(captured, result =>
    {
        Console.WriteLine(captured);
    });
}
```

### 6.3 回调地狱的演化

```
传统模式                  Task延续              async/await
────────────────────────────────────────────────────────
BeginRead(               Task.Run(()           await Task
  buffer,                  => Read())             .Delay(1000);
  callback => {           .ContinueWith(        await ReadAsync();
    Process(data);          task => {           await ProcessAsync(
    BeginWrite(               Process();          data);
      data,                 });                await WriteAsync(
      callback2 => {                             data);
        ...
      });
    });
  });
```

---

## 七、总结

| 概念 | 一句话 |
|------|--------|
| **回调** | 把一段代码（函数）作为参数传给另一个函数，让它在完成时调用你 |
| **委托** | C# 中类型安全的回调载体，`Action`/`Func` 是内置委托 |
| **事件** | 多播委托，支持多个订阅者的回调 |
| **Task** | 异步操作的抽象，`ContinueWith` = 结构化回调 |
| **async/await** | 编译器用状态机把 async/await 编译成回调链，让你用同步写法写异步代码 |

**ProfControl 中的回调**：GoTo、GetShortest、SetInterval 等方法都采用传统回调模式。理解回调机制是理解这套仿真调度代码的关键。

---

*笔记生成日期：2026-06-22*
