---
layout: post
title: "C#中集合swap remove"
date: 2026-03-24           
tags: [性能, C#]
toc: true
comments: true
author: zhangj
---

在不关注list集合元素顺序时，删除集合内指定索引下标时，将复杂度从O(n) 降至 O(1) 

平时要删除指定下标的元素时可能会直接这样做，但当元素位于集合靠前的位置时，删除后所有元素都需要向前移动一位。

```plain
 var list = Enumerable.Range(1, 100).Select(x => Guid.NewGuid().ToString()).ToList();
 var index = 50;
 list.RemoveAt(index);
```

我们可以将集合中最后一个元素的值赋给当前需要删除的下标位置，然后在删除最后一个元素。

```plain
var list = Enumerable.Range(1, 100).Select(x => Guid.NewGuid().ToString()).ToList();
var index = 50;
list[index] = list[^1];
list.RemoveAt(list.Count - 1);
```

需要注意并不是所有场景都可以使用，必须要确定不关注集合内元素顺序。

[Remove O(N) List.RemoveAt from RegexCache.Add by stephentoub · Pull Request #106581 · dotnet/runtime (github.com)](https://github.com/dotnet/runtime/pull/106581/files/28f434c3f3415181901a803985d05b19b0d18107)

