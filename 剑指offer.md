# 综述

文档记录了LeetCode上面有的剑指offer的题，后续在把书上的习题搬过来



# [剑指 Offer 03. 数组中重复的数字](https://leetcode-cn.com/problems/shu-zu-zhong-zhong-fu-de-shu-zi-lcof/) :star:

找出数组中重复的数字。


在一个长度为 n 的数组 nums 里的所有数字都在 0～n-1 的范围内。数组中某些数字是重复的，但不知道有几个数字重复了，也不知道每个数字重复了几次。请找出数组中任意一个重复的数字。

示例 1：

```
输入：
[2, 3, 1, 0, 2, 5, 3]
输出：2 或 3 
```



限制：

- 2 <= n <= 100000



解法一：排序后顺序遍历

解法二：使用一个set

**解法三：借助数组下标进行原地替换。**数组的值就是对应下标的值，如果发现目标下标上有重复，那么直接得出结论

```java
class Solution {
    public int findRepeatNumber(int[] nums) {
        if(nums.length == 2){
            if(nums[0] == nums[1]){
                return nums[0];
            }
            return -1;
        }

        Arrays.sort(nums);
        for(int i = 0; i < nums.length - 1; i++){
            if(nums[i] == nums[i + 1]){
                return nums[i];
            }
        }

        return -1;
    }
}
执行用时：3 ms, 在所有 Java 提交中击败了57.24% 的用户
内存消耗：45.8 MB, 在所有 Java 提交中击败了97.02% 的用户
```



评论区

如果没有重复数字，那么正常排序后，数字i应该在下标为i的位置，所以思路是重头扫描数组，遇到下标为i的数字如果不是i的话，（假设为m),那么我们就拿与下标m的数字交换。在交换过程中，如果有重复的数字发生，那么终止返回ture

```java
class Solution {
    public int findRepeatNumber(int[] nums) {
        int temp;
        for(int i=0;i<nums.length;i++){
            while (nums[i]!=i){
                if(nums[i]==nums[nums[i]]){
                    return nums[i];
                }
                temp=nums[i];
                nums[i]=nums[temp];
                nums[temp]=temp;
            }
        }
        return -1;
    }
}


作者：derrick_sun
链接：https://leetcode-cn.com/problems/shu-zu-zhong-zhong-fu-de-shu-zi-lcof/solution/yuan-di-zhi-huan-shi-jian-kong-jian-100-by-derrick/
来源：力扣（LeetCode）
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```





# [剑指 Offer 04. 二维数组中的查找](https://leetcode-cn.com/problems/er-wei-shu-zu-zhong-de-cha-zhao-lcof/) :star:

在一个 n * m 的二维数组中，每一行都按照从左到右递增的顺序排序，每一列都按照从上到下递增的顺序排序。请完成一个高效的函数，输入这样的一个二维数组和一个整数，判断数组中是否含有该整数。

 

示例:

现有矩阵 matrix 如下：

```
[
  [1,   4,  7, 11, 15],
  [2,   5,  8, 12, 19],
  [3,   6,  9, 16, 22],
  [10, 13, 14, 17, 24],
  [18, 21, 23, 26, 30]
]

```

给定 target = `5`，返回 `true`。

给定 target = `20`，返回 `false`。



**限制：**

- 0 <= n <= 1000
- 0 <= m <= 1000

**注意：**本题与主站 240 题相同：https://leetcode-cn.com/problems/search-a-2d-matrix-ii/



与 [74. 搜索二维矩阵](https://leetcode-cn.com/problems/search-a-2d-matrix/) 有点相似，但是做法不同，不能直接二分。但是这个从左上到右下是递增的，所以可以先往右走，当大于target了就往下走，遇见了就是true，更简单一些。

```java
class Solution {
    public boolean findNumberIn2DArray(int[][] matrix, int target) {
        if (matrix.length == 0 || matrix[0].length == 0){
            return false;
        }

        if (target < matrix[0][0]){
            return false;
        }

        if (target == matrix[0][0]){
            return true;
        }
        int n = matrix.length;
        int m = matrix[0].length;

        int row = 0;
        int col = 0;
        while (row + 1 < n){
            if (matrix[row + 1][col] == target){
                return true;
            }
            if (matrix[row + 1][col] < target){
                if (row == n - 2){
                    row++;
                    break;
                }
                row++;
            }else {
                break;
            }
        }

        for (int i = row; i >= 0; i--){
            while (col + 1 < m){
                if (matrix[i][col + 1] == target){
                    return true;
                }
                if (matrix[i][col + 1] < target){
                    if (col == m - 2){
                        col++;
                        break;
                    }
                    col++;
                }else {
                    break;
                }
            }
        }
        return false;
    }
}
执行用时：0 ms, 在所有 Java 提交中击败了100.00% 的用户
内存消耗：44 MB, 在所有 Java 提交中击败了81.73% 的用户
```

还是考察的边界条件的处理，我这个代码写的实在是太丑陋了，思路不清晰，所以代码写的相当的乱，而且从左开始遍历也不好处理边界。



官方

方法一：暴力

**方法二：线性查找**

由于给定的二维数组具备每行从左到右递增以及每列从上到下递增的特点，当访问到一个元素时，可以排除数组中的部分元素。

**从二维数组的右上角开始查找**。如果当前元素等于目标值，则返回 true。如果当前元素大于目标值，则移到左边一列。如果当前元素小于目标值，则移到下边一行。

可以证明这种方法不会错过目标值。如果当前元素大于目标值，说明当前元素的下边的所有元素都一定大于目标值，因此往下查找不可能找到目标值，往左查找可能找到目标值。如果当前元素小于目标值，说明当前元素的左边的所有元素都一定小于目标值，因此往左查找不可能找到目标值，往下查找可能找到目标值。

- 若数组为空，返回 `false`
- 初始化行下标为 0，列下标为二维数组的列数减 1
- 重复下列步骤，直到行下标或列下标超出边界
  - 获得当前下标位置的元素 num
  - 如果 num 和 target 相等，返回 true
  - 如果 num 大于 target，列下标减 1
  - 如果 num 小于 target，行下标加 1
- 循环体执行完毕仍未找到元素等于 `target` ，说明不存在这样的元素，返回 `false`

```java
class Solution {
    public boolean findNumberIn2DArray(int[][] matrix, int target) {
        if (matrix == null || matrix.length == 0 || matrix[0].length == 0) {
            return false;
        }
        int rows = matrix.length, columns = matrix[0].length;
        int row = 0, column = columns - 1;
        while (row < rows && column >= 0) {
            int num = matrix[row][column];
            if (num == target) {
                return true;
            } else if (num > target) {
                column--;
            } else {
                row++;
            }
        }
        return false;
    }
}


作者：LeetCode-Solution
链接：https://leetcode-cn.com/problems/er-wei-shu-zu-zhong-de-cha-zhao-lcof/solution/mian-shi-ti-04-er-wei-shu-zu-zhong-de-cha-zhao-b-3/
来源：力扣（LeetCode）
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```

