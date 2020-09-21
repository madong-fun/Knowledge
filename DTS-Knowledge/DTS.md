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

**原理**：双指针移动法，关于partSort 选择基础数值，是从如果选择最右边元素，遍历时，需要先移动左指针（见partSort方法）；如果选择最左边的元素作为基准数值，需要先移动右指针（见partSort2方法）。

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
    
    public static int partSort2(int[] nums,int left, int right){

        int point = nums[left];
        int start = left;

        while (left < right){

            while (left < right && nums[right] >= point){
                right--;
            }

            while (left < right && nums[left] <= point){
                left ++;
            }

            if (left != right){
                swap(nums,left,right);
            }
        }
        swap2(nums,left,start);
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

**满二叉树**：一棵二叉树的结点要么是叶子结点，要么它有两个子结点（如果一个二叉树的层数为K，且结点总数是(2^k) -1，则它就是满二叉树。）

**平衡二叉树**： 它或者是一颗空树，或它的左子树和右子树的深度之差(平衡因子)的绝对值不超过1，且它的左子树和右子树都是一颗平衡二叉树。

**最优二叉树（哈夫曼树）**：树的带权路径长度达到最小，称这样的二叉树为最优二叉树，也称为哈夫曼树(Huffman Tree)。哈夫曼树是带权路径长度最短的树，权值较大的结点离根较近。

## 回顾二叉树的遍历

### 1.先序遍历 访问顺序：先根节点，再左子树，最后右子树；

```
    //二叉树的遍历，前序遍历
    //访问顺序：先根节点，再左子树，最后右子树；
    private static List<TreeNode> preOrderTraverse(TreeNode root){
        List<TreeNode> list = new LinkedList<>();
        if (root != null){
            list.add(root);
            preOrderTraverse(root.getLeft());
            preOrderTraverse(root.getRight());
        }
        return list;
    }
```

### 2.中序遍历 访问顺序：先左子树，再根节点，最后右子树；

   ```
       private static List<TreeNode> inOrderTraverse(TreeNode root){
        List<TreeNode> list = new LinkedList<>();
        if (root != null){
            preOrderTraverse(root.getLeft());
            list.add(root);
            preOrderTraverse(root.getRight());
        }
        return list;
    }
   ```
### 3.后续遍历 访问顺序：先左子树，再右子树，最后根节点

```
    private static List<TreeNode> postOrderTraverse(TreeNode root){
        List<TreeNode> list = new LinkedList<>();
        if (root != null){
            preOrderTraverse(root.getLeft());
            preOrderTraverse(root.getRight());
            list.add(root);
        }
        return list;
    }
    
```

### 4.层序遍历 访问顺序：先从跟节点层次开始，


        private static void levelOrderTraverse(TreeNode root){
        if (root == null) {
            return;
        }
        List<TreeNode> list = new LinkedList<>();
        LinkedList<TreeNode> queue = new LinkedList<TreeNode>();
        queue.add(root);
        TreeNode node;
        while (!queue.isEmpty()){
            node = queue.poll();
            list.add(node);
            if (node.getLeft() != null) {
                queue.add(node.getLeft());
            }
            if (node.getRight() != null) {
                queue.add(node.getRight());
            }
        }
    }



### 路径总和

给定一个二叉树和一个目标和，判断该树中是否存在根节点到叶子节点的路径，这条路径上所有节点值相加等于目标和。

方法：双队列，一个存储节点，一个存储从根到当前节点的Sum。 层序遍历二叉树，判断是否到达叶子节点，如果到达，则判读是否等于目标和。



```java
// 循环
public boolean hasPathSum(TreeNode root, int sum) {

        if(root == null) return false;

        LinkedList<TreeNode> queue = new LinkedList<>();
        LinkedList<Integer> values = new LinkedList<>();
        queue.offer(root);
        values.offer(root.val);
        TreeNode node;
        int temp;
        while(!queue.isEmpty()){
            node = queue.poll();
            temp = values.poll();

            if(node.right == null && node.left == null){
                if(temp == sum){
                    return true;
                }
                continue;
            }

            if(node.left != null){
                queue.offer(node.left);
                values.offer(temp + node.left.val);
            }
            if(node.right != null){
                queue.offer(node.right);
                values.offer(temp + node.right.val);
            }

        }

        return false;
    }
```



```java
    // 递归
    public boolean hasPathSum(TreeNode root, int sum) {

        if(root == null) return false;

        if(root.right == null && root.left == null){
            return sum == root.val;
        }


        return hasPathSum(root.left,sum - root.val) || hasPathSum(root.right,sum - root.val);
    }
```





### 完全二叉树判定

**方法：** 判断一个树是否属于完全二叉树可以从以下2个条件来判断：

- 任何一个结点如果右孩子不为空，左孩子却是空，则一定不是完全二叉树
- 当一个结点出现右孩子为空时候，判断该结点的层次遍历后继结点是否为叶子节点，如果全部都是叶子节点，则是完全二叉树，如果存在任何一个结点不是叶节点，则一定不是完全二叉树。

```java
    // 完全二叉树判定
    public static boolean isCompleteTree(TreeNode root){

        if (root == null){
            return false;
        }

        LinkedList<TreeNode> list = new LinkedList<>();
        list.offer(root);

        TreeNode node;
        TreeNode left;
        TreeNode right;
        boolean isLeaf = false;
        while (!list.isEmpty()){

            node = list.poll();
            left = node.left;
            right = node.right;

            if (isLeaf && (left == null || right == null)) return false; //开启叶节点判断标志位时，如果层次遍历中的后继结点不是叶节点 -> false

            if (null == left && null != right) return false;  //右孩子不等于空，左孩子等于空  -> false

            if (left != null){
                list.offer(left);
            }

            if (right != null){
                list.offer(right);
            }else{
                isLeaf = true;
            }


        }

        return true;
    }
```



### 搜索二叉树判定

二叉搜索树：如果一棵树为空树，那么是二叉搜索树；如果左子树的所有节点都小于根节点，右子树所有节点都大于根节点，那么是二叉搜索树

**方法：** 二叉搜索树是中序有序的，因为左子树<根<右子树

```
    public static boolean isBST(TreeNode root){
        Stack<TreeNode> stack = new Stack<>();

        TreeNode node = root;

        List<Integer> list = new ArrayList<>();

        while (stack.isEmpty() || node !=null){
            while (node != null){
                stack.push(node);
                node = node.left;
            }

            if (!stack.isEmpty()){
                node = stack.pop();
                list.add(node.value);
                node = node.right;
            }

        }

        for (int i = 1; i < list.size() ; i++) {

            if (list.get(i-1) > list.get(i)){
                return false;
            }
        }

        return true;

    }
```



### 查找第K小/大的元素

**方法：** 先将左子树压栈，然后弹栈，--k，取右节点

```
    // 利用二叉搜索树 查找第K小的元素
    public static TreeNode kSmallest(TreeNode bst, int k){

        Stack<TreeNode> stack = new Stack<>();

        while (bst!= null || !stack.isEmpty()){

            while (bst!= null){
                stack.push(bst);
                bst = bst.left;
            }

            bst = stack.pop();
            if (--k == 0) break;
            bst = bst.right;
        }
        
        return bst;

    }
```





### 二叉搜索树创建



```java
    public static TreeNode createBST(int[] tree){
        TreeNode bst = null;
        for(int key : tree) {
            bst = insert(bst,key);
        }
        return bst;
    }


    private static TreeNode insertBST(TreeNode root, int value){

        TreeNode node = new TreeNode(); // 待插入节点
        node.setValue(value);

        if (root == null){
            root = node;
        }else { // 寻找待插入节点的父节点

            TreeNode current = root;
            TreeNode parent;

            while (current != null){
                parent = current;
                if (current.value > value){ // 左子树
                    current = current.left;
                    if (current == null){
                        parent.left = node;
                        return root;
                    }

                }else {
                    current = current.right;
                    if (current == null){
                        parent.right = node;
                        return root;
                    }
                }
            }

        }



        return root;
    }
```





### 删除二叉搜索树

[二叉搜索树节点删除]: https://www.cnblogs.com/yahuian/p/10813614.html

二叉搜索树删除节点主要有三种情况：

- 要删除节点有零个孩子，即叶子节点
- 要删除节点有一个孩子
- 要删除节点有两个孩子

```
    private static boolean deleteBST(TreeNode root, int key){

        TreeNode current = root;

        TreeNode parent = new TreeNode();

        Boolean isRight = false;

        while (current.value != key){

            parent = current;
            if (parent.value < key){
                current = current.right;
                isRight = true;
            }else {
                current = current.left;
                isRight = false;
            }

            if (current == null){
                return false;
            }
        }

        // 1. 要删除节点有零个孩子，即叶子节点
        if (current.left == null && current.right == null){
            if (current == root){ // 如果当前要删除 节点为根节点
                root = null;
            }else {
                if (isRight){
                    parent.right = null;
                }else {
                    parent.left = null;
                }
            }
            //2.要删除节点有一个孩子
        }else if (current.left == null){
            if (current == root){
                root = current.right;
            }else if (isRight){
                parent.right = current.right;
            }else {
                parent.left = current.right;
            }
        }else if (current.right == null){
            if (current == root){
                root = current.left;
            }else if (isRight){
                parent.right =current.left;
            }else {
                parent.left = current.left;
            }
        }else{  // 3. 要删除节点有两个孩子

            TreeNode successor=getSuccessor(current);    //找到要删除结点的后继结点
            if(current==root)
                root=successor;
            else if(isRight)
                parent.right=successor;
            else
                parent.left=successor;

            successor.left=current.left;
            return true;

        }





        // 3. 要删除节点有两个孩子

        // 3.1 后继节点为待删除节点的右子


        return false;
    }

    //寻找要删除节点的中序后继结点
    private static TreeNode getSuccessor(TreeNode node){

        TreeNode successorParent = node;
        TreeNode successor = node;
        TreeNode current = node.right;
        //用来寻找后继结点
        while (current != null){
            successorParent = successor;
            successor = current;
            current = current.left;
        }

        //如果后继结点为要删除结点的右子树的左子，需要预先调整一下要删除结点的右子树
        if(successor!=node.right) {
            successorParent.left=successor.right;
            successor.right=node.right;
        }
        return successor;

    }
```



### 二叉搜索树查找

**原理：** 类似于二分查找法

```jAVA
    /**
     *  二叉查找树 - 查找
     * @param root
     * @param key
     * @return
     */
    public static boolean searchBST(TreeNode root, int key){

        if (root == null){
            return false;
        }else {
            if (root.value == key){
                return true;
            }
            if (root.value > key){
                searchBST(root.left,key);
            }else{
                searchBST(root.right,key);
            }
        }

        return false;
    }
```







