+++
title = "Quick Sort Elegant Implementation"
date = "2020-07-27T19:05:30+08:00"
author = "Michael Chan"
authorTwitter = "konomichael_" #do not include @
tags = ["sort", "snippet"]
keywords = ["sort", "snippet"]

+++

This is a code snippet of my implementation of quicksort.

```go
func QuickSort(s Sortable) {
	quickSort(s, 0, s.Len()-1)
}

func quickSort(s Sortable, l, r int) {
	if l < r {
		p := partition(s, l, r)
		quickSort(s, l, p-1)
		quickSort(s, p+1, r)
	}
}

func partition(s Sortable, l, r int) int {
	randomIdx := l + rand.Intn(r-l+1)
	s.Swap(l, randomIdx)

	i, j := l+1, r
	for {
		for i <= j && s.Less(i, l) {
			i++
		}
		for i <= j && !s.Less(j, l) {
			j--
		}
		if i > j {
			break
		}
		s.Swap(i, j)
	}

	s.Swap(l, j)

	return j
}
```

