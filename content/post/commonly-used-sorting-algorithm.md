---
title: "Commonly Used Sorting Algorithm"
date: 2022-03-19T15:14:12+08:00
---

**作为一个程序员在谈到`算法`这个词的时候，第一反应就是那些令人头疼的LeetCode题目，那本篇文章我们就讲讲程序员几个基础排序算法。**

- 冒泡排序
- 选择排序
- 插入排序
- 希尔排序
- 快速排序
- 归并排序

**复杂度对比图**
![算法复杂度](https://www.runoob.com/wp-content/uploads/2019/03/sort.png)

## 1.冒泡排序
- 对无序数列进行安装每2个相邻数为一组的比较。
- 如果每相邻的2个数,左边比右边大 `left > right`就交换的位置。
- 针对所有的元素重复以上的步骤，除了最后一个。
- 持续每次对越来越少的元素重复上面的步骤，直到没有任何一对数字需要比较。
  
**动图解析**  
![bubble sort](https://www.runoob.com/wp-content/uploads/2019/03/bubbleSort.gif)

**代码实现**
```go
func Bubble(arr []int) {
    for i := 0; i < len(arr); i++ {
        // 冒泡是每次相邻的2个元素排
        for j := 0; j < len(arr)-1-i; j++ {
            if arr[j] > arr[j+1] {
                arr[j], arr[j+1] = arr[j+1], arr[j]
            }
        }
    }
}
```

## 2. 选择排序
- 首先在未排序序列中找到最小（大）元素，存放到排序序列的起始位置。
- 再从剩余未排序元素中继续寻找最小（大）元素，然后放到已排序序列的末尾。
- 重复第二步，直到所有元素均排序完毕。

**动图解析**
![selection sort](https://www.runoob.com/wp-content/uploads/2019/03/selectionSort.gif)

**代码实现**
```go
// 选择排序 O(n^2)
func Selection(arr []int) {
    // 这里的减一是因为需要通过下标方法元素 元素下标是从0开始的
    // 假设mid是最小值，然后和后面元素进行比较
    // 如果发现有比mid小的元素，就更新mid坐标
    for i := 0; i < len(arr)-1; i++ {
        min := i
        for j := i + 1; j < len(arr); j++ {
            if arr[min] > arr[j] {
                min = j
            }
        }
        // 一轮结束之后交换2个元素位置
        arr[i], arr[min] = arr[min], arr[i]
    }
}
```

## 3. 插入排序
- 将无序数列看成一个有序数列(范围是第一个元素)和无序数列(范围就是第二元素到末尾元素)。
- 然后从头到尾扫描无序数列，把扫描到的元素插入到有序数列的适当位置。
- 如果待插入的元素与有序序列中的某个元素相等，则将待插入元素插入到相等元素的后面。
  
**动图解析**
![insertion sort](https://www.runoob.com/wp-content/uploads/2019/03/insertionSort.gif)

**代码实现**
```go
func Insertion(numbers []int) {
    for i := 1; i < len(numbers); i++ {
        pervIndex := i - 1
        current := numbers[i]
        // 用上一个元素比较当前的元素
        for pervIndex >= 0 && numbers[pervIndex] > current {
            numbers[pervIndex+1] = numbers[pervIndex]
            // 向左移动方便下次比较
            pervIndex -= 1
        }
        // 如果pervIndex没有变化说明就不需要操作
        if pervIndex+1 != i {
            numbers[pervIndex+1] = current
        }
    }
}
```
## 4. 希尔排序
- 希尔排序属于缩小增量排序。
- 将数据区分成特定间隔的几个小区块。
- 每排完一轮数据渐进式变成有序的。
- 以插入排序法排完区块内的数据后再渐渐减少间隔的距离。

**动图解析**
![shell sort](https://tva1.sinaimg.cn/large/0081Kckwgy1gkh1hyq8a5g30qi0exkc0.gif)

**代码实现**
```go
// 希尔排序 O(n log n)
func Shell(arr []int) {
    var pervIndex, current int
    // 1.先把原数据安装自定义步长分组
    for gap := len(arr) / 2; gap > 0; gap /= 2 {
      // 2.然后分好组的数据进行选择排序
        for i := gap; i < len(arr); i++ {
          // 希尔排序
          pervIndex = i - gap
          current = arr[i]
          for pervIndex >= 0 && arr[pervIndex] > current {
              arr[pervIndex+gap] = arr[pervIndex]
              pervIndex -= gap
          }
          if pervIndex+gap != i {
              arr[pervIndex+gap] = current
          }
        }
    }
}
```

## 5. 递归函数
- 递归函数就是在函数里面调用自己。
- 下面就是一个阶乘例子

```go
// https://zh.wikipedia.org/wiki/%E9%9A%8E%E4%B9%98
func factorial(n int) int {
    if n == 1 {
        return 1
    }
    return n * factorial(n-1)
}
```

## 6. 归并排序
- 该算法是采用`分治法（Divide and Conquer）`的一个非常典型的应用。
- 先将一个无序数列进行分组。
- 然后子组也进行分组。
- 将已有序的子序列合并，得到完全有序的序列；即先使每个子序列有序，再使子序列段间有序。若将两个有序表合并成一个有序表，称为二路归并。

**动图解析**
![merge sort](https://tva1.sinaimg.cn/large/0081Kckwgy1gkj9pvmv9wj31170qlh0x.jpg)


**1.代码实现**

```java
package me.ibyte.algorithm.sort;

import me.ibyte.algorithm.Sort;

import java.util.Arrays;

// Go写多了换Java写一些 思路和逻辑是一样的
public class MergeSort implements Sort {

    public Integer[] sort(Integer[] arr) {
        // 取到中轴 分组
        int middle = arr.length / 2;
        if (arr.length < 2) {
            return arr;
        }
        // 通过中轴分左右组
        Integer[] left = Arrays.copyOfRange(arr, 0, middle);
        Integer[] right = Arrays.copyOfRange(arr, middle, arr.length);
        // 重复分组 直到不能分组为止并且进行小块归并排序
        return merge(sort(left), sort(right));
    }

    private Integer[] merge(Integer[] left, Integer[] right) {
        // 存放排序好的零时数组
        Integer[] result = new Integer[left.length + right.length];
        int i = 0;
        // 不为空就进行排序
        while (left.length != 0 && right.length != 0) {
            if (left[0] <= right[0]) {
                result[i++] = left[0];
                // 排好的就减去
                left = Arrays.copyOfRange(left, 1, left.length);
            } else {
                result[i++] = right[0];
                right = Arrays.copyOfRange(right, 1, right.length);
            }
        }
        // 检测剩下的
        while (left.length != 0) {
            result[i++] = left[0];
            left = Arrays.copyOfRange(left, 1, left.length);
        }
        while (right.length != 0) {
            result[i++] = right[0];
            right = Arrays.copyOfRange(right, 1, right.length);
        }
        return result;
    }
}

```
**源代码查看地址[源代码](https://github.com/auula/java-algorithm/blob/main/src/main/java/me/ibyte/algorithm/sort/MergeSort.java)**

**2.Golang实现就想尝试一下go特性高阶函数**
```go
func mergeSort(arr []int) []int {
    var result []int
    if len(arr) < 2 {
        return arr
    }
    middle := len(arr) / 2
    // 注意这是切片 切取的时候是包左 不包右
    left := arr[:middle]
    right := arr[middle:]
    return func(left, right []int) []int {
      // 分组不能保证左右各组数据个数是一样的
        for len(left) != 0 && len(right) != 0 {
            if left[0] <= right[0] {
              result = append(result, left[0])
              left = left[1:]
            } else {
              result = append(result, right[0])
              right = right[1:]
            }
        }
        for len(left) != 0 {
            result = append(result, left[0])
            // 不断减小剩下的
            left = left[1:]
        }
        for len(right) != 0 {
            result = append(result, right[0])
            right = right[1:]
        }
        return result
    }(mergeSort(left), mergeSort(right))
}
```

## 7. 快速排序
- 该算法是采用`分治法（Divide and Conquer）`的一个非常典型的应用。
- 从数列中挑出来一个元素作为基准`pivot`。
- 重新排序数列,所有元素比基准值小的摆放在基准前面,所有元素比基准值大的摆在基准的后面(相同的数可以到任一边)，在这个分区退出之后,该基准就处于数列的中间位置，这个称为分区`(partition)`操作。
- 递归地`(recursive)`把小于基准值元素的子数列和大于基准值元素的子数列排序。

**视频解析**

<iframe src="//player.bilibili.com/player.html?aid=62621532&bvid=BV1at411T75o&cid=108813206&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true" style="width:300px;hight:500px" > </iframe>

**代码实现**

```java
package me.ibyte.algorithm.sort;


import me.ibyte.algorithm.Sort;


// https://www.bilibili.com/video/BV1at411T75o?zw
public class QuickSort implements Sort {
    @Override
    public Integer[] sort(Integer[] arr) {
        return quickSort(arr,0,arr.length-1);
    }

    private  Integer[] quickSort(Integer[] arr, int left, int right) {
        if (left >= right) {
            return arr;
        }
        // L记录开始位置 R记录尾巴位置
        int pivot = arr[left],L=left,R=right;
        while(left < right){
            while (left < right && arr[right] >= pivot){
                right--;
            }
            arr[left] = arr[right];
            while(left < right && arr[left] <= pivot){
                left++;
            }
            arr[right] = arr[left];
        }
        arr[left] = pivot;
        // L =0 left = 是中心轴值的位置 -1 分左右作用
        quickSort(arr,L,left-1);
        quickSort(arr,left+1,R);
        return  arr;
    }
}
```

## 算法对比
**根据个人电脑配置不同耗时不同！！!**
**随机数据范围是:`rand.Intn(99999) - 9999)`**
**测试过程中包含了随机数生成逻辑代码，所有算法速度应该减去随机数生成耗时（实际时间应该更短）。**


| 算法 | 数量积 | 时间                      |
| ---- | ------ | ------------------------- |
| 冒泡 | 10万   | 14.118 seconds            |
| 选择 | 10万   | 8.862 seconds             |
| 插入 | 10万   | 1.69 seconds              |
| 希尔 | 10万   | 0.25 seconds              |
| 希尔 | 100万  | 0.509 seconds             |
| 希尔 | 1000万 | 3.314 seconds             |
| 希尔 | 1亿    | 40s 左右(5次重复测试结果) |
| 归并 | 100万  | 0.752 seconds             |
| 快速 | 100万  | 0.109 seconds             |

**测试源代码链接[查看源代码](https://github.com/auula/go-algorithm/blob/master/sort.go),别忘了`star`😯。**