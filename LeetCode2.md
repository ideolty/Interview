# [633. 平方数之和](https://leetcode-cn.com/problems/sum-of-square-numbers/)

给定一个非负整数 `c` ，你要判断是否存在两个整数 `a` 和 `b`，使得 $a^2 + b^2 = c$ 。

**示例 1：**

```
输入：c = 5
输出：true
解释：1 * 1 + 2 * 2 = 5
```

**提示：**

- $0 <= c <= 2^{31} - 1$



这个暴力的来，然后加上剪枝，也就是说$0 + \sqrt{c} ^ 2 = c$，那么b的最大值就是 $\sqrt{c}$ 了，然后b从0开始遍历，剪完之后对结果进行开方，如果开完之后是个整数，那么就得到了结果。

```java
class Solution {
    public boolean judgeSquareSum(int c) {
        for (int a = 0; a <= Math.sqrt(c); a++) {
            double b = c - a * a;
            b = Math.sqrt(b);
            if (b == (int)b){
                return true;
            }
        }
        return false;
    }
}
执行用时：7 ms, 在所有 Java 提交中击败了14.73% 的用户
内存消耗：35.1 MB, 在所有 Java 提交中击败了76.27% 的用户
```



评论区高赞

**基本分析**

根据等式 $a^2 + b^2 = c$，可得知 a 和 b 的范围均为 $[0,\sqrt{c}]$。

基于此我们会有以下几种做法。

**枚举**

我们可以枚举 `a` ，边枚举边检查是否存在 `b` 使得等式成立。

这样做的复杂度为 $O(\sqrt{c})$。

```java
class Solution {
    public boolean judgeSquareSum(int c) {
        int max = (int)Math.sqrt(c);
        for (int a = 0; a <= max; a++) {
            int b = (int)Math.sqrt(c - a * a);
            if (a * a + b * b == c) return true;
        }
        return false;
    }
}
```



双指针

由于 a 和 b 的范围均为 $[0,\sqrt{c}]$，因此我们可以使用「双指针」在 $[0,\sqrt{c}]$范围进行扫描：

- $a^2 + b^2 == c$ : 找到符合条件的 a 和 b，返回 true
- $a^2 + b^2 < c$ : 当前值比目标值要小，a++
- $a^2 + b^2 > c$ : 当前值比目标值要大，b--

```java
class Solution {
    public boolean judgeSquareSum(int c) {
        int a = 0, b = (int)Math.sqrt(c);
        while (a <= b) {
            int cur = a * a + b * b;
            if (cur == c) {
                return true;
            } else if (cur > c) {
                b--;
            } else {
                a++;
            }
        }
        return false;
    }
}


作者：AC_OIer
链接：https://leetcode-cn.com/problems/sum-of-square-numbers/solution/gong-shui-san-xie-yi-ti-san-jie-mei-ju-s-7qi5/
来源：力扣（LeetCode）
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```



**费马平方和**

费马平方和 : 奇质数能表示为两个平方数之和的充分必要条件是该质数被 4 除余 1 。

翻译过来就是：当且仅当一个自然数的质因数分解中，满足 4k+3 形式的质数次方数均为偶数时，该自然数才能被表示为两个平方数之和。
