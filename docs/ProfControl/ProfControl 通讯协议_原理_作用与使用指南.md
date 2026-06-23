# ProfControl 通讯协议：原理、作用与使用指南

# ProfControl 通讯协议：原理、作用与使用指南

## 一、什么是通讯协议？

> **形象比喻**：语言就是协议。中国人和中国人能沟通是因为都懂中文；中国人和美国人如果各自只懂自己的语言，就无法交流。如果美国人也懂中文，就可以交流了。

在 ProfControl 中，**通讯协议脚本** 是一套专门用来对接 AGV、物流设备、厂内复杂设备的“翻译官”程序。

---

## 二、通讯协议的原理

### 2.1 数据流的两个方向

- **下发命令**：ProfControl → 车体（路径、动作等）
- **上传状态**：车体 → ProfControl（电量、位置、速度等）

### 2.2 事件驱动机制（核心原理）

ProfControl **不是轮询**问“车该走了没”，而是采用**事件驱动**机制：

- 系统条件满足（如路径规划完成）→ 自动触发事件
- 通讯脚本预先注册事件 → 事件触发时执行对应函数

---

## 三、通讯协议的作用

| 作用 | 说明 |
|------|------|
| 桥接异构系统 | 将 ProfControl 的“内部语言”翻译成车体 PLC 能理解的“字节报文” |
| 适配不同设备 | 不同品牌 AGV（海康、极智嘉、快仓等）协议不同，通过不同脚本来适配 |
| 组装/解析报文 | 将路径、动作等结构化数据打包成字节流发送，反之解析收到的字节流 |
| 实现双向通讯 | 下发命令，接收状态（电量、位置） |
| 交通管制联动 | 与 ProfControl 的路径规划和交通管制联动，完成智能调度 |

---

## 四、通讯协议脚本的结构

### 4.1 协议主体脚本（示例）

```csharp
namespace ProfControl
{
    public partial class CommuTest  // partial，和下面的类是一个类
    {
        public void InitVirtual()
        {
            var mapAgvVirtual = new EventMap<IAgvVirtual>();
            mapAgvVirtual.Register = (agvVirtual) =>
            {
                agvVirtual.AgvVirtualUpdateStatusEvent += AgvVirtual_UpdateStatusEvent;
                agvVirtual.AgvVirtualGetNewMarkEvent += AgvVirtual_GetNewMarkEvent;
            };
            mapAgvVirtual.Unregister = (agvVirtual) =>
            {
                agvVirtual.AgvVirtualUpdateStatusEvent -= AgvVirtual_UpdateStatusEvent;
                agvVirtual.AgvVirtualGetNewMarkEvent -= AgvVirtual_GetNewMarkEvent;
            };
            Maps.Add(mapAgvVirtual);
        }

        // 模拟车体上报位置/状态
        private void AgvVirtual_UpdateStatusEvent(object sender, AgvVirtualUpdateStatusEventArgs e)
        {
            var agvVirtual = sender as IAgvVirtual;
            // 模拟发送电量、速度等数据
        }

        // 模拟车体到达新地标
        private void AgvVirtual_GetNewMarkEvent(object sender, AgvVirtualGetNewMarkEventArgs e)
        {
            var agvVirtual = sender as IAgvVirtual;
            // 通知 ProfControl 车到达了某个站点
        }
    }
}
```

---

## 五、API 数据接收接口（上传数据时使用）

当车体发来数据，解析完后，调用以下 API 将数据“喂”给 ProfControl：

| API 调用 | 作用 | 使用场景 |
|----------|------|-----------|
| `API.BatteryRecv(agv, 85)` | 传递电池电量（0-100） | 车体上报电量时 |
| `API.VoltageRecv(agv, 48.5)` | 传递电池电压 | 车体上报电压时 |
| `API.SpeedRecv(agv, 1.2)` | 传递当前速度（m/s） | 车体上报速度时 |
| `API.LocationRecv(agv, x, y)` | 传递坐标位置 | 激光/二维码导航车 |
| `API.NewMarkPointRecv(agv, "A01")` | 传递到达的新地标 | 到达新站点时调用 |
| `API.DriveDistanceRecv(agv, dx, dy)` | 传递位移增量 | 磁导航车 |
| `API.AngleRecv(agv, 90.5)` | 传递车体角度（度） | 需要角度信息时 |
| `API.DoActionRecv(agv, actionId)` | 通知动作已完成 | 机械手/顶升完成反馈 |
| `API.WarningRecv(agv, "电池过低")` | 传递告警信息 | 车体异常时 |
| `API.CommQualityRecv(agv, 95)` | 传递通讯质量（0-100） | 监控通讯健康度 |

---

## 六、详细使用步骤（从零开始配置）

### 步骤 1：编写通讯协议脚本

- 打开 ProfControl → 切换到脚本页面 → 创建新项目（类型选“通讯协议”）
- 创建文件并粘贴模板代码
- 按车体协议编写组包/解包逻辑
- 编译

### 步骤 2：添加协议到系统中

- 设置 → 通讯协议配置 → 点击 `+` → 选择编译好的协议
- 文件路径：`CommScript\你的协议名\bin\Debug\你的协议.dll`

### 步骤 3：配置通讯单元

- 设置 → 通讯配置 → 点击 `+` → 填写以下内容：
  - IP 地址
  - 端口号（多台同类型设备端口必须不重复）

### 步骤 4：测试通讯

- 点击“发送测试”按钮，发送 `Hello world!` 测试字符串，确认链路是否通

### 步骤 5：运行调试

- 点击运行按钮，检查日志输出，确认数据收发正常

---

## 七、完整实际案例：对接一台 TCP AGV

### 场景描述

- AGV 通过 TCP 通讯
- ProfControl 规划路径后，将路径地标列表通过 TCP 下发
- AGV 每隔 1 秒上报一次状态（电量、速度、当前地标）

### 车体协议格式

#### 下发路径报文格式：

```
包头（0xAA 0x55）+ 命令字（0x01）+ 地标数量 + 地标1 + 地标2 + ... + CRC16
```

#### 上传状态报文格式：

```
包头（0x55 0xAA）+ 长度 + 电量 + 速度 + 当前地标 + CRC16
```

### 通讯脚本实现

#### 下发路径（组包）

```csharp
// 1. 提取地标列表
List<string> marks = new List<string>();
foreach (var route in routes)
{
    marks.Add(route.InStation.MarkPointValueMirror);
    marks.Add(route.OutStation.MarkPointValueMirror);
}

// 2. 组装报文
List<byte> sendBuffer = new List<byte>();
sendBuffer.Add(0xAA);
sendBuffer.Add(0x55);
sendBuffer.Add(0x01); // 命令字：路径下发
sendBuffer.Add((byte)marks.Count);

foreach (var mark in marks)
{
    var markBytes = Encoding.ASCII.GetBytes(mark.PadRight(4, ' '));
    sendBuffer.AddRange(markBytes);
}

// 计算 CRC
ushort crc = CommuHelper.CalcRC16(0, sendBuffer.ToArray(), sendBuffer.Count);
sendBuffer.AddRange(BitConverter.GetBytes(crc));

// 3. 通过 TCP 发送
if (devInfo.Connected == true)
{
    CommuData(devInfo.Session, sendBuffer.ToArray(), sendBuffer.Count);
    Tools.trace($"已向{agv.Name}下发路径：{string.Join("->", marks)}");
}
```

#### 接收解析（上传状态）

```csharp
public void RecvData<T>(ICommMedia commu, DevWithIpInfo ipInfo, SBuffer<T> _buffer)
{
    ArrayBuffer<byte> buffer = _buffer as ArrayBuffer<byte>;
    if (ipInfo.Dev is IAgv agv)
    {
        while (true)
        {
            if (buffer.Count < 4) break;

            // 查找包头 0x55 0xAA
            if (buffer[0] != 0x55 || buffer[1] != 0xAA)
            {
                buffer.Move(1);
                continue;
            }

            int length = buffer[2];
            if (buffer.Count < length + 4) break;

            // 解析数据
            byte battery = buffer[3];
            byte speed = buffer[4];
            string mark = Encoding.ASCII.GetString(buffer.Skip(5).Take(4).ToArray());

            // 调用 API 传递给 ProfControl
            API.BatteryRecv(agv, battery);
            API.SpeedRecv(agv, speed / 10.0);
            API.NewMarkPointRecv(agv, mark.Trim());

            // 移动缓存
            buffer.Move(length + 4);
            if (buffer.Empty) break;
        }
    }
}
```

---

## 八、通讯脚本中的可用事件汇总

| 事件名称 | 触发时机 | 典型用途 |
|----------|-----------|-----------|
| `GetNewRoutesEvent` | 路径规划完成 | 下发路径报文（最重要） |
| `EnterNewRouteEvent` | AGV 进入一段新路径 | 更新车体当前路段信息 |
| `DoActionEvent` | 需要执行机构动作 | 下发顶升/滚筒/夹取指令 |
| `RouteFinishedEvent` | 一段子路径走完 | 确认车体完成某段路线 |
| `TaskFinishedEvent` | 整个任务完成 | 触发任务完成逻辑 |
| `OnlineEvent` | 设备通讯建立 | 发送初始化/注册报文 |
| `OfflineEvent` | 设备通讯断开 | 告警/停止下发 |
| `DecisionRestrictionEvent` | 路径决策时 | 限制某些路径不让走 |

---

## 九、通讯方式支持一览

| 通讯方式 | ProfControl 角色 | 适用场景 |
|----------|------------------|-----------|
| TCP 服务器 | 作为 Server，等待车体连接 | 多数 AGV 场景 |
| TCP 客户端 | 作为 Client，主动连接车体 | 车体是 Server 时 |
| UDP | 收发 UDP 报文 | 实时性要求高、允许丢包 |
| HTTP | 提供/调用 REST API | 与 WMS/MES 对接 |
| ModBus TCP | Master/Slave | PLC 设备 |
| S7 读写 | 西门子 PLC | 西门子 PLC 场景 |

---

## 十、常见问题排查（FAQ）

| 问题 | 可能原因 | 解决方法 |
|------|----------|-----------|
| × 连接不上 | IP/端口错误 / 防火墙拦截 | 检查 IP 端口，关闭防火墙测试 |
| × 不下发路径 | 协议脚本未注册事件 | 检查 `map.Register` 是否绑定了 `GetNewRoutesEvent` |
| × 车不走 | 报文格式不对 | 用调试助手抓包对比协议文档 |
| × 不上报位置 | 未调用 `API.NewMarkPointRecv` | 检查 `RecvData` 中的解析逻辑 |
| × 编译失败 | 文件名和类名不一致 | 确保 `.cs` 文件名 = 类名 |
| × 启动后不生效 | 通讯配置未勾选“启用” | 勾选启用并保存 |

---

## 总结一句话

> **通讯协议脚本 = 翻译官 + 信使**  
> - 翻译：把 ProfControl 的路径/动作翻译成车体听得懂的字节流  
> - 传递：把车体的状态/电量/位置传递给 ProfControl  
> - 底层用 TCP/UDP/HTTP 做管道，上层用事件驱动做触发，中间用脚本代码做转换
