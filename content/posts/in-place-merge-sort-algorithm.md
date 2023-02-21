+++
title = "In-Place Merge Sort Algorithm"
date = "2020-04-24T23:22:15+08:00"
author = "Michael Chan"
authorTwitter = "konomichael_" #do not include @
cover = ""
tags = ["sort", "algorithm"]
keywords = ["sort", "algorithm"]
description = ""
showFullContent = false
readingTime = true

+++

Merge sort is a popular sorting algorithm that has an average-case time complexity of O(n log n). The algorithm works by dividing the input array into two halves, sorting them recursively, and then merging them back together. However, the standard merge sort algorithm requires extra memory to hold the two sub-arrays during the merge step, which can be a bottleneck for large arrays. In contrast, in-place merge sort is a variant of the merge sort algorithm that sorts the array in place, using no extra memory.

In this blog post, we'll explore the in-place merge sort algorithm and how it works.

<!--more-->

## Overview of Merge Sort

Before diving into in-place merge sort, let's briefly review the standard merge sort algorithm. The basic idea of merge sort is to divide the input array into two halves, sort them recursively, and then merge them back together. Here's the Golang code for merge sort:

```go
func mergeSort(arr []int) []int {
    if len(arr) <= 1 {
        return arr
    }
    mid := len(arr) / 2
    left := arr[:mid]
    right := arr[mid:]
    left = mergeSort(left)
    right = mergeSort(right)
    return merge(left, right)
}

func merge(left []int, right []int) []int {
    merged := make([]int, len(left)+len(right))
    i, j, k := 0, 0, 0
    for i < len(left) && j < len(right) {
        if left[i] < right[j] {
            merged[k] = left[i]
            i++
        } else {
            merged[k] = right[j]
            j++
        }
        k++
    }
    for i < len(left) {
        merged[k] = left[i]
        i++
        k++
    }
    for j < len(right) {
        merged[k] = right[j]
        j++
        k++
    }
    return merged
}
```

In the above code, the `mergeSort()` function takes an array `arr` and sorts it in ascending order using the merge sort algorithm. The base case is when the length of `arr` is 1 or less, which is already sorted. Otherwise, the function divides `arr` into two halves, `left` and `right`, and sorts each half recursively using `mergeSort()`. Once the two halves are sorted, the function merges them back together into a single sorted array using the `merge()` function. The `merge()` function takes two arrays, `left` and `right`, and returns a merged array containing all `left` and `right` elements, sorted in ascending order.

## In-Place Merge Sort

In-place merge sort is a variant of the merge sort algorithm that sorts the array in place, without using any extra memory. The key idea is to merge the two halves of the array by swapping elements in the original array, rather than creating a new array to hold the merged result. This is accomplished by using a divide-and-conquer approach, similar to the standard merge sort algorithm.

Here's the pseudocode for in-place merge sort:

```go
type Sortable interface {
	Len() int
	Less(i, j int) bool
	Swap(i, j int)
}

func MergeSort(s Sortable) {
	mergeSort(s, 0, s.Len())
}

// merge_sort sort s[start, end) in ascending order
func mergeSort(s Sortable, start, end int) {
	if end-start <= 1 {
		return
	}

	size := end - start
	mid := start + size>>1

	mergeSort(s, start, mid)
	mergeSort(s, mid, end)

	merge(s, start, mid, end)
}

func merge(s Sortable, a, m, b int) {
	ln, rn := m-a, b-m
	if ln == 0 || rn == 0 {
		return
	}

	if ln == 1 && rn == 1 {
		if s.Less(m, a) {
			s.Swap(a, m)
		}
		return
	}

	var p, q int

	if ln <= rn {
		q = m + rn>>1
		i, j := a, m
		// find lowest idx such that s[idx] >= s[q]
		// if not found, then idx = m
		for i < j {
			mid := i + (j-i)>>1
			if s.Less(mid, q) {
				i = mid + 1
			} else {
				j = mid
			}
		}
		p = i
	} else {
		p = a + ln>>1
		i, j := m, b
		// find highest idx such that s[idx-1] < s[p]
		// if not found. then idx = m
		for i < j {
			mid := i + (j-i)>>1
			if s.Less(mid, p) {
				i = mid + 1
			} else {
				j = mid
			}
		}
		q = i
	}

	// sort s[p, q) by rotatation
	// if p == m or q == m then rotate twice, which means nothing change
	if p != m && q != m {
		rotate(s, p, m, q)
	}
	m = p + q - m

	merge(s, a, p, m)
	merge(s, m, q, b)
}

func rotate(s Sortable, p, m, q int) {
	reverse(s, p, m)
	reverse(s, m, q)
	reverse(s, p, q)
}

// reverse [a, b)
func reverse(s Sortable, a, b int) {
	for a < b {
		s.Swap(a, b-1)
		a++
		b--
	}
}
```

The `merge` function is the core of the in-place merge sort algorithm. It takes as input the `Sortable` slice to be sorted, the starting index `a` of the left subarray, the midpoint index `m` that divides the left and right subarrays, and the ending index `b` of the right subarray.

The function first calculates the sizes of the two subarrays as `ln = m - a` and `rn = b - m`. If either subarray has size zero, then there is nothing to do, so the function simply returns.

If both subarrays have size one, the function compares the elements at the two indices `a` and `m`. If the element at `m` is less than the element at `a`, the function swaps them.

```text
            a=0                                  m=6                                       b=13
             │                                    │                                         │
             ▼                                    ▼                                         ▼
           ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
merge      │ -784│ 38  │ 74  │ 238 │ 959 │ 9845│-5467│ 0   │ 0   │  42 │ 905 │7586 │7586 │
           └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
                          ▲                                          ▲
                          │                                          │
initialize q              p=2              m = p+q-m=5             q=(m+b)/2=9
                          │                 │                        │
binary search             ▼                 ▼                        ▼
           ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
rotate     │ -784│ 38  │-5467│  0  │  0  │ 74  │ 238 │ 959 │9845 │  42 │ 905 │7586 │7586 │
           └─────┴─────┴──┬──┴─────┴─────┴─────┴─────┴─────┴───┬─┴─────┴─────┴─────┴─────┘
                          │                                    │
                          └────────────akready sorted──────────┘

           ┌─────┬─────┬─────┬─────┬─────┐
merge      │ -784│ 38  │-5467│  0  │  0  │
           └─────┴─────┴─────┴─────┴─────┘

           ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
merge      │ 74  │ 238 │ 959 │9845 │  42 │ 905 │7586 │7586 │
           └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
```



For subarrays with size greater than one, the function uses a binary search to find the index `p` that divides the left subarray into two parts such that the maximum element in the left subarray is less than or equal to the minimum element in the right subarray. Similarly, it finds the index `q` that divides the right subarray into two parts such that the maximum element in the left subarray is less than or equal to the minimum element in the right subarray.

Once the indices `p` and `q` are known, the function rotates the slice so that the elements in the left subarray are in the correct order, followed by the elements in the right subarray. The function then recursively merges the left subarray and the right subarray.

The `rotate` function takes as input the `Sortable` slice to be rotated, and three indices `p`, `m`, and `q`. It first reverses the slice from `p` to `m`, then reverses the slice from `m` to `q`, and finally reverses the entire slice from `p` to `q-1`.

The `reverse` function takes as input the `Sortable` slice to be reversed, and two indices `a` and `b`. It iterates over the slice from `a` to `b-1`, swapping each element with its corresponding element at the opposite end of the slice, until it reaches the midpoint of the slice.

In summary, the in-place merge sort algorithm sorts a `Sortable` slice in O(n log n) time and O(1) space(without taking stack in count), by repeatedly dividing the slice into two subarrays, sorting each subarray recursively, and merging the two sorted subarrays in place. The algorithm uses a binary search to find the index that divides each subarray into two parts, and a rotation operation to ensure that the elements in the left subarray are in the correct order, followed by the elements in the right subarray.
