---
layout: post
title: 常用排序算法实现及复杂度分析
category: 技术
tags: [排序算法, sort]
keywords: 排序算法
description: 
---

> 用正确的工具，做正确的事情

排序和查找是所有计算机算法的核心，高效的排序和查找算法至关重要。根据排序实现的原理进行划分，排序算法基本可以分为4类：

> 1.基于插入的排序算法，比如直接插入排序算法、折半插入排序算法、链表插入排序算法以及希尔排序算法。希尔排序是基于插入的排序算法中效率最高的算法，另外归并排序也可以看成是一种插入排序算法。
  2.基于选择的排序算法，比如选择排序算法、堆排序算法。堆排序算法是效率最高的选择排序算法，堆排序是一种不稳定的排序算法，时间复杂度是O(nlogn),空间复杂度为O(1)。
  3.基于交换的排序算法，比如冒泡排序算法、快速排序算法。快速排序算法是最高效的基于交换的排序算法，它是一种不稳定的排序算法，时空复杂度为O(nlogn)和O(1)。上面的三种类型的排序算法不管是插入排序还是交换排序，都是基于排序序列中元素的比较，上面的那三类排序算法可以统称为基于比较的排序算法，基于比较的排序算法其优的时间复杂度为O(nlogn)，这个结论可以证明，在此略过。那么有没有更快的，时间复杂度更优的排序算法，答案是有的，那就是基于非比较的排序算法。
  4.基于非比较的排序算法，比如计数排序、基数排序和桶排序算法，这三种排序算法的时间复杂度都为O(n)，但这三种排序算法都存在一定的限制。

## 1.基于插入的排序算法

### 1.1 插入排序

原理：将待排序元素依次插入到前有有序的序列中，直接插入排序的实现如下：

	void insert_sort(int l, int r, int list[]) {
	    int i, j, t;
	    for(i = l+1; i <= r; i++) {
	        t = list[i];
	        for(j = i-1; j >= l && list[j] > t; j--) { /* 小于t的元素依次右移 */
	            list[j+1] = list[j];
	        }
	        list[j+1] = t; /* 在合适的位置插入元素 */
	    }
	    return;
	}

直接插入排序的时间开销主要是：1、通过比较以查找合适的插入位置。2、元素的移动，对直接插入排序的优化也主要针对这两点，减少查找插入位置比较的次数可以使用折半查找的方法-折半插入排序，减少元素的移动可以使用静态链表结构来存在记录-链表插入排序。插入排序对基本有序的序列效率较高，可以将序列分割成多个子序列，然后对多个子序列进行跳跃式插入，当整个序列基本有序后，对整个序列进行一次插入排序-希尔排序。

### 1.2 希尔排序

原理：插入排序对基本有序的序列效率较高，可以将序列分割成多个子序列，然后对多个子序列进行跳跃式插入，当整个序列基本有序后，对整个序列进行一次插入排序。希尔排序的一种实现如下:

	void shell_sort(int a[], int len) {
	    int i, j, gap;
	    for(gap = len/2; gap > 0; gap = gap/2) {/* 控制步长 */
	        for(i = 0; i < gap; i++) { /* 控制子序列的起始元素 */
	            for(j = i+gap; j < len; j = j+gap) {/* 子序列进行插入排序 */
	                if(a[j] < a[j-gap]){/* 判断是否需要向前插入 */
	                    int temp = a[j];
	                    while(j-gap >= 0 && temp < a[j-gap]) {
	                        a[j] = a[j-gap];
	                        j = j-gap;
	                    }
	                    a[j] = temp;
	                }
	            }
	        }
	    }
	}

1.3 归并排序

原理：建立在归并操作上的一种高效排序算法，时间复杂度O(nlogn)的稳定排序算法，但是归并排序需要O(n)的空间复杂度。归并排序算法基于分治的思想，该算法一种递归的实现如下：

	#include <stdlib.h>
	#include <stdio.h>
	#include <time.h>
	/* 合并两个有序序列，temp为合并过程使用的临时数组 */
	void merger_array(int a[], int first, int middle, int last, int temp[]){
	    int l1 = first,  l2 = middle + 1;
	    int r1 = middle, r2 = last;            
	    int k = first;
	
	    while(l1 <= r1 && l2 <= r2) { /* 合并序列过程 */
	        if (a[l1] <= a[l2])
	            temp[k++] = a[l1++];
	        else
	            temp[k++] = a[l2++];
	    }
	    while(l1 <= r1)
	        temp[k++] = a[l1++];
	    while(l2 <= r2)
	        temp[k++] = a[l2++];
	
	    for (l1 =first; l1 < k; l1++) /* 将合并后的序列拷贝回原数组 */
	        a[l1] = temp[l1];
	
	    return;
	}
	/* 通过分治的方法，进行归并排序 */
	void merger_sort(int a[], int first, int last, int temp[]) {
	    int middle = (first+last)/2;
	    if (first == last)  /* 递归结束条件 */
	        return;
	    merger_sort(a, first, middle, temp); 
	    merger_sort(a, middle+1, last, temp);
	    merger_array(a, first, middle, last, temp);
	    return;
	}
	
	/* 填充并生成一个随机数组 */
	void build_rand_array(int a[], int len) {
	    int i;
	
	    srand((int)time(0));
	    for (i = first; i < len; i++)
	        a[i] = rand()%len;
	    return;
	}
	
	/* 打印数组 */
	void print_array(int a[], int len) {
	    int i;
	
	    for (i = 0; i < len; i++)
	        printf(i != len-1 ? "%4d ": "%4d\n", a[i]);
	    return;
	}
	
	int main(void) {
	    int a[10], temp[10];
		
	    build_rand_array(a, 10);
	    print_array(a, 10);
	    merger_sort(a, 0, 9, temp);
	    print_array(a, 10);
	
	    return 0;
	}


基于递归的归并算法实现存在一些问题：1、当问题规模较大时，递归深度较大，可能会导致程序栈溢出。2、递归函数开销较大，非递归的归并算法如下：

	void merger_sort_noRecur(int a[], int first, int last, int temp[]) {
	    int s = 2, i; /* s表示划分数组的步长，i表示当前数组的位置 */
	    int n = last+1; /* 数组长度 */
	    
	    while(s < n) { /* 因为最后还有一轮归并，不需要s<=n */
	        /* 每一轮循环后，a[first...first+s-1] ...子序列保持有序 */
	        i = first;
	        while(i+s < n) {/* 因为有对剩余部分的处理，因此不需要i+s<=n */
	            merger_array(a, i, i+s/2-1, i+s-1, temp);
	            i = i+s;
	        }
	        /* 处理剩余部分，注意当i+s/2>=n,表示剩余部分的长度不大于s/2，
	           此时剩余部分有序，不需要归并 */
	        if(i+s/2 < n)
	            merger_array(a, i, i+s/2-1, n-1, temp);
	        s = s*2;
	    }
	    /* 最后一轮归并，此时a[first...first+s/2-1] a[first+s/2...n-1]已经有序 */
	    merger_array(a, first, first+s/2-1, n-1, temp);
	}

另外可以利用待排序数组的局部有序性，提高归并排序算法归并操作的效率，进而提高归并排序算法的效率-自然归并排序。

## 2.基于选择的排序算法

2.1 选择排序
原理：依次选出待排序列的最小值，与待排序列的第一元素交换位置。

	void swap(int *a, int *b) {int t = *a;*a = *b;*b = t;}
	
	void select_sort(int l, int r, int list[]) {
	    int i, j, t, k;
	    for(i = l; i < r; i++)
	    {
	        k = i;
	        for(j = i+1; j <= r; j++)  /* 选出待排序列的最小元素 */
	            if (list[j] < list[k])
	                k = j;
	        swap(&list[i], &list[k]);/* 将待排序列的最小元素与待排序列的第一个元素交换位置 */
	    }
	    return;
	}

选择排序的主要开销在于：选出最大或最小的元素，O(n)的时间复杂度，对于选择排序的优化主要是降低选出最大或最小元素的时间复杂度。堆排序可以看做是对选择排序的一种优化，堆排序将在后面介绍。

### 2.2 堆排序

原理：堆排序是一种时间复杂度O(nlogn)，空间复杂度O(1)的不稳定排序算法。堆用一颗完全二叉树表示，可用数组来保存一颗完全二叉树的结构和数据。堆排序利用了完全二叉树下面的性质：

> 1.一颗完全二叉树有n个节点，那么分支节点的个数是n/2（向下取整）。
  2.用一个数组a[n]来保存一颗完全二叉树，数组下标从0开始，a[i]的左右孩子节点为a[2*i+1]、a[2*i+2]，如　　　　果存在的话，a[i]的父亲节点为a[(i-1)/2]，(i-1)/2为向下取整。

堆排序的一种实现如下：

	#include <stdlib.h>
	#include <stdio.h>
	#define SWAP(x, y, type) \
	    do {type z = x; x = y; y = z;} \
	    while (0)
	
	/* 调整堆  */
	void adjust_heap(int a[], int len, int i);
	/* 将数组a创建成一个最大堆 */
	void build_heap(int a[], int len);
	/* 堆排序 */
	void heap_sort(int a[], int len);
	
	void adjust_heap(int a[], int len, int i)
	{
	    int p = i;                /* 记录父节点下标 */
	    int l = 2*p+1, r = 2*p+2; /* 左右孩子节点的下标 */
	    int t = p, large = a[p]; /* 当前最大值节点，初始化为父节点 */
	
	    while(l < len || r < len) {
	        if(r < len && a[r] > large) {
	            t = r;
	            large = a[r];
	        }
	        if(l < len && a[l] > large){
	            t = l;
	            large = a[l];
	        }
	        if (t == p) /* t == i代表已经是堆，可以返回 */
	            return;
	        else {
	            SWAP(a[p], a[t], int);
	
	            p = t; /* 调整以a[t]为根的子树 */
	            l = 2*p+1, r = 2*p+2;
	            large = a[p], t = p;
	        }
	    }
	
	}
	
	void build_heap(int a[], int len)
	{
	    int i = len/2-1;
	    /* 从下标最大的分支节点开始构建一个堆结构 */
	    for( ; i >= 0; i--) { 
	        adjust_heap(a, len, i);
	    }
	}
	
	void heap_sort(int a[], int len)
	{
	    int l = len;
	    /* 首先将数组构建成堆结构 */
	    build_heap(a, l);
	    /* 取下堆顶元素放入有序序列，将剩下的在调整为一个堆结构 */
	    while(l > 1) 
	    {
	        l = l-1;
	        SWAP(a[0], a[l], int);/* a[l-1...len]有序 */
	        adjust_heap(a, l, 0); /* 从a[0]开始调整堆 */
	    }
	
	}
	
## 3.基于交换的排序算法

### 3.1 冒泡排序：

冒泡排序算法的原理：相邻的元素之间比较，如果相邻两个元素间不满足顺序关系，则相邻两个元素交换位置。基本的冒泡排序算法如下：

	void swap(int *a, int *b) {int t = *a;*a=*b;*b=t;}
	
	void bubble_sort_v1(int l, int r, int list[]) {
	    int i, j, t;
	    for (i = l; i < r; i++)
	        for (j = l; j < r-i+l; j++)
	            if(list[j] > list[j+1])
	                swap(&list[j], &list[j+1]);
	    return;
	}

在冒泡过程中，如果在某一趟冒泡过程中，没有元素的交换，那么表明当前序列已经有序，可以结束冒泡，按此方法优化后的冒泡排序如下：

	void bubble_sort_v2(int l, int r, int list[]) {
	    int i, j, t, is_sorted;
	    for (i = l; i < r; i++) {
	        is_sorted = 1;
	        for (j = l; j < r-i+l; j++) /* list[r-i+1..r]已经为有序序列 */
	            if(list[j] > list[j+1]) {
	                swap(&list[j], &list[j+1]);
	                is_sorted = 0;              /* 本轮冒泡发生交换，那么表明当前序列无序 */
	            }
	        if (is_sorted) break;
	    }
	    return;
	}

对冒泡排序做进一步优化：每一轮冒泡的结果是使得序列中有序序列的长度增加，上面的冒泡算法中每次冒泡仅仅将有序序列的长度增加1，增加了不必要的元素比较。为此在每一论冒泡过程中，记录最后一次元素交换的位置，在该位置之后的序列已经有序，优化后的排序算法如下：

	void bubble_sort_v3(int l, int r, int list[]) {
		int i, j, t, is_sorted;
	    i = r;  /* i记录有序序列的最左下标，list[i...r]为有序序列 */
	    while(i > l) {
	        is_sorted = 1;
	        for (j = l; j < i; j++)
	            if(list[j] > list[j+1]) {
	                swap(&list[j], &list[j+1]);
	                t = j;                       /* t记录每一轮冒泡，最后元素交换发生的位置 */
	                is_sorted = 0;
	            }
	        if (is_sorted) break;
	        i = t;
	    }
	    return;
	}

### 3.2 快速排序

快速排序算法的原理：从待排序列中选取一个元素作为支点，以支点将待排序列划分成两个部分，左边部分的元素都比支点元素小，右边部分的元素都比支点元素大，然后按此方法对左右两部分进行快速排序，整个过程递归的进行。快速排序存在两个基本的不同版本：Hoare版和Lomuto版，两个版本的不同之处在于partition算法，Hoare版以支点元素从待排序列的两端进行序列的划分，Lomuto版以支点元素从待排序列的一端开始序列划分。Hoare版和Lomuto版快速排序算法如下：

	int partition_Hoare(int l, int r, int list[]) {
	    int i, j, t, k;
	    i = l;
	    j = r;
	    k = list[i]; /* 选择序列的第一个元素作为支点元素 */
	    while (i != j){
	        while (list[j] >= k && j > i) j--;
	        if (j > i)
	            list[i] = list[j];
	        while (list[i] <= k && j > i) i++;
	        if (j > i)
	            list[j] = list[i];
	    }
	    list[i] = k; /* 最后将支点元素放到合适的位置 */
	    return i;
	}
	
	int partition_Lomuto(int l, int r, int list[]) {
	    int i, k, m;
	    k = list[r]; /* 选择最后一个元素作为支点元素 */
	    m = l-1;     /* list[l...m]为小于等于k的元素 */
	    for(i = l; i < r; i++) {   /* 遍历列表，将小于等于k的元素左移 */
	        if (list[i] <= k) {
	            m ++;
	            if (m != i)
	                swap(&list[m], &list[i]);
	        }
	    }
	    swap(&list[++m], &list[r]);/* 最后将支点元素放到合适的位置 */
	    return m;
	}
	void quick_sort(int l, int r, int list[]) {
	    int m;
	    if (l >= r)
	        return;
	    m = partition_Lomuto(l, r, list);
	    quick_sort(l, m-1, list);
	    quick_sort(m+1, r, list);
	    return;
	}

支点元素的选择对于快速排序至关重要，选择待排序列固定位置的元素作为支点元素，在某些情形下会使得快速排序具有O(n2)的时间复杂度，对于快速排序的优化一般可以将支点元素的选择随机化或者每次选择序列的中间位置元素作为支点元素。Lomuto版的划分算法可以用来实现针对单向链表的快速排序算法。

## 4. 基于非比较的排序算法

### 4.1 计数排序
原理：对一个待排序列，如果我们知道小于等于元素A[i]的数的个数，那么我们就可以确定A[i]在有序序列中的位置。计数排序要求数据的范围在0到k之间的整数，引入了一个辅助数组C，数组C的大小为k，存储了待排序数组中值小于等于C的索引值的个数，计数排序的一种实现如下：

	#include <stdlib.h>
	#include <stdio.h>
	
	/* A[]:计数数组
	 * k:计数范围[0,k]
	 * D[]:原始数据数组
	 * n:数据规模 */
	void counting_sort(int A[], int k, int D[], int n) {
	    int C[k+1], T[n];
	    int i, j;
	    /* 初始化 */
	    for(i = 0; i < k+1; i++)
	        C[i] = 0;
	
	    /* 每个值为A[i]的计数的个数 */
	    for(i = 0; i < n; i++)
	        C[A[i]]++;
	
	    /* 小于等于i的计数的个数 */
	    for(i = 1; i < k+1; i++)
	        C[i] = C[i-1] + C[i];
	
	    /* 参考C[i]的i信息，排序数据数组D */
	    for(i = n-1; i >= 0; i--) {
	         /* A[i]是D[i]的计数数据，
	          * C[A[i]]是小于等于A[i]的计数个数，
	          * 因此第一个A[i]对应的数据应该在有序序列的位置是C[A[i]]-1,
	          * 序列下标从0开始 */
	        T[C[A[i]]-1] = D[i];
	        C[A[i]] --;
	    }
	
	    for(i = 0; i < n; i++)
	        D[i] = T[i];
	}
	int main(void) {
	    int n = 10, k = 9;
	    int i;
	    int D[] = {0,40,20,10,50,80,90,30,50,90};
	    int A[] = {0,4,2,1,5,8,9,3,5,9};
	    counting_sort(A, k, D, n);
	    for(i = 0; i < n; i++)
	        printf(i ==9 ? "%d\n" : "%d ", D[i]);
	    return 0;
	}

计数排序简单分析：

> 计数排序仅适合于小范围的数据进行排序;
  不能对浮点数进行排序;
  时间复杂度为O(n),稳定的排序算法。

### 4.2 基数排序

原理：基数排序是将整数按位进行排序，从低位开始，对每一位使用稳定的排序算法如计数排序进行排序，直到最高位排序完成，此时序列有序。针对每一位进行排序时必须使用稳定的排序算法，否则会出现错误。基数排序算法可以看成是对计数排序的一种改进，基于计数排序的基数排序算法的一种实现如下：		


	/* 基数排序：
	 * A: 待排序数组
	 * n：问题规模
	 * wordLen：排序数组中最大数的位数 */
	void radix_sort(int D[], int n, int wordLen) {
	    int i, j, l = wordLen;
	    int base = 1;
	    int A[n]; /* 计数数组 */
	    base = base*10;
	    while(l > 0){
	        /* 计算计数数组 */
	        for(i = 0; i < n; i++) {
	            A[i] = D[i]%base;  /* 截取当前位到末位的数 */
	            A[i] = A[i]/(base/10);/* 截取当前位 */
	        }
	        counting_sort(A, 9, D, n);/* 针对每一位进行计数排序 */
	        l --;
	        base = base*10;
	    }
	}
	int main(void) {
	    int n = 10, k = 9;
	    int i;
	    int D[] = {0,40,20,10,50,80,90,30,50,90};  
	    radix_sort(D, n, 2);
	    for(i = 0; i < n; i++)
	        printf(i ==9 ? "%d\n" : "%d ", D[i]);
	    return 0;
	}
	
	
基数排序简单分析：

> 基数排序算法仅排序整数序列，与计数不同基数排序可以排序大整数。
  对每一位进行排序时，需要使用稳定的排序算法，保证在排序高位时低位的顺序不会变。
  时间复杂度为O(n)的稳定排序算法。


### 4.3 桶排序
原理：对于一组规模为N的待排序数据，将这些数据划分为M个区间（即放入M个桶中）。根据某种映射函数，将这N个数据放入M个桶中。然后对每个桶中的数据进行排序，最后依次输出，得到已排序数据。桶排序要求待排序的元素都属于一个固定的且有限的区间范围内。

桶排序的简单分析：

> 桶排序必须知道待排序列中元素值的区间。
  与计数排序和基数排序相比，桶排序可以排序浮点数。
  桶排序是一种稳定的，时间复杂度为O(n)的排序算法。
  桶排序比较消耗存储。



玩的开心 !!!
