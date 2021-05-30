---
title: 十大排序算法
author: chukeey
date: 2021-05-30 22:55:00 +0800
categories: [技术, 数据结构与算法]
tags: [算法, 排序, go]
pin: false
---

### 1. 前言
排序算法是最经典的算法知识，目前共有十种排序算法：冒泡排序、选择排序、插入排序、归并排序、快速排序、希尔排序、堆排序、计数排序、桶排序、基数排序。

最近刚好在学习go语言，因此想着用go语言实现这些排序算法，纯当练练手，顺便回顾下排序算法~

### 2. 算法实现
#### 2.1 冒泡排序
**算法思想：**
> 算法过程：
>
> 1. 在未排序序列中比较相邻的元素，如果第一个比第二个小/大，就交换它们两个，将最小/大值冒泡至最右侧
> 2. 从剩余未排序元素中继续过程1
> 3. 重复过程2，直到所有元素均排序完毕

**代码实现：**

~~~go
// 冒泡排序：是稳定排序
func bubblingSort(data []int) {
	size := len(data)
	if size <= 1 {
		return
	}
    // 优化排序
	swap := false
	for i := size-1; i > 0; i--  {
		for j := 0; j < i; j++ {
			if data[j] > data[j+1] {
				data[j], data[j+1] = data[j+1], data[j]
				swap = true
			}
		}
		if swap == false {
			break
		}
	}
}
~~~

#### 2.2 选择排序

**算法思想：**

> 选择排序和冒泡排序相似，算法过程：
>
> 1. 在未排序序列中找到最小/大元素，存放到排序序列的起始位置
> 2. 从剩余未排序元素中继续寻找最小/大元素，然后放到已排序序列的末尾
> 3. 重复第二步，直到所有元素均排序完毕

**代码实现：**

```go
// 选择排序：是不稳定排序
func selectionSort(data []int) {
	size := len(data)
	if size <= 1 {
		return
	}
	for i := 0; i < size - 1; i++  {
		for j := i+1; j < size; j++ {
			if data[i] > data[j] {
				data[j], data[i] = data[i], data[j]
			}
		}
	}
}
```

#### 2.3 插入排序

**算法思想：**

> 算法过程：
>
> 1. 把待排序序列分成已排序和未排序两部分，初始的时候把第一个元素认为是已排好序的
> 2. 从第二个元素开始，在已排好序的子数组中寻找到该元素合适的位置并插入该位置
> 3. 重复过程2直到最后一个元素被插入已排序数组中

**代码实现：**

```go
// 插入排序：是稳定排序
func insertSort(data []int) {
	size := len(data)
	if size <= 1 {
		return
	}
	for i := 1; i < size; i++  {
		base := data[i]
		j := i - 1
		for ; j >= 0 && data[j] > base; j-- {
			data[j] = data[j-1]
		}
		data[j] = base
	}
}
```



