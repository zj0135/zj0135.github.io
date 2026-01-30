---
layout: post
title: "平滑加权轮询负载均衡"
date: 2026-01-30
tags: [平滑, 轮询, 负载均衡, C#]
toc: true
comments: true
author: zhangj
---

## 引言

**平滑加权轮询算法**，在严格遵循权重比例的前提下，实现请求分配的时间平滑性，避免高权重服务器被连续命中导致瞬时压力集中。

## 适用场景

| 场景                 | 适用性     | 说明                                       |
| -------------------- | ---------- | ------------------------------------------ |
| **异构服务器集群**   | ⭐⭐⭐⭐⭐ | 不同配置服务器（如 8核/4核）按能力分配流量 |
| **需要严格权重比例** | ⭐⭐⭐⭐⭐ | 金丝雀发布、灰度流量控制（如 95%:5%）      |
| **避免瞬时压力集中** | ⭐⭐⭐⭐   | 比朴素轮询更平滑，保护高权重节点           |
| **结合健康检查**     | ⭐⭐⭐⭐   | 故障时自动降权，恢复后平滑回升             |
| **会话无状态服务**   | ⭐⭐⭐⭐⭐ | Web API、微服务网关等                      |

## 与朴素加权轮询对比

| 特性     | 朴素加权轮询              | 平滑加权轮询                |
| -------- | ------------------------- | --------------------------- |
| 实现方式 | 生成 `[A,A,A,B]` 列表循环 | 动态计算三变量状态          |
| 流量分布 | 瞬时集中（连续命中A）     | 时间维度平滑分散            |
| 权重调整 | 需重建列表                | 动态修改 `effective_weight` |

## 核心原理

### 三变量状态机

每个后端服务器维护三个关键变量：

- **weight** 配置的静态权重
- **effective_weight** 当前有效权重（支持动态调整）
- **current_weight** 动态计算权重（算法核心）

### 分配流程（每次请求）

1. **累加**：`current_weight += effective_weight`（遍历所有可用服务器）
2. **选择**：选出 `current_weight` **最大**的服务器（相等时取配置顺序靠前者）
3. **归零**：`selected.current_weight -= total_effective_weight`（所有可用服务器 effective_weight 之和）
4. **转发**：请求发送至选中服务器

### 为什么“平滑”？

- **避免连续命中**：通过“累加+归零”机制，高权重服务器不会被连续选中（除非权重极高）
- **严格比例**：长期统计下，分配次数 ≈ 权重比例（如权重 5:1 → 5/6 : 1/6）
- **动态适应**：结合健康检查可动态调整 `effective_weight`，实现故障转移

## 代码实现

```csharp
public class ServiceCustom
    {
        public string Name { get; set; }
        public int Weight { get; set; }          // 配置权重（静态）
        public int EffectiveWeight { get; set; } // 当前有效权重（可动态调整）
        public int CurrentWeight { get; set; }   // 动态计算权重

        public ServiceCustom(string name, int weight)
        {
            Name = name;
            Weight = weight;
            EffectiveWeight = weight;
            CurrentWeight = 0;
        }

        /// <summary>
        /// 模拟服务器故障：降低有效权重
        /// </summary>
        public void MarkAsFailed()
        {
            EffectiveWeight = Math.Max(1, EffectiveWeight / 2); // 避免归零
            CurrentWeight = 0; // 重置当前权重
        }

        /// <summary>
        /// 模拟服务器恢复：重置有效权重
        /// </summary>
        public void Recover()
        {
            EffectiveWeight = Weight;
            CurrentWeight = 0;
        }

        public void Invoke()
        {
            Console.WriteLine($"服务器：{Name}");
        }
    }
    public class SmoothWeightedRoundRobin
    {
        public readonly List<ServiceCustom> _services = new List<ServiceCustom>
        {
            new ServiceCustom("ServiceA",5),
            new ServiceCustom("ServiceB",2),
            new ServiceCustom("ServiceC",1)
        };

        private readonly object _lock = new object();

        public ServiceCustom SelectServer()
        {
            lock (_lock)
            {
                // 过滤掉有效权重 <=0 的服务器（模拟不可用状态）
                var availableServers = _services
                    .Where(s => s.EffectiveWeight > 0)
                    .ToList();

                if (availableServers.Count == 0)
                    throw new InvalidOperationException("error");

                var totalEffectiveWeight = 0;
                ServiceCustom selected = null;
                var maxCurrentWeight = int.MinValue;

                foreach (var server in availableServers)
                {
                    server.CurrentWeight += server.EffectiveWeight;
                    totalEffectiveWeight += server.EffectiveWeight;

                    // 步骤2: 选择 current_weight 最大的服务器（相等时取第一个）
                    if (server.CurrentWeight > maxCurrentWeight)
                    {
                        maxCurrentWeight = server.CurrentWeight;
                        selected = server;
                    }
                }

                // 步骤3: 归零操作
                if (selected != null)
                {
                    selected.CurrentWeight -= totalEffectiveWeight;
                    return selected;
                }

                throw new InvalidOperationException("error");
            }
        }
    }
```

| 服务器     | 配置权重 | 被选中次数 | 实际比例 |
| ---------- | -------- | ---------- | -------- |
| `ServiceA` | 5        | 15         | 62.5%    |
| `ServiceB` | 2        | 6          | 25.0%    |
| `ServiceC` | 1        | 3          | 12.5%    |


