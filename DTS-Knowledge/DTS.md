# Data and structure

## 排序

### 选择排序

**原理：** 第一次循环从数组中选择最小或者最大的那个元素放在最前面或者最后面的位置，下一次循环从n-1个元素中选出最小或者最大的那个元素放在最前面或者最后面的位置，后面依次类推。经过n-1次完成排序。



```java
    public static void selectSort(int[] array){

        for (int i = 0; i < array.length ; i++) {

            int k = i;

            for (int j = i+1; j < array.length ; j++) {

                if (array[k] > array[j]){
                    k = j;
                }
            }

            if ( k != i){
                swap(array,k,i);

            }

        }

    }


    private static void swap(int[] array,int x ,int y){
        array[x] = array[x] ^ array[y];
        array[y] = array[x] ^ array[y];
        array[x] = array[x] ^ array[y];
    }
```

从代码中可以看出一共遍历了n + n-1 + n-2 + … + 2 + 1 = n * (n+1) / 2 = 0.5 * n ^ 2 + 0.5 * n，那么时间复杂度是O(N^2)。

### 快速排序

**原理：** 快速排序通常以下几步：

1. 在数组中选一个基准数（通常为数组第一个）；
2. 将数组中小于基准数的数据移到基准数左边，大于基准数的移到右边；
3. 对于基准数左、右两边的数组，不断重复以上两个过程，直到每个子集只有一个元素，即为全部有序

```jAVA
    private static void quickSort(int[] array ,int left,int right){

        if (left > right){
            return;
        }

        //定义排序左边界
        int i = left;
        //定义排序右边界
        int j=right;
        //临时变量存放基数的值
        int k=array[i];

        while (i < j){

            while (i < j && array[j] > k){
                j--;
            }

            //将右边小于等于基准元素的数填入右边相应位置
            array[i] = array[j];

            while (i < j && array[i] < k){
                i++;
            }
            //将左边大于基准元素的数填入左边相应位置
            array[j] = array[i];
        }

        array[i] = k;

        quickSort(array,left,i - 1);
        quickSort(array,i+1,right);

    }
```



#### 解法二：

**原理**：双指针移动法

```
    private static void quickSort2(int[] array,int left,int right){

        if (left > right){
            return;
        }

        int index = partSort(array,left,right);
        quickSort2(array,left,index-1);
        quickSort2(array,index + 1,right);
    }

    private static int partSort(int[] array, int left, int right){

        int key = array[right];
        int start = right;

        while (left < right){


            while (left < right && array[left] <= key){
                ++left;
            }

            while (left < right && array[right] >= key){
                --right;
            }

              if (right != left){
                  swap2(array,left,right);
              }

        }

        swap2(array,left,start);

        return left;

    }


    private static void swap2(int[] array,int x ,int y){
        int temp;
        temp = array[y];
        array[y] = array[x];
        array[x] = temp;

    }
```



# 树

**完全二叉树**：若设二叉树的深度为k，除第 k 层外，其它各层 (1～k-1) 的结点数都达到最大个数，第k 层所有的结点都**连续集中在最左边**，这就是完全二叉树

**满二叉树：**一棵二叉树的结点要么是叶子结点，要么它有两个子结点（如果一个二叉树的层数为K，且结点总数是(2^k) -1，则它就是满二叉树。）

**平衡二叉树：**它或者是一颗空树，或它的左子树和右子树的深度之差(平衡因子)的绝对值不超过1，且它的左子树和右子树都是一颗平衡二叉树。

**最优二叉树（哈夫曼树）**：树的带权路径长度达到最小，称这样的二叉树为最优二叉树，也称为哈夫曼树(Huffman Tree)。哈夫曼树是带权路径长度最短的树，权值较大的结点离根较近。