---
title: go语言版十大排序算法
author: chukeey
date: 2021-05-30 22:55:00 +0800
categories: [计算机技术, 数据结构与算法]
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

```go
// 冒泡排序：是稳定排序，复杂度：O(n2)，空间复杂度：O(1)
func bubblingSort(data []int) {
	size := len(data)
	if size <= 1 {
		return
	}
	swap := false
	for i := size - 1; i > 0; i-- {
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
```

#### 2.2 选择排序

**算法思想：**

> 选择排序和冒泡排序相似，算法过程：
>
> 1. 在未排序序列中找到最小/大元素，存放到排序序列的起始位置
> 2. 从剩余未排序元素中继续寻找最小/大元素，然后放到已排序序列的末尾
> 3. 重复第二步，直到所有元素均排序完毕

**代码实现：**

```go
// 选择排序：是不稳定排序，复杂度：O(n2)，空间复杂度：O(1)
func selectionSort(data []int) {
	size := len(data)
	if size <= 1 {
		return
	}
	for i := 0; i < size-1; i++ {
		for j := i + 1; j < size; j++ {
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
// 插入排序：是稳定排序，复杂度：O(n2)，空间复杂度：O(1)
func insertSort(data []int) {
	size := len(data)
	if size <= 1 {
		return
	}
	for i := 1; i < size; i++ {
		base := data[i]
		j := i - 1
		for ; j >= 0 && data[j] > base; j-- {
			data[j+1] = data[j]
		}
		data[j+1] = base
	}
}
```
#### 2.4 归并排序

**算法思想：**

> 算法过程：
>
> 1. 将序列每相邻两个数字进行归并操作，形成ceil(n/2)个序列
> 2. 若此时序列数不是1个则将上述序列再次归并，形成ceil(n/4)个序列
> 3. 重复步骤2，直到所有元素排序完毕，即序列数为1

**代码实现：**

```go
// 归并排序：是稳定排序，时间复杂度：O(nlogn)，空间复杂度：O(n)
func mergeSort(data []int)  {
	if len(data) <= 1 {
		return
	}
	mid := len(data)/2
	mergeSort(data[:mid])
	mergeSort(data[mid:])
	mergeArray(data, mid)
}

// 合并数组
func mergeArray(data []int, mid int) {
	size := len(data)
	var temp = make([]int, size)
	k, i, j := 0, 0, mid
	for i < mid && j < size {
		if data[i] < data[j] {
			temp[k] = data[i]
			i++
		} else {
			temp[k] = data[j]
			j++
		}
		k++
	}
	for i < mid {
		temp[k] = data[i]
		i++
		k++
	}
	for j < size {
		temp[k] = data[j]
		j++
		k++
	}
	for i := 0; i < size; i++ {
		data[i] = temp[i]
	}
}
```

#### 2.5 快速排序

**算法思想：**

> 算法过程：
>
> 1. 从序列中挑出一个元素，称为"基准"（pivot）
> 2. 重新排序序列，所有比基准值小的元素摆放在基准前面，所有比基准值大的元素摆在基准后面，该基准就处于数列的中间位置
> 3. 递归地把小于基准值元素的子序列和大于基准值元素的子序列排序

**代码实现：**

```go
// 快速排序：是不稳定排序，时间复杂度：O(nlogn)，空间复杂度：O(logn)
func quickSort(data []int) {
	if len(data) <= 1 {
		return
	}
	l := partition(data)
	quickSort(data[:l])
	quickSort(data[l+1:])
}

// 分区
func partition(data []int) int {
	base := data[0]
	l, r := 0, len(data)-1
	for l < r {
		for l < r && data[r] >= base {
			r--
		}
		data[l] = data[r]
		for l < r && data[l] <= base {
			l++
		}
		data[r] = data[l]
	}
	data[l] = base
	return l
}
```

#### 2.6 **堆排序**

**算法思想：**

> 算法过程：
>
> 1. 建立一个最小堆，每次输出根元素
> 2. 重新建立最小堆
> 3. 重复1、2过程，直到树为空

**代码实现：**

```go
// 堆排序：是不稳定排序，时间复杂度：O(nlogn)，空间复杂度：O(1)
func heapSort(data []int) {
	size := len(data)
	if size <= 1 {
		return
	}
	// 构建堆顶
	for i := size/2 - 1; i >= 0; i-- {
		adjustHeap(data, i, size)
	}
	// 调整堆结构+交换堆顶元素与末尾元素
	for i := size - 1; i > 0; i-- {
		data[0], data[i] = data[i], data[0]
		adjustHeap(data, 0, i)
	}
}

// 堆调整
func adjustHeap(data []int, start int, end int) {
	temp := data[start]
	for i := start*2 + 1; i < end; i = i*2 + 1 {
		if i+1 < end && data[i] < data[i+1] {
			i++
		}
		if data[i] > temp {
			data[start] = data[i]
			start = i
		} else {
			break
		}
	}
	data[start] = temp
}
```

#### 2.7 **希尔排序**

**算法思想：**

> 是插入排序的优化版，算法过程：
>
> 1. 选择一个增量序列
> 2. 按增量序列个数k，对序列进行 k 趟排序
> 3. 每趟排序，根据对应的增量ti，将待排序列分割成若干长度为m 的子序列，分别对各子表进行直接插入排序

**代码实现：**

```go
// 希尔排序：是不稳定排序，时间复杂度：O(nlogn)，空间复杂度：O(1)
// 常见的增量序列：
// 1.最初Donald Shell提出的增量，即折半降低直到1。据研究，使用希尔增量，其时间复杂度还是O(n2)。
// 2.Hibbard增量：{1, 3, ..., 2k-1}，该增量序列的时间复杂度大约是O(n1.5)。
// 3.Sedgewick增量：(1, 5, 19, 41, 109,...)，其生成序列或者是94i - 92i + 1或者是4i - 3*2i + 1。
func shellSort(data []int) {
	size := len(data)
	if size <= 1 {
		return
	}
	for delta := size/2; delta >= 1; delta/=2 {
		for i := delta; i < size; i++ {
			for j := i; j >= delta && data[j] < data[j-delta]; j-=delta {
				data[j], data[j-delta] = data[j-delta], data[j]
			}
		}
	}
}
```

#### 2.8 计数排序

**算法思想：**

> 是插入排序的优化版，算法过程：
>
> 1. 找出待排序的数组中最大和最小的元素
> 2. 统计数组中每个值i出现的次数，存入计数数组的第i项
> 4. 反向填充目标数组：将每个元素i放在新数组的第C(i)项，每放一个元素就将C(i)减去1

**代码实现：**

```go
// 计数排序：是稳定排序，时间复杂度：O(n+k)，空间复杂度：O(k)
func countSort(data []int) {
	size := len(data)
	if size <= 1 {
		return
	}
	// 找出数组最大最小值
	min, max := math.MaxInt8, math.MinInt8
	for _, v := range data {
		if v < min {
			min = v
		}
		if v > max {
			max = v
		}
	}
	// 计数数组
	c := make([]int, max-min+1)
	for _, v := range data {
		c[v-min]++
	}
	// 将值替换到原数组
	index := 0
	for i, v := range c {
		for v > 0 {
			data[index] = i + min
			v--
			index++
		}
	}
}
```

#### 2.9 桶排序

**算法思想：**

> 计数排序是桶排序的一种特殊情况，可以把计数排序当成每个桶里只有一个元素的情况，桶排序算法过程：
>
> 1. 找出待排序的数组中最大和最小的元素
> 2. 遍历数组，计算每个元素i存放的桶
> 3. 每个桶各自排序
> 4. 遍历桶数组，把排序好的元素放进输出数组

**代码实现：**

```go
// 桶排序：是不稳定排序（依赖底层排序的稳定性），时间复杂度：O(n+k)，空间复杂度：O(k)
func bucketSort(data []int) {
	size := len(data)
	if size <= 1 {
		return
	}
	// 找出数组最大最小值
	min, max := math.MaxInt8, math.MinInt8
	for _, v := range data {
		if v < min {
			min = v
		}
		if v > max {
			max = v
		}
	}
	// 桶
	bucketNum := (max-min)/size+1
	bucket := make([][]int, bucketNum)
	for _, v := range data {
		index := (v-min)/size
		bucket[index] = append(bucket[index], v)
	}
	// 对每个桶进行排序，将值替换到原数组
	index := 0
	for _, items := range bucket {
		// 底层用的快排，因此不稳定
		sort.Ints(items)
		for _, v := range items {
			data[index] = v
			index++
		}
	}
}
```

#### 2.10 基数排序

**算法思想：**

> 基数排序(Radix Sort)是桶排序的扩展，算法过程：
>
> 1. 取得数组中的最大数，并取得位数；
> 2. arr为原始数组，从最低位开始取每个位组成radix数组；
> 3. 对radix进行计数排序（利用计数排序适用于小范围数的特点）

**代码实现：**

```go
// 基数排序：是稳定排序，时间复杂度：O(n*k)，空间复杂度：O(k)
func radixSort(data []int)  {
	size := len(data)
	if size <= 1 {
		return
	}
	// 找出数组最大值
	max:= math.MinInt8
	for _, v := range data {
		if v > max {
			max = v
		}
	}
	// 按位数排序
	for exp := 1; max/exp > 0; exp *= 10 {
		bucket := make([][]int, 10)
		for i := 0; i < size; i++ {
			index := (data[i]/exp)%10
			bucket[index] = append(bucket[index], data[i])
		}
		index := 0
		for _, items := range bucket {
			for _, v := range items {
				data[index] = v
				index++
			}
		}
	}
}
```



