---
layout: post
title: ".NET Dump 分析：Task.WhenAll 一直未完成的根因排查"
date: 2026-04-14
tags: [.NET, 调试, dotnet-dump, 正则表达式, 性能]
toc: true
comments: true
author: zhangj
---

## 引言

本文记录一次真实生产环境的 Dump 排查过程，使用Codex 定位到 `Task.WhenAll` 卡死的根因——**正则表达式的灾难性回溯（Catastrophic Backtracking）**。

## 问题背景

程序运行时创建多个 `Task`，使用 `Task.WhenAll(...)` 等待，但程序一直未完成。通过抓取 Dump 文件，使用 `dotnet-dump` 工具进行离线分析。

---

## 分析过程

### 第一步：一次性抓取总体状态

使用 `clrthreads`、`threadpool`、`syncblk`、`dumpexceptions` 等命令获取全局视图：

```powershell
dotnet-dump analyze .\Demo.DMP `
  -c "eeversion" `
  -c "clrthreads" `
  -c "threadpool" `
  -c "threadpoolqueue" `
  -c "dumpasync --stats" `
  -c "parallelstacks" `
  -c "syncblk" `
  -c "dumpexceptions" `
  -c "exit"
```

**关键输出：**

```
ThreadCount:      21
UnstartedThread:  0
BackgroundThread: 17
PendingThread:    0
DeadThread:       3

CPU utilization:  13%
Workers Total:    3
Workers Running:  0
Workers Idle:     3
Worker Min Limit: 8
Worker Max Limit: 32767
```

线程池状态正常，Worker 空闲数量充足，说明**不是线程池饥饿**。

从 `parallelstacks` 发现有一条 async 链路卡住：

```
  ~~~~ 196c
     1 System.Threading.SemaphoreSlim.WaitUntilCountOrTimeout(...)
     1 System.Collections.Concurrent.BlockingCollection<...>.TryTakeWithNoTimeValidation(...)
     1 Microsoft.Extensions.Logging.Console.ConsoleLoggerProcessor.ProcessLogQueue()
```

这不是业务线程。真正的业务线程在 196c（OS Thread Id: 0x196c），栈顶是：

```
System.Text.RegularExpressions.RegexInterpreter.Go()
System.Text.RegularExpressions.RegexRunner.Scan(...)
System.Text.RegularExpressions.Regex.Match(...)
```

**初步结论：线程不在等待锁或 I/O，而是在执行正则匹配。**

---

### 第二步：定位 Task.WhenAll 的等待对象

使用 `dumpheap` 找到 `WhenAllPromise` 对象：

```powershell
dotnet-dump analyze .\Demo.DMP `
  -c "dumpheap -type System.Threading.Tasks.Task+WhenAllPromise" `
  -c "exit"
```

```
         Address               MT           Size
    0003403251d0     07fe42aac988             80

Statistics:
          MT Count TotalSize Class Name
07fe42aac988     1        80 System.Threading.Tasks.Task+WhenAllPromise
Total 1 objects, 80 bytes
```

检查该 `WhenAllPromise` 的状态：

```powershell
dotnet-dump analyze .\Demo.DMP `
  -c "dumpobj 0003403251d0" `
  -c "taskstate 0003403251d0" `
  -c "gcroot 0003403251d0" `
  -c "exit"
```

**关键输出：**

```
000007fe427ba100  40008b4       38 ...ding.Tasks.Task[]  0 instance 0000000340325028 m_tasks
000007fe40a1b1f0  40008b5       40         System.Int32  1 instance                1 m_count
WaitingForActivation
```

- `m_count = 1`：总共等 50 个子任务，还剩 **1 个未完成**
- 状态为 `WaitingForActivation`，说明这个 Task 还在等待

从 GC Root 追踪引用链：

```
Thread 196c:
    12d9c300 7fe425638b2 ...<<StartAutoWsAkAsync>b__76>d.MoveNext()
        -> 00024079f388     AsyncStateMachineBox
        -> 0003c0b86dd0     UnwrapPromise
        -> 0003403251d0     WhenAllPromise
```

这证明 `b__76` 分支的 Task 正是未完成的那一个。

---

### 第三步：定位具体执行点

切到该线程查看托管栈：

```powershell
dotnet-dump analyze .\Demo.DMP `
  -c "setthread 27" `
  -c "clrstack -a" `
  -c "exit"
```

**栈顶关键帧：**

```
0000000012D9B7F0 000007FE425197C0 System.Text.RegularExpressions.RegexInterpreter.Go()
0000000012D9B9D0 000007FE424C6D7C System.Text.RegularExpressions.RegexRunner.Scan(...)
0000000012D9BA30 000007FE424C6A56 System.Text.RegularExpressions.Regex.Run(...)
0000000012D9BAE0 000007FE4253D340 System.Text.RegularExpressions.Regex.Match(...)
0000000012D9BB30 000007FE430D3F7C JYGZService+<>c__DisplayClass17_0.<Ws_Sj_Cl_List>b__0(...)
```

调用链路：

```
StartAutoWsAkAsync -> b__76 (分支)
  -> WSJYRegex
    -> Ws_Sj_Cl_List
      -> Regex.Match (卡住点)
```

---

### 第四步：读取正则运行对象

```powershell
dotnet-dump analyze .\Demo.DMP `
  -c "dumpobj 000000014067d428" `
  -c "dumpobj 00000001406d1570" `
  -c "exit"
```

**关键字段：**

```
System.Text.RegularExpressions.RegexInterpreter:
    runtext     = 00000003c0bda6f8  (实际匹配文本)
    runtextbeg = 0                  (文本起始)
    runtextend = 1200               (文本结束，约 1200 字符)
    runtextpos = 524                (当前位置)
    _timeout   = -1                  (无超时!)
    _ignoreTimeout = true           (忽略超时)
    _operator  = 151
    _codepos   = 24
```

**致命问题发现：**

- `_timeout = -1` 表示**没有设置超时**
- `_ignoreTimeout = true` 表示**忽略超时检查**
- 匹配文本约 1200 字符
- 正则正在第 524 个字符位置执行，已消耗大量时间

---

### 第五步：提取正则表达式

pattern 对象地址为 `00000001406d1484`，通过 `dw` 命令导出 UTF-16 字符：

```powershell
dotnet-dump analyze .\Demo.DMP `
  -c "dw 00000001406d1484 -c 112 --show-address" `
  -c "exit"
```

---

## 根因总结

### 直接原因

- `Task.WhenAll` 等 50 个子任务，剩 **1 个未完成**
- 该任务卡在 `StartAutoWsAkAsync -> WSJYRegex -> Ws_Sj_Cl_List` 链路内
- 执行点在 `Regex.Match` 调用

### 深层原因

| 问题       | 说明                                               |
| ---------- | -------------------------------------------------- |
| 正则结构   | 嵌套量词 `(?:...*?...*?)*?` + 前瞻断言 `(?!)` 组合 |
| 回溯风险   | 文本约 1200 字符，中文标点复杂，触发指数级回溯     |
| 无超时保护 | `_timeout = -1`，正则执行无法被及时中断            |

### 最终判定

**这是正则表达式灾难性回溯（Catastrophic Backtracking）导致的长时间执行**，表现为 `Task.WhenAll` 一直不返回。

---

## 修复建议

1. **必须项**：所有业务正则增加超时（使用带超时的 `Regex.Match` 重载）
2. **正则优化**：重写该规则，避免"宽泛重复 + 可选 + 前瞻"的危险组合
3. **单元测试**：对可疑规则加入长文本、边界文本、反例文本的防回归测试

**推荐的超时写法：**

```csharp
// 推荐：为正则设置超时时间
var regex = new Regex(pattern, RegexOptions.None, TimeSpan.FromSeconds(1));
var match = regex.Match(input);

// 避免：无超时限制
var regex2 = new Regex(pattern, RegexOptions.None, TimeSpan.FromMilliseconds(-1));
```

---

## 关键输出汇总

| 命令                            | 关键发现                                 |
| ------------------------------- | ---------------------------------------- |
| `clrthreads`                    | 21 线程，3 个 Worker 空闲                |
| `dumpheap -type WhenAllPromise` | WhenAll 地址 `0003403251d0`，`m_count=1` |
| `dumpobj WhenAllPromise`        | `m_tasks` 数组含 50 个任务               |
| `gcroot WhenAllPromise`         | 4 个根，指向 `b__76` 的 async 状态机     |
| `setthread 27; clrstack`        | 栈顶 `RegexInterpreter.Go()`             |
| `dumpobj RegexInterpreter`      | `_timeout=-1`，`_ignoreTimeout=true`     |
| `dw pattern`                    | 还原正则表达式文本                       |

---
