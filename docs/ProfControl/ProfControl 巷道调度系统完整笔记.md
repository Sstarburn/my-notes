# ProfControl 巷道调度系统完整笔记

#ProfControl 巷道调度系统完整笔记

## 一、巷道基础概念

### 1. 什么是巷道？

巷道是 ProfControl 自动识别的一组**无岔路的连续路径**。

| 类型 | 说明 |
|------|------|
| 孤巷道 | 两端都只有1条路径连接 |
| 单通道巷道 | 一端1条路径（入口），一端≥3条（出口） |
| 双通道巷道 | 两端都≥3条路径 |

### 2. 巷道结构（以实际地图为例）

- 每条垂直的站点列被横穿通道分割成上下两个巷道
- **下巷道**：从底部最深处 → 到横穿通道口，例：`3 → 4 → 23 → 24 → 6`
- **上巷道**：从横穿通道口 → 到顶部最深处，例：`8 → 25 → 26 → 27 → 28`

### 3. 巷道口与巷底

- **巷道口**：连接横穿通道的站点（如 `6`、`8`），通常有 3+ 条路径
- **巷底**：最深处，死胡同站点（如 `3`、`28`），只有 1 条路径

> 口诀：`OrderStations` 的**第一个** = 最深（巷底），**最后一个** = 巷口

### 4. 获取巷道信息

```csharp
// 从站点获取所属巷道
var pocketLines = station.BelongPocketLines;

// 遍历巷道中的站点（从里到外）
foreach (var sta in pocketLine.OrderStations) { }

// 逆向遍历（从外到里）
foreach (var sta in pocketLine.OrderStations.Reverse()) { }

// 判断巷道是否包含取货点
bool isPickupPocket = currentPocketLine.OrderStations.Any(x => x.GetValue<bool>("取货点"));

// 判断巷道是否被占用
if (巷道占用表.ContainsKey(currentPocketLine)) { }
```

---

## 二、OrderStations 与 Reverse() 详解

### 1. OrderStations（从里到外）

- **顺序**：从巷底（最深）→ 巷口
- **示例**：`[3, 4, 23, 24, 6]`
- **使用场景**：从最深处开始放货、清点库存、避免堵住门口

### 2. Reverse()（从外到里）

- **顺序**：从巷口 → 巷底（最深）
- **示例**：`[6, 24, 23, 4, 3]`
- **使用场景**：从门口开始取货、找空闲站点、提高效率

### 3. 代码示例

```csharp
// 放货：从里往外放（OrderStations）
foreach (var sta in pocketLine.OrderStations)
{
    if (sta.GetValue<bool>("取货点") && sta.IsEmpty())
    {
        // 放货！先放最里面
        break;
    }
}

// 取货：从外往里找（Reverse）
foreach (var sta in pocketLine.OrderStations.Reverse())
{
    if (sta.GetValue<bool>("取货点") && sta.HaveCargo("货物_1") && sta.Tag == null)
    {
        targetStation = sta;  // 优先取门口的货
        break;
    }
}
```

> **一句话结论**：**取货用 Reverse()，放货/清仓用 OrderStations**

---

## 三、站点状态判断（四个核心方法）

| 方法 | 含义 | 使用场景 |
|------|------|----------|
| `IsEmpty()` | 站点是否有货物 | 找空位放货 |
| `AgvOn()` | 是否有 AGV 停着 | 检查是否有车挡路 |
| `AgvComing()` | 是否有 AGV 正在驶来 | 路线规划避让 |
| `Tag != null` | 是否被任务锁定 | 防止任务冲突 |

### 组合判断示例

```csharp
// 检查站点是否"不可用"
if (!s.IsEmpty() || s.AgvComing() || s.Tag != null || s.AgvOn())
{
    hasObstacle = true;  // 这条路被堵住了
    break;
}
```

---

## 四、TaskInfo 任务信息类

### 完整定义

```csharp
public class TaskInfo
{
    public IAgv AgvName { get; set; }           // 执行任务的AGV
    public IStation PickStation { get; set; }   // 取货站点
    public IStation DropStation { get; set; }   // 送货站点
    public IPocketLines PocketLine { get; set; } // 所在巷道
    public bool IsMovingObstacle { get; set; }   // 是否正在移动障碍物

    public DateTime BlockStartTime { get; set; } = DateTime.MinValue;
    public DateTime TaskStartTime { get; set; } = DateTime.Now;
    public int RetryCount { get; set; } = 0;
    public int DropRetryCount { get; set; } = 0;

    public TaskInfo(IAgv agv, IStation PickStation, IPocketLines pocketLine = null)
    {
        this.AgvName = agv;
        this.PickStation = PickStation;
        this.PocketLine = pocketLine;
        this.IsMovingObstacle = false;
        agv.Tag = this;
    }
}
```

### 使用方式

```csharp
// 创建任务并绑定
var taskInfo = new TaskInfo(agv, pickStation, pocketLine);
agv.Tag = taskInfo;
pickStation.Tag = taskInfo;

// 读取任务信息
var task = agv.Tag as TaskInfo;
if (task != null && task.IsMovingObstacle)
{
    // 正在搬运障碍物
}
```

---

## 五、AGV 导航与取消导航

### 核心 API

| API | 说明 |
|-----|------|
| `agv.GoTo(targetMark, callback)` | 导航到目标 |
| `agv.KillAllRoute()` | 清除所有导航路径 |
| `agv.CanRoute(mark)` | 能否导航到目标 |

### 取消导航并重置状态

```csharp
private void CancelAgvNavigation(IAgv agv, string resetStatus = "空闲")
{
    agv.KillAllRoute();
    agv.Tag = null;
    agvMovingStatus[agv] = false;
    agv.SetValue("小车状态", resetStatus);
    
    var key = 巷道占用表.FirstOrDefault(x => x.Value == agv).Key;
    if (key != null) 巷道占用表.Remove(key);
}
```

### 核心模式：GoTo + 回调链

```csharp
agv.GoTo(targetMark, (finished) =>
{
    if (finished)
    {
        // 取货/放货操作
        agv.GoTo(nextMark, (finished2) => { });
    }
});
```

---

## 六、交通管制（交管）判断与处理

### 核心判断属性

| API | 说明 |
|-----|------|
| `agv.waiting` | 是否正在被交通管制阻塞 |
| `agv.CurrentBlockReason.ToString()` | 阻塞原因中文描述 |
| `agv.CurrentBlockReason.Suppose` | 阻塞原因枚举 |
| `agv.CancelRouteOverTime` | 阻塞超时重导航阈值（秒） |

### 阻塞原因枚举

| 枚举值 | 说明 |
|--------|------|
| None | 无阻塞 |
| ByAgv | 被空闲 AGV 阻挡 |
| ByBlockAgv | 被阻塞 AGV 阻挡 |
| ByAreaLock | 区域干涉 |
| ByUsedPocket | 巷道保护 |
| ManualTraffic | 手动交管 |

### 阻塞检测与超时处理

```csharp
if (agv.waiting)
{
    if (taskInfo.BlockStartTime == DateTime.MinValue)
    {
        taskInfo.BlockStartTime = DateTime.Now;
    }
    else if ((DateTime.Now - taskInfo.BlockStartTime).TotalSeconds > 30)
    {
        // 超时，切换策略
        switch (agv.GetValue<string>("小车状态"))
        {
            case "去取货": // 更换取货点
            case "去送货": // 更换放货点
            case "移动障碍物": // 取消任务
        }
    }
}
else
{
    taskInfo.BlockStartTime = DateTime.MinValue;
}
```

### 手动交管

```csharp
agv.HoldOn(station, "等待放行");  // 强制等待
agv.ReleaseHold();                // 恢复
```

---

## 七、障碍物搬运与巷道锁机制

### 问题根因

`MoveObstacle()` 没有锁定巷道，多个 AGV 可能同时操作同一巷道

### 解决方案：巷道级别互斥锁

```csharp
// 修复1：检查巷道是否已被占用
if (巷道占用表.ContainsKey(currentPocketLine))
{
    return;  // 已被占用，跳过
}

// 修复2：锁定巷道
巷道占用表[currentPocketLine] = agv;

// 修复3：释放巷道
var key = 巷道占用表.FirstOrDefault(x => x.Value == agv).Key;
if (key != null) 巷道占用表.Remove(key);
```

### 效果

同一时刻，同一巷道只允许一个 AGV 操作（取货、放货、搬障碍物）

---

## 八、等待队列机制

### 数据结构

```csharp
Dictionary<IPocketLines, Queue<TaskInfo>> 巷道等待队列 = new();
```

### 统一释放方法

```csharp
private void ReleasePocketLine(IPocketLines pocketLine, IAgv agv)
{
    // 移除占用
    巷道占用表.Remove(pocketLine);
    
    // 检查等待队列
    if (巷道等待队列.ContainsKey(pocketLine) && 巷道等待队列[pocketLine].Count > 0)
    {
        var nextTask = 巷道等待队列[pocketLine].Dequeue();
        var nextAgv = nextTask.AgvName;
        
        巷道占用表[pocketLine] = nextAgv;
        nextAgv.GoTo(nextTask.PickStation.Mark, reason =>
        {
            // 执行取货任务
        });
    }
}
```

---

## 九、找空位逻辑的 Bug 修复

### 问题

最深不通时，整个巷道被跳过，浪费门口空位

### 修复

```csharp
// 修复前（有Bug）
if (hasObstacle) break;

// 修复后（正确）
if (hasObstacle) continue;  // 继续检查下一个更靠近门口的站点
```

---

## 十、常用 API 速查表

| 分类 | API | 用途 |
|------|-----|------|
| 遍历 | `API.Agvs` | 遍历所有 AGV |
| 遍历 | `API.Stations` | 遍历所有站点 |
| 导航 | `agv.GoTo()` | 导航到目标 |
| 导航 | `agv.KillAllRoute()` | 取消导航 |
| 货物 | `sta.IsEmpty()` | 是否空 |
| 货物 | `sta.HaveCargo()` | 是否有货物 |
| 属性 | `sta.GetValue<T>()` | 读属性 |
| 属性 | `sta.SetValue()` | 写属性 |
| 定时 | `API.SetInterval()` | 定时执行 |
| 传参 | `agv.Tag` | 绑定任务数据 |
| 交管 | `agv.waiting` | 是否被阻塞 |

---

## 十一、总结速查表

| 概念 | 含义 |
|------|------|
| 巷道 | 从死胡同到横穿通道的一列站点 |
| OrderStations | 从最深到门口 |
| Reverse() | 从门口到最深 |
| 巷道口 | 连接主路的站点 |
| IsEmpty() | 站点是否为空 |
| AgvOn() | 是否有 AGV 停着 |
| AgvComing() | 是否有 AGV 正驶来 |
| Tag | 是否被任务锁定 |
| waiting | 是否被交管阻塞 |
| TaskInfo | 任务信息容器 |

---

## 十二、实际项目完整流程

```
1. 绘制地图 → 站点 + 路径 + AGV
2. 定义属性 → QuickGProperty 注册属性
3. 定时调度 → API.SetInterval 每2秒执行
4. 遍历巷道 → 检查占用、找货、处理障碍
5. 锁定巷道 → 防止多车同时进入
6. 派车取货 → 导航、取货、送货
7. 释放巷道 → 通知等待队列
```

> **核心要点**：学透 **巷道 + GoTo回调 + 货物系统 + Tag传参 + QuickGProperty** 这五个核心概念
```

直接选中上面的代码块，复制粘贴到你的 Markdown 编辑器中即可。