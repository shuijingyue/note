例题：颜色分类
给定一个包含红色、白色和蓝色，一共 n 个元素的数组，原地 对它们进行排序，使得相同颜色的元素相邻，并按照红色、白色、蓝色顺序排列。

此题中，我们使用整数 0、 1 和 2 分别表示红色、白色和蓝色。

示例 1：


输入：nums = [2,0,2,1,1,0]
输出：[0,0,1,1,2,2]
示例 2：


输入：nums = [2,0,1]
输出：[0,1,2]
示例 3：


输入：nums = [0]
输出：[0]
示例 4：


输入：nums = [1]
输出：[1]
提示：

n == nums.length
1 <= n <= 300
nums[i] 为 0、1 或 2
进阶：

你可以不使用代码库中的排序函数来解决这道题吗？
你能想出一个仅使用常数空间的一趟扫描算法吗？
📺 视频题解

思路分析：解决这道问题需要有快速排序 partition 的知识作为基础。

我们可以在区间上设置两个表示分界的位置，并且定义循环不变量：

所有的元素在区间 [0..zero) = 0；
所有的元素在区间 [zero..i) = 1；
区间 [i..two) 是程序没有看到的部分；
所有的元素在区间 [two..len - 1] = 2，这里 len 表示数组的长度。
这种定义下，为了让初始化的时候三个区间都为空区间，zero = 0，two = len，程序没有看到的部分是整个数组。程序什么时候终止呢？当 i == two 时，三个子区间正好覆盖了整个数组，程序没有看到的部分为空，因此循环可以继续的条件是：i < two 。其它的细节我们放在代码中。

参考代码 1：


import java.util.Arrays;


public class Solution {

    public void sortColors(int[] nums) {
        int len = nums.length;
        if (len < 2) {
            return;
        }
        
        int zero = 0;
        int two = len;
        int i = 0;
        while (i < two) {
            if (nums[i] == 0) {
                swap(nums, i, zero);
                zero++;
                i++;
            } else if (nums[i] == 1) {
                i++;
            } else {
                two--;
                swap(nums, i, two);
            }
        }
    }

    private void swap(int[] nums, int index1, int index2) {
        int temp = nums[index1];
        nums[index1] = nums[index2];
        nums[index2] = temp;
    }
}
复杂度分析：

时间复杂度：O(N)O(N)，这里 NN 是输入数组的长度；
空间复杂度：O(1)O(1)。
如果我们按照下面这种方式定义循环不变量：

所有的元素在区间 [0..zero] = 0；
所有的元素在区间 (zero..i) = 1；
区间 [i..two] 是程序没有看到的部分；
所有的元素在区间 (two..len - 1] = 2，这里 len 表示数组的长度。
这种定义下，为了让初始化的时候三个区间都为空区间，zero = -1，two = len - 1。程序什么时候终止呢？当 i == two + 1 时，三个子区间正好覆盖了整个数组，因此循环可以继续的条件是：i <= two 。其它的细节我们放在代码中。

参考代码 2：

Java

public class Solution {

    public void sortColors(int[] nums) {
        int len = nums.length;
        if (len < 2) {
            return;
        }
        int zero = -1;
        int two = len - 1;
        int i = 0;
        while (i <= two) {
            if (nums[i] == 0) {
                zero++;
                swap(nums, i, zero);
                i++;
            } else if (nums[i] == 1) {
                i++;
            } else {
                swap(nums, i, two);
                two--;
            }
        }
    }

    private void swap(int[] nums, int index1, int index2) {
        int temp = nums[index1];
        nums[index1] = nums[index2];
        nums[index2] = temp;
    }
}
复杂度分析：（同「参考代码 1」）。

总结：循环不变量是写对代码、分析边界条件的基础。

在循环变量 i 遍历的过程中，人为定义的循环不变的性质，决定了「初始化」「遍历过程」和「循环终止」条件。初始化的时候，变量的初始值需要保证三个区间为空区间，而循环终止的时候，循环变量 i 需要使得我们定义的三个区间覆盖整个数组。