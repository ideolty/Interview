# 综述

文档记录了LeetCode上面有的剑指offer的题，后续在把书上的习题搬过来。

难度比起主站来说，不是简单了一点半点。



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



# [剑指 Offer 05. 替换空格](https://leetcode-cn.com/problems/ti-huan-kong-ge-lcof/)

请实现一个函数，把字符串 `s` 中的每个空格替换成"%20"。

**示例 1：**

```
输入：s = "We are happy."
输出："We%20are%20happy."
```

**限制：**

`0 <= s 的长度 <= 10000`



说实话，没太懂，一个个字符遍历过去不就可以了吗？或者直接使用replace的api，或者使用字符串splite，然后在把数组使用 %20 串起来。题目到底想考察什么内容？

```java
class Solution {
    public String replaceSpace(String s) {
        return s.replace(" ", "%20");
    }
}
执行用时：0 ms, 在所有 Java 提交中击败了100.00% 的用户
内存消耗：36.1 MB, 在所有 Java 提交中击败了92.56% 的用户
```



看了一下书的内容，本题应该是面对 C 语言来出的题，而且这里应该说明使用数组来存储每一个字符，并且要求原地修改。那么把空格替换的时候，会把一个长度的字符替换为3个。而解法的亮点为，**从后往前遍历**，这样后续的字符串就不需要进行重复的移动。



评论区

```c++
class Solution {
public:
    string replaceSpace(string s) {
        int count = 0, len = s.size();
        for (char c : s) {
            if (c == ' ') count++;
        }
        s.resize(len + 2 * count);
        for(int i = len - 1, j = s.size() - 1; i < j; i--, j--) {
            if (s[i] != ' ')
                s[j] = s[i];
            else {
                s[j - 2] = '%';
                s[j - 1] = '2';
                s[j] = '0';
                j -= 2;
            }
        }
        return s;
    }
};


作者：demigodliu
链接：https://leetcode-cn.com/problems/ti-huan-kong-ge-lcof/solution/tu-jie-guan-fang-tui-jian-ti-jie-ti-huan-3l74/
来源：力扣（LeetCode）
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```





# [剑指 Offer 06. 从尾到头打印链表](https://leetcode-cn.com/problems/cong-wei-dao-tou-da-yin-lian-biao-lcof/)

输入一个链表的头节点，从尾到头反过来返回每个节点的值（用数组返回）。

**示例 1：**

```
输入：head = [1,3,2]
输出：[2,3,1]
```

**限制：**

`0 <= 链表长度 <= 10000`



就很奇怪的题，因为解法非常多，这里缺少了限制条件。

- 首先可以考虑先做链表倒转，之后再遍历一次，这样只需要遍历两遍即可。

- 第二，由于是java可以使用 list ，算是动态数组，直接遍历链表，最后把数组倒转也是可以的。
- 当然也可以先遍历链表，得到链表长度，然后再建一个数组，倒着放数字就可以了。

这里我选择了第3种做法，感觉是更满足面试官的要求

```java
class Solution {
    public int[] reversePrint(ListNode head) {
        int count = 0;
        ListNode node = head;
        while (node != null){
            node = node.next;
            count++;
        }
        int[] nums = new int[count];
        for (int i = count - 1; i >= 0; i--){
            nums[i] = head.val;
            head = head.next;
        }
        return nums;
    }
}
执行用时：0 ms, 在所有 Java 提交中击败了100.00% 的用户
内存消耗：39 MB, 在所有 Java 提交中击败了77.76% 的用户
```



官方的是借助 “栈” 来实现的，那么同理递归也是可以的……





# [剑指 Offer 11. 旋转数组的最小数字](https://leetcode-cn.com/problems/xuan-zhuan-shu-zu-de-zui-xiao-shu-zi-lcof/)

把一个数组最开始的若干个元素搬到数组的末尾，我们称之为数组的旋转。输入一个递增排序的数组的一个旋转，输出旋转数组的最小元素。例如，数组 `[3,4,5,1,2]` 为 `[1,2,3,4,5]` 的一个旋转，该数组的最小值为1。 

**示例 1：**

```
输入：[3,4,5,1,2]
输出：1
```

**示例 2：**

```
输入：[2,2,2,0,1]
输出：0
```

注意：本题与主站 154 题相同：https://leetcode-cn.com/problems/find-minimum-in-rotated-sorted-array-ii/



主站的154是hard，这个为啥是easy？其实感觉和主站第81题解法一致

这个题，第一眼想法就是二分。考虑一般情况下

- 如果发生了旋转，那么最右的值一定是小于等于最左的。
- 然后中间值与最右值进行对比，确定最小值是在左边还是右边。

考虑最特殊的情况，数组全是一样的数字，那么二分也只能退化成遍历了。

```java
class Solution {
    public int minArray(int[] numbers) {
        // 如果没有旋转 直接返回第一个
        if (numbers[0] < numbers[numbers.length - 1]){
            return numbers[0];
        }
        return dfs(numbers, 0, numbers.length - 1);
    }
    
    private int dfs(int[] nums, int l, int r){
        if (l >= r){
            return nums[l];
        }

      	// low + (high - low) / 2 更加好一些，可以防止溢出
        int mid = (l + r) / 2;
        if (nums[mid] == nums[l] && nums[mid] == nums[r]){
            return Math.min(dfs(nums, l, mid - 1), dfs(nums, mid + 1, r));
        }

        if (nums[mid] <= nums[r]){
            return dfs(nums, l, mid);
        }else {
            return dfs(nums, mid + 1, r);
        }
    }
}
执行用时：0 ms, 在所有 Java 提交中击败了100.00% 的用户
内存消耗：38.3 MB, 在所有 Java 提交中击败了37.76% 的用户
```





官方思路差不多，但是使用迭代，减少了栈的开销，内存使用上来说应该会少很多。

```java
class Solution {
    public int minArray(int[] numbers) {
        int low = 0;
        int high = numbers.length - 1;
        while (low < high) {
            int pivot = low + (high - low) / 2;
            if (numbers[pivot] < numbers[high]) {
                high = pivot;
            } else if (numbers[pivot] > numbers[high]) {
                low = pivot + 1;
            } else {
                high -= 1;
            }
        }
        return numbers[low];
    }
}


作者：LeetCode-Solution
链接：https://leetcode-cn.com/problems/xuan-zhuan-shu-zu-de-zui-xiao-shu-zi-lcof/solution/xuan-zhuan-shu-zu-de-zui-xiao-shu-zi-by-leetcode-s/
来源：力扣（LeetCode）
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```



# [剑指 Offer 17. 打印从1到最大的n位数](https://leetcode-cn.com/problems/da-yin-cong-1dao-zui-da-de-nwei-shu-lcof/)

输入数字 `n`，按顺序打印出从 1 到最大的 n 位十进制数。比如输入 3，则打印出 1、2、3 一直到最大的 3 位数 999。

**示例 1:**

```
输入: n = 1
输出: [1,2,3,4,5,6,7,8,9]
```

说明：

- 用返回一个整数列表来代替打印
- n 为正整数



本题返回的都是int，不用考虑溢出，但是在面试的时候，主要考察大数，而大数考察的是字符串来做加减。



评论区

大数打印解法：

实际上，本题的主要考点是大数越界情况下的打印。需要解决以下三个问题：
1. 表示大数的变量类型：

    无论是 short / int / long ... 任意变量类型，数字的取值范围都是有限的。因此，大数的表示应用字符串 String 类型。

2. 生成数字的字符串集：

    使用 int 类型时，每轮可通过 +1 生成下个数字，而此方法无法应用至 String 类型。并且， String 类型的数字的进位操作效率较低，例如 "9999" 至 "10000" 需要从个位到千位循环判断，进位 4 次。

    观察可知，生成的列表实际上是 n 位 000 - 999 的 全排列 ，因此可避开进位操作，通过递归生成数字的 String 列表。

3. 递归生成全排列：

    基于分治算法的思想，先固定高位，向低位递归，当个位已被固定时，添加数字的字符串。例如当 n = 2时（数字范围 1−99 ），固定十位为 0-9 ，按顺序依次开启递归，固定个位 000 - 999 ，终止递归并添加数字字符串。

```java
class Solution {
    StringBuilder res;
    int nine = 0, count = 0, start, n;
    char[] num, loop = {'0', '1', '2', '3', '4', '5', '6', '7', '8', '9'};
    public String printNumbers(int n) {
        this.n = n;
        res = new StringBuilder();
        num = new char[n];
        start = n - 1;
        dfs(0);
        res.deleteCharAt(res.length() - 1);
        return res.toString();
    }
    void dfs(int x) {
        if(x == n) {
            String s = String.valueOf(num).substring(start);
            if(!s.equals("0")) res.append(s + ",");
            if(n - start == nine) start--;
            return;
        }
        for(char i : loop) {
            if(i == '9') nine++;
            num[x] = i;
            dfs(x + 1);
        }
        nine--;
    }
}


作者：jyd
链接：https://leetcode-cn.com/problems/da-yin-cong-1dao-zui-da-de-nwei-shu-lcof/solution/mian-shi-ti-17-da-yin-cong-1-dao-zui-da-de-n-wei-2/
来源：力扣（LeetCode）
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```







# [剑指 Offer 18. 删除链表的节点](https://leetcode-cn.com/problems/shan-chu-lian-biao-de-jie-dian-lcof/)

给定单向链表的头指针和一个要删除的节点的值，定义一个函数删除该节点。

返回删除后的链表的头节点。

**注意：**此题对比原题有改动

示例 1:
```
输入: head = [4,5,1,9], val = 5
输出: [4,1,9]
解释: 给定你链表中值为 5 的第二个节点，那么在调用了你的函数之后，该链表应变为 4 -> 1 -> 9.
```
示例 2:
```
输入: head = [4,5,1,9], val = 1
输出: [4,5,9]
解释: 给定你链表中值为 1 的第三个节点，那么在调用了你的函数之后，该链表应变为 4 -> 5 -> 9.
```
**说明：**

- 题目保证链表中节点的值互不相同
- 若使用 C 或 C++ 语言，你不需要 `free` 或 `delete` 被删除的节点



没啥好说的，基本的链表操作

```java
class Solution {
    public ListNode deleteNode(ListNode head, int val) {
        ListNode node = new ListNode(Integer.MIN_VALUE);
        node.next = head;

        ListNode pre = node;
        while(head != null){
            if(head.val == val){
                pre.next = head.next;
                head.next = null;
                break;
            }
            pre = head;
            head = head.next;
        }

        return node.next;
    }
}
执行用时：0 ms, 在所有 Java 提交中击败了100.00% 的用户
内存消耗：37.8 MB, 在所有 Java 提交中击败了68.78% 的用户
```



# [剑指 Offer 20. 表示数值的字符串](https://leetcode-cn.com/problems/biao-shi-shu-zhi-de-zi-fu-chuan-lcof/)

请实现一个函数用来判断字符串是否表示数值（包括整数和小数）。

数值（按顺序）可以分成以下几个部分：
1. 若干空格
2. 一个 小数 或者 整数
3. （可选）一个 'e' 或 'E' ，后面跟着一个 整数
4. 若干空格

小数（按顺序）可以分成以下几个部分：
1. （可选）一个符号字符（'+' 或 '-'）
2. 下述格式之一：
	1. 至少一位数字，后面跟着一个点 '.'
	2. 至少一位数字，后面跟着一个点 '.' ，后面再跟着至少一位数字
	3. 一个点 '.' ，后面跟着至少一位数字

整数（按顺序）可以分成以下几个部分：
1. （可选）一个符号字符（'+' 或 '-'）
2. 至少一位数字

部分数值列举如下：

- `["+100", "5e2", "-123", "3.1416", "-1E-16", "0123"]`

部分非数值列举如下：

- `["12e", "1a3.14", "1.2.3", "+-5", "12e+5.4"]`

 

示例 1：
```
输入：s = "0"
输出：true
```
示例 2：
```
输入：s = "e"
输出：false
```
示例 3：
```
输入：s = "."
输出：false
```
示例 4：
```
输入：s = "    .1  "
输出：true
```


提示：
- 1 <= s.length <= 20
- s 仅含英文字母（大写和小写），数字（0-9），加号 '+' ，减号 '-' ，空格 ' ' 或者点 '.' 。



题目就是想考察写代码的能力吧，这和算法没啥关系啊，用正则估计会更好。

别做，这种有限状态自动机的题目就TM是个坑，而且题目里面的情况还没有给全，在做完之后，准确率下降率1个点。














# [剑指 Offer 21. 调整数组顺序使奇数位于偶数前面](https://leetcode-cn.com/problems/diao-zheng-shu-zu-shun-xu-shi-qi-shu-wei-yu-ou-shu-qian-mian-lcof/)

输入一个整数数组，实现一个函数来调整该数组中数字的顺序，使得所有奇数位于数组的前半部分，所有偶数位于数组的后半部分。

**示例：**

```
输入：nums = [1,2,3,4]
输出：[1,3,2,4] 
注：[3,1,2,4] 也是正确的答案之一。
```



**提示：**

1. `0 <= nums.length <= 50000`
2. `1 <= nums[i] <= 10000`



数组原地替换，左右双指针，左边的遇到偶数则停止，右边的遇到奇数则停止，然后交换，当指针相交的时候就终止。

```java
class Solution {
    public int[] exchange(int[] nums) {
        if (nums.length == 0) return nums;

      	// 这里边界定了 -1 需要注意边界等问题
        int l = -1;
        int r = nums.length;
        while (l < r){
            while (l < r && l < nums.length - 1 && nums[++l] % 2 == 1);

            while (l < r && nums[--r] % 2 == 0);

            int tmp = nums[l];
            nums[l] = nums[r];
            nums[r] = tmp;
        }
        return nums;
    }
}
执行用时：2 ms, 在所有 Java 提交中击败了98.10% 的用户
内存消耗：46 MB, 在所有 Java 提交中击败了96.10% 的用户
```



评论区

解法一：首尾双指针

- 定义头指针 leftleftleft ，尾指针 rightrightright .

- left 一直往右移，直到它指向的值为偶数

- right 一直往左移， 直到它指向的值为奇数

- 交换 nums[left] 和 nums[right] .

- 重复上述操作，直到 left==right .

  

解法二：快慢双指针

- 定义快慢双指针 fast 和 low ，fast 在前， low 在后 .

- fast 的作用是向前搜索奇数位置，low 的作用是指向下一个奇数应当存放的位置

- fast 向前移动，当它搜索到奇数时，将它和 nums[low] 交换，此时 low 向前移动一个位置 .

- 重复上述操作，直到 fast 指向数组末尾 .

  

```c++
class Solution {
public:
    vector<int> exchange(vector<int>& nums) {
        int low = 0, fast = 0;
        while (fast < nums.size()) {
            if (nums[fast] & 1) {
                swap(nums[low], nums[fast]);
                low ++;
            }
            fast ++;
        }
        return nums;
    }
};


作者：huwt
链接：https://leetcode-cn.com/problems/diao-zheng-shu-zu-shun-xu-shi-qi-shu-wei-yu-ou-shu-qian-mian-lcof/solution/ti-jie-shou-wei-shuang-zhi-zhen-kuai-man-shuang-zh/
来源：力扣（LeetCode）
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```





# [剑指 Offer 26. 树的子结构](https://leetcode-cn.com/problems/shu-de-zi-jie-gou-lcof/)

输入两棵二叉树A和B，判断B是不是A的子结构。(约定空树不是任意一个树的子结构)

B是A的子结构， 即 A中有出现和B相同的结构和节点值。

例如:

```
给定的树 A:
  	 3
    / \
   4   5
  / \
 1   2
 
给定的树 B：
    4 
  /
 1
```

返回 true，因为 B 与 A 的一个子树拥有相同的结构和节点值。

**示例 1：**

```
输入：A = [1,2,3], B = [3,1]
输出：false
```

**示例 2：**

```
输入：A = [3,4,5,1,2], B = [4,1]
输出：true
```

**限制：**

- 0 <= 节点个数 <= 10000



还是dfs遍历，需要注意不要重复遍历。把每一个节点都看成是根节点，本课树是否匹配就看左右子树是否匹配，如果不匹配就继续从下一个节点开始尝试。

```java
class Solution {
    public boolean isSubStructure(TreeNode A, TreeNode B) {
        if (B == null || A == null){
            return false;
        }
        return dfs(A, B, true);
    }

    private boolean dfs(TreeNode A, TreeNode B, boolean isRoot){
        if (A == null && B == null){
            return true;
        }

        if (A == null){
            return false;
        }

        if (B == null){
            return true;
        }

        if (A.val == B.val){
            boolean left = dfs(A.left, B.left, false);
            boolean right = dfs(A.right, B.right, false);
            if (left && right){
                return true;
            }
        }

      	// 增加一个标志位 用来判断本轮是不是根节点 如果是在判断子树则不需要再继续递归
        if (isRoot){
            boolean left = dfs(A.left, B, true);
            if (left) return true;
            return dfs(A.right, B, true);
        }

        return false;
    }
}
执行用时：0 ms, 在所有 Java 提交中击败了100.00% 的用户
内存消耗：40.1 MB, 在所有 Java 提交中击败了76.22% 的用户
```



评论高赞

1. 先序遍历树 A 中的每个节点 $n_A$ ；（对应函数 isSubStructure(A, B)）
2. 判断树 A 中 以 $n_A$ 为根节点的子树 是否包含树 B 。（对应函数 recur(A, B)）

```java
class Solution {
    public boolean isSubStructure(TreeNode A, TreeNode B) {
        return (A != null && B != null) && (recur(A, B) || isSubStructure(A.left, B) || isSubStructure(A.right, B));
    }
    boolean recur(TreeNode A, TreeNode B) {
        if(B == null) return true;
        if(A == null || A.val != B.val) return false;
        return recur(A.left, B.left) && recur(A.right, B.right);
    }
}


作者：jyd
链接：https://leetcode-cn.com/problems/shu-de-zi-jie-gou-lcof/solution/mian-shi-ti-26-shu-de-zi-jie-gou-xian-xu-bian-li-p/
来源：力扣（LeetCode）
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```



# [剑指 Offer 50. 第一个只出现一次的字符](https://leetcode-cn.com/problems/di-yi-ge-zhi-chu-xian-yi-ci-de-zi-fu-lcof/)

在字符串 s 中找出第一个只出现一次的字符。如果没有，返回一个单空格。 s 只包含小写字母。

**示例:**

```
s = "abaccdeff"
返回 "b"

s = "" 
返回 " "
```

**限制：**

- 0 <= s 的长度 <= 50000



```java
class Solution {
    public char firstUniqChar(String s) {
        Map<Character, Integer> map = new HashMap<>();
        char[] chars = s.toCharArray();
        for (int i = 0; i < chars.length; i++) {
            char c = chars[i];
            Integer index = map.get(c);
            if (index != null){
                chars[i] = ' ';
                chars[index] = ' ';
            }else {
                map.put(c, i);
            }
        }

        for (char c : chars) {
            if (c != ' ') return c;
        }

        return ' ';
    }
}
执行用时：14 ms, 在所有 Java 提交中击败了74.22% 的用户
内存消耗：38.8 MB, 在所有 Java 提交中击败了63.15% 的用户
```

这种实在是太慢了。