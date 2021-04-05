---
title: 图的5种常用的最短路算法
author: chukeey
date: 2021-03-21 21:00:00 +0800
categories: [技术, 数据结构与算法]
tags: [算法, 图]
pin: false
---

### 1. 简介
图的5种常用的最短路算法为：朴素Dijkstra、 堆优化Dijkstra、Bellman-Ford、SPFA、 Floyd。
以下为这5种算法的应用场景以及时间复杂度：
- 多源汇最短路（任意两点间的最短路）
    - Floyd  O(n^3)
- 单源最短路
    - 无负边权（所有边长都为正数）
        - 稠密图： 朴素Dijkstra  O(n^2)
        - 稀疏图： 堆优化Dijkstra  O(mlogn)
    - 有负边权
        - SPFA  一般O(m)，最坏O(nm)
        - 有边数限制：Bellman-Ford  O(nm)

### 2. 算法实现：
#### 2.1 朴素Dijkstra
**算法思想：**
> 朴素DijKstra的思想是通过一个距离起点A最近的点B，缩短起点A通过点B到达其他点的距离，因为只有通过距离起点最近的点，才有可能缩短起点到达其他点的距离。
    所以一共分两步：
    （1）找到距离起点最近的点。
    （2）通过该点缩短起点到达其他点的距离。

**代码实现：**
```php
function dijkstra($n, $edges)
 {
     $dist = [];
     $state = [];

     // 绘图
     $graph = [];
     for ($i = 1; $i <= $n; $i++) {
         $state[$i] = false;
         $dist[$i] = PHP_INT_MAX;
         for ($j = 1; $j <= $n; $j++) {
             $graph[$i][$j] = PHP_INT_MAX;
         }
     }
     foreach ($edges as [$u, $v, $w]) {
         $graph[$u][$v] = $graph[$v][$u] = $w;
     }

     // 朴素迪杰斯特拉算法
     $dist[1] = 0;
     for ($i = 1; $i <= $n; $i++) { // 循环n次， 每次找出一个离起点最近的点。
         $t = -1;
         for ($j = 1; $j <= $n; $j++) {
             if(!$state[$j] && ($t == -1 || $dist[$j] < $dist[$t])) {
                 $t = $j;
             }
         }
         $state[$t] = true;
         for ($j = 1; $j <= $n; $j++) {
             if($dist[$j] > $dist[$t] + $graph[$t][$j]) {
                 $dist[$j] = $dist[$t] + $graph[$t][$j];
             }
         }
     }

     return $dist[$n] >= PHP_INT_MAX ? -1 : $dist[$n];
 }
```

#### 2.2 堆优化Dijkstra
 **算法思想：**
 > 堆优化其实就是使用优先队列对图BFS一遍，在搜索的同时更新点的距离。朴素算法需要遍历所有点找出距离最近的点，使用优先队列使这一步的时间复杂度从N变为了logN。
   对于走迷宫问题，我们很容易可以使用BFS找出最短路，因为每个点与点之间的距离是一样的，但是当边权不同时，显然我们必须先遍历近距离的点，这时我们就可以使用优先队列来达到这一目的。

**代码实现：**
```php
function heapDijkstra($n, $edges)
{
    // 绘图
    $graph = [];
    foreach ($edges as [$u, $v, $w]) {
        $graph[$u][$v] = $graph[$v][$u] = $w;
    }
    $dist = array_fill(1, $n, PHP_INT_MAX);
    $dist[1] = 0;

    // 堆优化迪杰斯特拉算法
    $priorityQueue = new SplPriorityQueue;
    $priorityQueue->setExtractFlags(SplPriorityQueue::EXTR_BOTH);
    $priorityQueue->insert(1, 0);
    while ($priorityQueue->count()) {
        $ins = $priorityQueue->extract();
        $data = $ins['data'];
        $priority = -$ins['priority']; // SplPriorityQueue 是大根堆所以加负号
        foreach ($graph[$data] as $node => $w) {
            if ($w + $priority < $dist[$node]) {
                $dist[$node] = $w + $priority;
                $priorityQueue->insert($node, -$dist[$node]);
            }
        }
    }

    return $dist[$n] >= PHP_INT_MAX ? -1 : $dist[$n];
}
```

#### 2.3 Bellman-Ford
 **算法思想：**
 > Bellman-Ford算法是先将每个点到达起点的距离设为正无穷，然后直接暴力枚举所有边，如果一个点的距离可以被更新，就更新该点的距离。
   枚举一次至少可以得到一个点到达起点的距离，直接无脑枚举n次，就可以得到所有点到起点的最短距离。

**代码实现：**
```php
function bellmanFord($n, $edges)
{
    $dist = array_fill(1, $n, PHP_INT_MAX);
    $dist[1] = 0;

    $m = count($edges);
    for ($i = 1; $i <= $n; $i++) {
        for ($j = 0; $j < $m; $j++) {
            $u = $edges[$j][0];
            $v = $edges[$j][1];
            $w = $edges[$j][2];
            if ($dist[$u] + $w < $dist[$v]) {
                $dist[$v] = $dist[$u] + $w;
            }
        }
    }

    return $dist[$n] >= PHP_INT_MAX ? -1 : $dist[$n];
}
```

#### 2.4 SPFA（(Shortest Path Faster Algorithm)）
 **算法思想：**
 > 我们在一次循环中做了很多不必要的操作，如果我们可以只更新那些前驱结点被更新过的点，就可以大大提升效率。因为只有前驱结点到达起点的距离减小了，它的后继结点到达起点的距离才能减小。
   所以我们用宽搜的思想，每次将被更新过的点入队，然后遍历队列中每个点的后继结点，重复更新操作和入队操作。
   只要不存在负环，起点到达任意点所经过的边数一定小于n，反之一定大于等于n因为只有n个点，不经过环的话最多经过n-1条边，只有在存在负环的情况下，SPFA才会经过负环，经过负环后，到达其他点边数就会大于等于n。

**代码实现：**
```php
function SPFA($n, $edges)
{
    // 绘图
    $graph = [];
    foreach ($edges as [$u, $v, $w]) {
        $graph[$u][$v] = $graph[$v][$u] = $w;
    }
    $dist = array_fill(1, $n, PHP_INT_MAX);
    $dist[1] = 0;

    $visited = array_fill(1, $n, false);
    $visited[1] = true;
    $cnt[1] = 1;
    $finish = false;

    $queue = new SplQueue;
    $queue->enqueue(1);
    while (!$finish && $queue->count()) {
        $value = $queue->dequeue();
        $visited[$value] = false;
        foreach ($graph[$value] as $node => $w) {
            if ($w + $dist[$value] < $dist[$node]) {
                $dist[$node] = $w + $dist[$value];
                // 最多经过边数为$n-1
                $cnt[$node] = $cnt[$value] + 1;
                if ($cnt[$node] > $n) {
                    $finish = true;
                    break;
                }
                if (!$visited[$node]) {
                    $queue->enqueue($node);
                    $visited[$node] = true;
                }
            }
        }
    }

    return $dist[$n] >= PHP_INT_MAX ? -1 : $dist[$n];
}
```

#### 2.5 Floyd
 **算法思想：**
 > 对于朴素Dijkstra算法来说，起点通过距离它最近的点去缩短其到达其他点的距离，Floyd算法本质上也是这样的思想。
  （1）通过一个点，遍历所有点对，更新距离。
  （2）通过n个点，遍历所有点对n次，更新距离。

**代码实现：**
```php
function floyd($n, $edges)
{
    $dist = [];
    for ($i = 1; $i <= $n; $i++) {
        for ($j = 1; $j <= $n; $j++) {
            $dist[$i][$j] = PHP_INT_MAX;
        }
        $dist[$i][$i] = 0;
    }
    foreach ($edges as [$u, $v, $w]) {
        $dist[$u][$v] = $dist[$v][$u] = $w;
    }

    for ($i = 1; $i <= $n; $i++) {
        for ($j = 1; $j <= $n; $j++) {
            for ($k = 1; $k <= $n; $k++) {
                $dist[$i][$j] = min($dist[$i][$j], $dist[$i][$k] + $dist[$k][$j]);
            }
        }
    }

    return $dist[1][$n] >= PHP_INT_MAX ? -1 : $dist[1][$n];
}
```
