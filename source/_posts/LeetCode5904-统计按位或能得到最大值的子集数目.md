---
title: LeetCode5904 统计按位或能得到最大值的子集数目
date: 2021-10-17 20:01:29
tags: [算法, 状态压缩, LeetCode]
categories: [ 算法]
about:
---
[LeetCode5904 统计按位或能得到最大值的子集数目](https://leetcode-cn.com/problems/count-number-of-maximum-bitwise-or-subsets/)
> 周赛题目，我没有写出来，中等难度，学习一下大佬的题解。

```C++
class Solution {
public:
    int countMaxOrSubsets(vector<int>& nums) {
        int count = 0;
        int maxvalue = -1;
        for(int i=0;i<(1<<nums.size());i++)
        {
            int res = 0;
            for(int v=0;v<nums.size();v++)
            {
                if(i&(1<<v)) res |= nums[v];
            }
            if(res>maxvalue)
            {
                maxvalue = res;
                count = 0;
            }
            if(res==maxvalue)count++;
        }
        return count;
    }
};
```
> 看完状态压缩是什么就会写了，甚至感觉还有点简单。
> 状态压缩表示每一个可能的状态，res |= nums[v]获取最大值
> 如果存在最大值 计数++ 如果超过最大值 更新最大值