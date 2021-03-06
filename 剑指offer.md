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

```java
public class PrintNumbers17 {
    public static void main(String[] args) {
        PrintNumbers17 printNumbers17 = new PrintNumbers17();
        List<StringBuilder> list = printNumbers17.printNumbers(3);
        int divide = 1;
        for (int i = 0; i < list.size(); i++) {
            int leftRemoveNum = 3 - getSize(i);
            System.out.println(list.get(i).substring(leftRemoveNum));
        }
    }

    public static int getSize(long number) {
        int count = 0;
        while (number > 0) {
            count += 1;
            number = (number / 10);
        }
        return count;
    }

    public List<StringBuilder> printNumbers(int n) {
        return dfs(n, n);
    }

    private List<StringBuilder> dfs(int level, int n){
        List<StringBuilder> result = new ArrayList<>();
        if (1 == level){
            for (int i = 0; i <= 9; i++){
                result.add(new StringBuilder().append(i));
            }
            return result;
        }

        List<StringBuilder> list = dfs(level - 1, n);
        for (int i = 0; i <= 9; i++){
            for (StringBuilder stringBuilder : list) {
                result.add(new StringBuilder().append(i).append(stringBuilder));
            }
        }
        return result;
    }
}
```



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



# [剑指 Offer 27. 二叉树的镜像](https://leetcode-cn.com/problems/er-cha-shu-de-jing-xiang-lcof/)

请完成一个函数，输入一个二叉树，该函数输出它的镜像。

例如输入：

     			4
        /   \
      2     7
     / \   / \
    1   3 6   9

镜像输出：

         4
        /   \
      7     2
     / \   / \
    9   6 3   1
**示例 1：**

```
输入：root = [4,2,7,1,3,6,9]
输出：[4,7,2,9,6,3,1]
```

限制：

- 0 <= 节点个数 <= 1000

注意：本题与主站 226 题相同：https://leetcode-cn.com/problems/invert-binary-tree/



题目主要主要迭代或者递归的时候注意一下，如果是在原树上改，不要覆盖了节点。由于题目没说要原地修改，那么当然是新来一棵树更简单。

```java
class Solution {
    public TreeNode mirrorTree(TreeNode root) {
        if (root == null) return null;
        TreeNode target = new TreeNode(root.val);
        dfs(root, target);
        return target;
    }


    private void dfs(TreeNode source, TreeNode target){
        if (source == null) return;

        target.val = source.val;
        if (source.left != null){
            target.right = new TreeNode();
        }
        if (source.right != null){
            target.left = new TreeNode();
        }

        dfs(source.left, target.right);
        dfs(source.right, target.left);
    }
}
执行用时：0 ms, 在所有 Java 提交中击败了100.00% 的用户
内存消耗：35.8 MB, 在所有 Java 提交中击败了51.07% 的用户
```









# [剑指 Offer 32 - I. 从上到下打印二叉树](https://leetcode-cn.com/problems/cong-shang-dao-xia-da-yin-er-cha-shu-lcof/)

从上到下打印出二叉树的每个节点，同一层的节点按照从左到右的顺序打印。

例如:
 给定二叉树: `[3,9,20,null,null,15,7]`,

```
    3
   / \
  9  20
    /  \
   15   7
```

返回：

```
[3,9,20,15,7]
```

**提示：**

1. `节点总数 <= 1000`



```java
class Solution {
    public int[] levelOrder(TreeNode root) {
        List<Integer> list = new ArrayList<>();
        Queue<TreeNode> queue = new ArrayDeque<>();
        if (root == null) return new int[0];

        queue.offer(root);
        while (!queue.isEmpty()){
            TreeNode node = queue.poll();
            list.add(node.val);
            if (node.left != null){
                queue.offer(node.left);
            }
            if (node.right != null){
                queue.offer(node.right);
            }
        }

        int[] nums = new int[list.size()];
        for (int i = 0; i < list.size(); i++) {
            nums[i] = list.get(i);
        }

        return nums;
    }
}
执行用时：2 ms, 在所有 Java 提交中击败了24.76% 的用户
内存消耗：38.4 MB, 在所有 Java 提交中击败了83.28% 的用户
```

为什么这么慢呢？



# [剑指 Offer 33. 二叉搜索树的后序遍历序列](https://leetcode-cn.com/problems/er-cha-sou-suo-shu-de-hou-xu-bian-li-xu-lie-lcof/)

输入一个整数数组，判断该数组是不是某二叉搜索树的后序遍历结果。如果是则返回 `true`，否则返回 `false`。假设输入的数组的任意两个数字都互不相同。

 

这个就是考察后续遍历的性质。

- 最后一个节点一定是根节点。
- 由于是二叉搜索树，那么比根小的一定是在左子树。遍历前面的数字，找到第一个比根大的数字，后面的都是右子树。
- 递归判断左右子树是不是二叉搜索树。

```java
class Solution {
    public boolean verifyPostorder(int[] postorder) {
        return isBinaryTress(postorder, 0, postorder.length - 1);

    }

    private boolean isBinaryTress(int[] postorder, int left, int right){
        if (left >= right){
            return true;
        }

        int root = postorder[right];
        int rightIndex = right;
        for (int i = left; i < right; i++){
            if (postorder[i] > root){
                rightIndex = i;
                break;
            }
        }

        for (int i = rightIndex + 1; i < right; i++){
            if (postorder[i] < root){
                return false;
            }
        }

        return isBinaryTress(postorder, left, rightIndex - 1) && isBinaryTress(postorder, rightIndex, right - 1);
    }
}
执行用时：0 ms, 在所有 Java 提交中击败了100.00% 的用户
内存消耗：36 MB, 在所有 Java 提交中击败了32.09% 的用户
```



评论区写的超级漂亮的代码

```java
class Solution {
    public boolean verifyPostorder(int[] postorder) {
        return recur(postorder, 0, postorder.length - 1);
    }
    boolean recur(int[] postorder, int i, int j) {
        if(i >= j) return true;
        int p = i;
        while(postorder[p] < postorder[j]) p++;
        int m = p;
        while(postorder[p] > postorder[j]) p++;
        return p == j && recur(postorder, i, m - 1) && recur(postorder, m, j - 1);
    }
}


作者：jyd
链接：https://leetcode-cn.com/problems/er-cha-sou-suo-shu-de-hou-xu-bian-li-xu-lie-lcof/solution/mian-shi-ti-33-er-cha-sou-suo-shu-de-hou-xu-bian-6/
来源：力扣（LeetCode）
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```







# [剑指 Offer 34. 二叉树中和为某一值的路径](https://leetcode-cn.com/problems/er-cha-shu-zhong-he-wei-mou-yi-zhi-de-lu-jing-lcof/)

输入一棵二叉树和一个整数，打印出二叉树中节点值的和为输入整数的所有路径。从树的根节点开始往下一直到叶节点所经过的节点形成一条路径。

 

示例:
给定如下二叉树，以及目标和 target = 22，

```
              5
             / \
            4   8
           /   / \
          11  13  4
         /  \    / \
        7    2  5   1
```

返回:

```
[
   [5,4,11,2],
   [5,8,4,5]
]
```

**提示：**

1. `节点总数 <= 10000`



题目没说明白，是必须要从根节点到叶子节点的完整链路的和等于目标值才算吗？节点居然还能出现负数？？剪枝也没法剪了，d到底吧

```java
class Solution {
    private List<List<Integer>> result = new ArrayList<>();

    public List<List<Integer>> pathSum(TreeNode root, int target) {
        if (root == null) return result;
        dfs(root, target, new ArrayList<>());
        return result;
    }

    private void dfs(TreeNode node, int target, List<Integer> list){
        if (node.left == null && node.right == null){
            if (target == node.val){
                list.add(node.val);
                result.add(new ArrayList<>(list));
                list.remove(list.size() - 1);
            }
            return;
        }

        if (node.left != null){
            list.add(node.val);
            dfs(node.left, target - node.val, list);
            list.remove(list.size() - 1);
        }
        if (node.right != null){
            list.add(node.val);
            dfs(node.right, target - node.val, list);
            list.remove(list.size() - 1);
        }
    }
}
执行用时：1 ms, 在所有 Java 提交中击败了100.00% 的用户
内存消耗：38.9 MB, 在所有 Java 提交中击败了46.41% 的用户
```





# [剑指 Offer 36. 二叉搜索树与双向链表](https://leetcode-cn.com/problems/er-cha-sou-suo-shu-yu-shuang-xiang-lian-biao-lcof/)

输入一棵二叉搜索树，将该二叉搜索树转换成一个排序的循环双向链表。要求不能创建任何新的节点，只能调整树中节点指针的指向。

![img](截图/leetCode/bstdlloriginalbst.png)

我们希望将这个二叉搜索树转化为双向循环链表。链表中的每个节点都有一个前驱和后继指针。对于双向循环链表，第一个节点的前驱是最后一个节点，最后一个节点的后继是第一个节点。

下图展示了上面的二叉搜索树转化成的链表。“head” 表示指向链表中有最小元素的节点。

![img](截图/leetCode/bstdllreturndll.png)

特别地，我们希望可以就地完成转换操作。当转化完成以后，树中节点的左指针需要指向前驱，树中节点的右指针需要指向后继。还需要返回链表中的第一个节点的指针。



观察两张图，可以发现这就是一个中序遍历后的结果，然后再额外维护一下双向指针就可以了。题目说是要求就地转换，但是没有说不可以使用额外的节点。

```java
class Solution {
    private Node head = new Node(-1);
    private Node current = head;

    public Node treeToDoublyList(Node root) {
        if (root == null) return null;

        dfs(root);
        // 补全双向的链接
        Node pre = current;
        current = head.right;
        while (current.right != null){
            current.left = pre;
            pre = current;
            current = current.right;
        }
        current.left = pre;
        current.right = head.right;
        return head.right;
    }

    private void dfs(Node node){
        if (node == null){
            return;
        }

        dfs(node.left);
        current.right = node;
        current = node;
        dfs(node.right);
    }
}
执行用时：0 ms, 在所有 Java 提交中击败了100.00% 的用户
内存消耗：37.6 MB, 在所有 Java 提交中击败了92.43% 的用户
```



评论区高赞

两步合为了一步，在递归的同时维护了双向指针。

```java
class Solution {
    Node pre, head;
    public Node treeToDoublyList(Node root) {
        if(root == null) return null;
        dfs(root);
        head.left = pre;
        pre.right = head;
        return head;
    }
    void dfs(Node cur) {
        if(cur == null) return;
        dfs(cur.left);
        if(pre != null) pre.right = cur;
        else head = cur;
        cur.left = pre;
        pre = cur;
        dfs(cur.right);
    }
}


作者：jyd
链接：https://leetcode-cn.com/problems/er-cha-sou-suo-shu-yu-shuang-xiang-lian-biao-lcof/solution/mian-shi-ti-36-er-cha-sou-suo-shu-yu-shuang-xian-5/
来源：力扣（LeetCode）
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```



# [剑指 Offer 38. 字符串的排列](https://leetcode-cn.com/problems/zi-fu-chuan-de-pai-lie-lcof/) :star:

输入一个字符串，打印出该字符串中字符的所有排列。

你可以以任意顺序返回这个字符串数组，但里面不能有重复元素。

示例:

```
输入：s = "abc"
输出：["abc","acb","bac","bca","cab","cba"]
```

限制：

`1 <= s 的长度 <= 8`



与主站第47题一样，这里给出一个不一样的评论区高赞。

他这边主要是进行交换，使得数组的始终分为已使用与未使用的两部分。

```java
class Solution {
    List<String> res = new LinkedList<>();
    char[] c;
    public String[] permutation(String s) {
        c = s.toCharArray();
        dfs(0);
        return res.toArray(new String[res.size()]);
    }
    void dfs(int x) {
        if(x == c.length - 1) {
            res.add(String.valueOf(c));      // 添加排列方案
            return;
        }
        HashSet<Character> set = new HashSet<>();
        for(int i = x; i < c.length; i++) {
            if(set.contains(c[i])) continue; // 重复，因此剪枝
            set.add(c[i]);
            swap(i, x);                      // 交换，将 c[i] 固定在第 x 位
            dfs(x + 1);                      // 开启固定第 x + 1 位字符
            swap(i, x);                      // 恢复交换
        }
    }
    void swap(int a, int b) {
        char tmp = c[a];
        c[a] = c[b];
        c[b] = tmp;
    }
}


作者：jyd
链接：https://leetcode-cn.com/problems/zi-fu-chuan-de-pai-lie-lcof/solution/mian-shi-ti-38-zi-fu-chuan-de-pai-lie-hui-su-fa-by/
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



# [剑指 Offer 53 - I. 在排序数组中查找数字 I](https://leetcode-cn.com/problems/zai-pai-xu-shu-zu-zhong-cha-zhao-shu-zi-lcof/)

统计一个数字在排序数组中出现的次数。

**示例 1:**

```
输入: nums = [5,7,7,8,8,10], target = 8
输出: 2
```

**示例 2:**

```
输入: nums = [5,7,7,8,8,10], target = 6
输出: 0
```

**限制：**

```
0 <= 数组长度 <= 50000
```

注意：本题与主站 34 题相同（仅返回值不同）：https://leetcode-cn.com/problems/find-first-and-last-position-of-element-in-sorted-array/



简单题，先二分然后再双指针。

```java
class Solution {
    public int search(int[] nums, int target) {
        int lo = 0;
        int hi = nums.length - 1;
        int shot = -1;

        while (lo <= hi){
            int mid = lo + (hi - lo) / 2;
            if (target == nums[mid]){
                shot = mid;
                break;
            }
            if (target > nums[mid]){
                lo = mid + 1;
            }else {
                hi = mid - 1;
            }
        }
        if (shot == -1){
            return 0;
        }

        int count = 1;
        lo = shot;
        hi = shot;
        while (++hi < nums.length && nums[hi] == nums[shot]){
            count++;
        }
        while (--lo >= 0 && nums[lo] == nums[shot]){
            count++;
        }
        return count;
    }
}
执行用时：0 ms, 在所有 Java 提交中击败了100.00% 的用户
内存消耗：41.1 MB, 在所有 Java 提交中击败了91.80% 的用户
```







# [剑指 Offer 56 - I. 数组中数字出现的次数](https://leetcode-cn.com/problems/shu-zu-zhong-shu-zi-chu-xian-de-ci-shu-lcof/) :cry:

一个整型数组 `nums` 里除两个数字之外，其他数字都出现了两次。请写程序找出这两个只出现一次的数字。要求时间复杂度是O(n)，空间复杂度是O(1)。

**示例 1：**

```
输入：nums = [4,1,4,6]
输出：[1,6] 或 [6,1]
```

**示例 2：**

```
输入：nums = [1,2,10,4,1,4,3,3]
输出：[2,10] 或 [10,2]
```

**限制：**

- `2 <= nums.length <= 10000`



空间复杂度是O(1)，所以排除map。也不能先排序，因为时间复杂度不允许。那么就可以借助数组下标，例如`a[1] = 5` 那么我们就把这个 5 移动到 a[4] 里面去，如果 a[4] 里面的值已经是 5 了，那么就改放一个占位符 例如Integer.MAX_VALUE之类的。一般来说是ok了，本题，可能会存在数组下标越界的问题，`a[i] = 10000` 直接超过了数组长度。那么我的做法是想不出……

之后第二次遍历数组，找出不为占位符的数字即可。





官方

**方法一：分组异或**

思路

让我们先来考虑一个比较简单的问题：

- 如果除了一个数字以外，其他数字都出现了两次，那么如何找到出现一次的数字？

答案很简单：全员进行异或操作即可。考虑异或操作的性质：对于两个操作数的每一位，相同结果为 0，不同结果为 1。那么在计算过程中，成对出现的数字的所有位会两两抵消为 0，最终得到的结果就是那个出现了一次的数字。

那么这一方法如何扩展到找出两个出现一次的数字呢？

如果我们可以把所有数字分成两组，使得：

- 两个只出现一次的数字在不同的组中；
- 相同的数字会被分到相同的组中。

那么对两个组分别进行异或操作，即可得到答案的两个数字。这是解决这个问题的关键。

那么如何实现这样的分组呢？

记这两个只出现了一次的数字为 a 和 b，那么所有数字异或的结果就等于 a 和 b 异或的结果，我们记为 x。如果我们把 x 写成二进制的形式 $x_k x_{k - 1} \cdots x_2 x_1 x_0$，其中$x_i \in \{ 0, 1 \}$，我们考虑一下 $x_i = 0$ 和 $x_i = 1$ 的含义是什么？它意味着如果我们把 a 和 b 写成二进制的形式，$a_i$和 $b_i$ 的关系——$x_i = 1$ 表示 $a_i$ 和 $b_i$ 不等，$x_i = 0$ 表示 $a_i$和 $b_i$ 相等。假如我们任选一个不为 0 的 $x_i$，按照第 i 位给原来的序列分组，如果该位为 0 就分到第一组，否则就分到第二组，这样就能满足以上两个条件，为什么呢？

- 首先，两个相同的数字的对应位都是相同的，所以一个被分到了某一组，另一个必然被分到这一组，所以满足了条件 2。
- 这个方法在 $x_i$ 的时候 a 和 b 不被分在同一组，因为 $x_i$ 表示 $a_i$ 和 $b_i$ 不等，根据这个方法的定义「如果该位为 0 就分到第一组，否则就分到第二组」可以知道它们被分进了两组，所以满足了条件 1。

在实际操作的过程中，我们拿到序列的异或和 x 之后，对于这个「位」是可以任取的，只要它满足 $x_i$。但是为了方便，这里的代码选取的是「不为 0 的最低位」，当然你也可以选择其他不为 0 的位置。

至此，答案已经呼之欲出了。

算法

先对所有数字进行一次异或，得到两个出现一次的数字的异或值。

在异或结果中找到任意为 1 的位。

根据这一位对所有的数字进行分组。

在每个组内进行异或操作，得到两个数字。

```java
class Solution {
    public int[] singleNumbers(int[] nums) {
        int ret = 0;
        for (int n : nums) {
            ret ^= n;
        }
        int div = 1;
        while ((div & ret) == 0) {
            div <<= 1;
        }
        int a = 0, b = 0;
        for (int n : nums) {
            if ((div & n) != 0) {
                a ^= n;
            } else {
                b ^= n;
            }
        }
        return new int[]{a, b};
    }
}


作者：LeetCode-Solution
链接：https://leetcode-cn.com/problems/shu-zu-zhong-shu-zi-chu-xian-de-ci-shu-lcof/solution/shu-zu-zhong-shu-zi-chu-xian-de-ci-shu-by-leetcode/
来源：力扣（LeetCode）
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```







# [剑指 Offer 58 - I. 翻转单词顺序](https://leetcode-cn.com/problems/fan-zhuan-dan-ci-shun-xu-lcof/)

输入一个英文句子，翻转句子中单词的顺序，但单词内字符的顺序不变。为简单起见，标点符号和普通字母一样处理。例如输入字符串"I am a student. "，则输出"student. a am I"。

**示例 1：**

```
输入: "the sky is blue"
输出: "blue is sky the"
```

示例 2：

```
输入: "  hello world!  "
输出: "world! hello"
解释: 输入字符串可以在前面或者后面包含多余的空格，但是反转后的字符不能包括。
```

**示例 3：**

```
输入: "a good   example"
输出: "example good a"
解释: 如果两个单词间有多余的空格，将反转后单词间的空格减少到只含一个。
```

**说明：**

- 无空格字符构成一个单词。
- 输入字符串可以在前面或者后面包含多余的空格，但是反转后的字符不能包括。
- 如果两个单词间有多余的空格，将反转后单词间的空格减少到只含一个。

注意：本题与主站 151 题相同：https://leetcode-cn.com/problems/reverse-words-in-a-string/

注意：此题对比原题有改动



```java
class Solution {
    public String reverseWords(String s) {
        if (s == null ) return "";
        s = s.trim();
        if (s.length() == 0) return "";
        
        char[] chars = s.toCharArray();
        StringBuilder result = new StringBuilder();
        for (int i = chars.length - 1; i >= 0; i--) {
            StringBuilder stringBuilder = new StringBuilder();
            char c = chars[i];
            while (c != ' '){
                stringBuilder.append(c);
                if (i == 0){
                    break;
                }
                i--;
                c = chars[i];
            }

            if (stringBuilder.length() != 0){
                result.append(stringBuilder.reverse()).append(" ");
            }
        }
        result.deleteCharAt(result.length() - 1);
        return result.toString();
    }
}
执行用时：4 ms, 在所有 Java 提交中击败了42.61% 的用户
内存消耗：38.2 MB, 在所有 Java 提交中击败了74.11% 的用户
```



评论区

方法一：双指针

算法解析：

- 倒序遍历字符串 s ，记录单词左右索引边界 i , j ；
- 每确定一个单词的边界，则将其添加至单词列表 res ；
- 最终，将单词列表拼接为字符串，并返回即可。

```java
class Solution {
    public String reverseWords(String s) {
        s = s.trim(); // 删除首尾空格
        int j = s.length() - 1, i = j;
        StringBuilder res = new StringBuilder();
        while(i >= 0) {
            while(i >= 0 && s.charAt(i) != ' ') i--; // 搜索首个空格
            res.append(s.substring(i + 1, j + 1) + " "); // 添加单词
            while(i >= 0 && s.charAt(i) == ' ') i--; // 跳过单词间空格
            j = i; // j 指向下个单词的尾字符
        }
        return res.toString().trim(); // 转化为字符串并返回
    }
}


作者：jyd
链接：https://leetcode-cn.com/problems/fan-zhuan-dan-ci-shun-xu-lcof/solution/mian-shi-ti-58-i-fan-zhuan-dan-ci-shun-xu-shuang-z/
来源：力扣（LeetCode）
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```



# [剑指 Offer 58 - II. 左旋转字符串](https://leetcode-cn.com/problems/zuo-xuan-zhuan-zi-fu-chuan-lcof/)

字符串的左旋转操作是把字符串前面的若干个字符转移到字符串的尾部。请定义一个函数实现字符串左旋转操作的功能。比如，输入字符串"abcdefg"和数字2，该函数将返回左旋转两位得到的结果"cdefgab"。

**示例 1：**

```
输入: s = "abcdefg", k = 2
输出: "cdefgab"
```

**示例 2：**

```
输入: s = "lrloseumgh", k = 6
输出: "umghlrlose"
```

**限制：**

- `1 <= k < s.length <= 10000`



简单题，热个身

```java
class Solution {
    public String reverseLeftWords(String s, int n) {
        if (n > s.length()) return s;
        return s.substring(n) + s.substring(0, n);
    }
}
执行用时：0 ms, 在所有 Java 提交中击败了100.00% 的用户
内存消耗：38.1 MB, 在所有 Java 提交中击败了77.17% 的用户
```



评论区高赞的骚操作

**方法一：字符串切片**

**方法二：列表遍历拼接**

算法流程：

- 新建一个 list(Python)、StringBuilder(Java) ，记为 res ；
- 先向 res 添加 “第 n + 1 位至末位的字符” ；
- 再向 res 添加 “首位至第 n 位的字符” ；
- 将 res 转化为字符串并返回。

利用求余运算，可以简化代码。

```java
class Solution {
    public String reverseLeftWords(String s, int n) {
        StringBuilder res = new StringBuilder();
        for(int i = n; i < n + s.length(); i++)
            res.append(s.charAt(i % s.length()));
        return res.toString();
    }
}


作者：jyd
链接：https://leetcode-cn.com/problems/zuo-xuan-zhuan-zi-fu-chuan-lcof/solution/mian-shi-ti-58-ii-zuo-xuan-zhuan-zi-fu-chuan-qie-p/
来源：力扣（LeetCode）
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```

