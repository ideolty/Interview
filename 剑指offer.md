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

