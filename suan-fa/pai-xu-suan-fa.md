# 排序算法

![img](https://www.pdai.tech/\_images/alg/alg-sort-overview-1.png)

直接上分析图：

### 直接插入排序

遍历序列中的元素，每次取一个待排序的元素逐次向前扫描 ，如果扫描到的元素大于待排序关键字，则把扫描到的元素往后移一个位置。最后找到插入位置，将待排序元素插入。

![](../.gitbook/assets/02\_02\_00.png)

代码实现如下：

```java
public static int[] test1(int[] arr) {
        //向前比较
        for (int i = 1; i < arr.length; i++) {
            int j = i - 1, tmp;
            tmp = arr[i];
            while (j >= 0 && tmp < arr[j]) {
                arr[j + 1] = arr[j];
                j--;
            }
            arr[j + 1] = tmp;
        }

        return arr;
    }
```

### shell排序

首先它把较大的数据集合分割成若干个小组（逻辑上分组），然后对每一个小组分别进行插入排序，此时，插入排序所作用的数据量比较小（每一个小组），插入的效率比较高

![](../.gitbook/assets/02\_02\_01.png)

```java
 /***
     * 希尔排序
     * @param arr
     * @return
     */
    public static int[] test2(int[] arr) {
        int m = 2;
        int k = arr.length / m;

        while (k > 0) {
            /**
             * 从高位开始进行插入排序
             */
            for (int i = arr.length / m; i < arr.length; i++) {
                int j = i - k, tmp;
                tmp = arr[i];
                while (j >= 0 && tmp < arr[j]) {
                    arr[j + k] = arr[j];
                    j = j - k;
                }
                arr[j + k] = tmp;
            }
            k = k / 2;
            m = m * 2;
        }
        return arr;
    }
```

### 直接选择排序

选择排序就是不断地从未排序的元素中选择最大（或者最小）的元素放入已经排好序的元素集合中

![](../.gitbook/assets/02\_02\_02.png)

```java
 /**
     * 直接选择排序
     * 每次选一个最大的元素放在首位
     *
     * @param arr
     */
    public static int[] test3(int[] arr) {
        int tmp, j;
        for (int i = 0; i < arr.length; i++) {
            j = i + 1;
            while (j < arr.length) {
                if (arr[i] > arr[j]) {
                    tmp = arr[i];
                    arr[i] = arr[j];
                    arr[j] = tmp;
                }
                j++;
            }
        }
        return arr;
    }
```

### 堆排序

把堆看成完全二叉树，大根堆—父亲大孩子小；小根堆—父亲小孩子大。

![](../.gitbook/assets/02\_02\_03.png)

```java
  /**
     * 堆排序
     * 参考PriorityQueue
     *
     * @param arr
     * @return
     */
    public static int[] test4(int[] arr) {
        for (int i = 1; i < arr.length; i++) {
            int k = i, tmp = 0;
            while (k > 0) {
                //从0开始
                int parent = (k - 1) >>> 1;
                if (arr[parent] < arr[k])
                    break;
                tmp = arr[k];
                arr[k] = arr[parent];
                arr[parent] = tmp;
                k = parent;
            }
        }
        return arr;
    }
```

### 冒泡排序

每一个元素与前一个元素进行比较交换,每次将最大的元素置于最后

![](../.gitbook/assets/02\_02\_04.png)

```java
/**
     * 冒泡排序
     *
     * @param arr
     * @return
     */
    public static int[] test5(int[] arr) {
        int tmp;
        for (int i = arr.length - 1; i >= 1; i--) {
            for (int j = 1; j <= i; j++) {
                if (arr[j - 1] > arr[j]) {
                    tmp = arr[j - 1];
                    arr[j - 1] = arr[j];
                    arr[j] = tmp;
                }
            }
        }
        return arr;
    }
```

### 快速排序

首先选择当前序列的一个元素作为轴枢（一般取第一个元素），之后递归将比该元素大的置于右侧，比该元素小的置于左侧，形成两个更小的子序列，成为下一次交换的初始序列

![](../.gitbook/assets/02\_02\_05.png)

```java
   /**
     * 快排
     *
     * @param arr
     */
    public static int[] test6(int[] arr, int i, int j) {
        int low = i, high = j;
        if (low < high) {
            int tmp = arr[low];
            while (low != high) {
                while (low < high && arr[high] >= tmp) high--;
                if (low < high) {
                    arr[low] = arr[high];
                    low++;
                }
                while (low < high && arr[low] <= tmp) low++;
                if (low < high) {
                    arr[high] = arr[low];
                    high--;
                }
            }
            arr[low] = tmp;
            test6(arr, i, low - 1);
            test6(arr, low + 1, j);
        }
        return arr;
    }
```

### 归并排序

![](../.gitbook/assets/02\_02\_06.png)

```java
 /**
     * 归并排序
     * Arrays.sort()采用了一种名为TimSort的排序算法，就是归并排序的优化版本
     *
     * @param arr
     * @return
     */
    public static int[] test7(int[] arr) {
        int[] temp = new int[arr.length];
        sort(arr, 0, arr.length - 1, temp);
        return arr;
    }

    private static int[] sort(int[] arr, int i, int j, int[] temp) {
        if (i < j) {
            int mid = (i + j) / 2;
            sort(arr, i, mid, temp);
            sort(arr, mid + 1, j, temp);
            merge(arr, i, mid, j, temp);
        }
        return arr;
    }

    private static void merge(int[] arr, int left, int mid, int right, int[] temp) {
        int i = left;
        int j = mid + 1;
        int t = 0; //临时数组指针
        while (i <= mid && j <= right) {
            if (arr[i] <= arr[j])
                temp[t++] = arr[i++];
            else
                temp[t++] = arr[j++];
        }
        while (i <= mid) temp[t++] = arr[i++]; //将剩余元素填充到temp
        while (j <= right) temp[t++] = arr[j++];
        t = 0;
        while (left <= right)
            arr[left++] = temp[t++];
    }
```

### 基数排序

“多关键字排序”，（1）最高位优先（2）最低位优先。例如最高位优先：先按最高位排成若干子序列，再对每个子序列按次高位进行排序。

如下图，低位优先的排序过程：每个桶相当于一个队列，先进先出规则。

![](../.gitbook/assets/02\_02\_07.png)

```java
  /**
     * 基数排序
     *
     * @param arr
     */
    public static int[] test8(int[] arr) {
        int n = 0;
        for (int temp : arr) {
            char[] chars = String.valueOf(temp).toCharArray();
            if (n < chars.length) {
                n = chars.length;
            }
        }
        return tongsort(arr, Math.pow(10, n));
    }

    private static int[] tongsort(int[] arr, double d) {
        int n = 1;
        int k = 0;
        int len = arr.length;
        int[][] buk = new int[10][len];
        int[] order = new int[10]; //保存桶里有多少数字
        while (n < d) {
            for (int num : arr) {
                int digit = (num / n) % 10;
                buk[digit][order[digit]] = num;
                order[digit]++;
            }
            for (int i = 0; i < 10; i++) {
                if (order[i] != 0) {
                    for (int j = 0; j < order[i]; j++) {
                        System.out.println(k +"...."+i+"...."+j);
                        arr[k] = buk[i][j];
                        k++;
                    }
                }
                order[i]=0;
            }
            n *= 10;
            k = 0;
        }
        return arr;
    }
```
