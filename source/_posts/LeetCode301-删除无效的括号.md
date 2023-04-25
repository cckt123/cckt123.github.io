---
title: LeetCode301-删除无效的括号
date: 2021-10-27 16:05:35
tags: [算法, 回溯, BFS, LeetCode]
categories: [ 算法]
about:
---
# [删除无效的括号](https://leetcode-cn.com/problems/remove-invalid-parentheses/)
>写这篇博客的时候，我之前博客内容并没有更完，但是我要集中注意力来处理手里的题目。
>这道题的难度是Hard，被我当成阅读理解处理了，下面的是我自己的理解与总结，可能不够细致，如果需要点开上面的题目，在后面找到官方题解会有帮助。

## 思路1 回溯+减枝
>回溯的本质即是暴力。
>暴力检索所有可能性，检出每一个可能合理的剔除方案，然后检查方案的合理性。
>首先检出可以合理相消的括号，计算剩余的括号。
>相消规则为同时存在左右括号，"()"就可以相消。
### 相消后情况
1. "((("  如果左括号数量大于右括号数量会导致情况1
2. ")))"  如果右括号数量大于左括号数量会导致情况2
3. "))(((" 如果左括号数量为0，右括号数量大于0，会导致情况3
### 回溯
>回溯统计左右括号数量，暴力尝试删除相消后对应数量的左右括号，删除后的字符串判断其合法性是否加入最后结果。
### 剪枝
1. 当Left（被删除的左括号数量）与Right为0时，停止回溯，进行合法性判断。
2. 当后续字符串长度小于Left+Right时，终止判断。

```C#
public class Solution {
    private IList<string> res = new List<string>();

    public IList<string> RemoveInvalidParentheses(string s) {
        int lremove = 0;
        int rremove = 0;

        for (int i = 0; i < s.Length; i++) {
            if (s[i] == '(') {
                lremove++;
            } else if (s[i] == ')') {
                if (lremove == 0) {
                    rremove++;
                } else {
                    lremove--;
                }
            }
        }
        Helper(s, 0, lremove, rremove);

        return res;
    }

    private void Helper(String str, int start, int lremove, int rremove) {
        if (lremove == 0 && rremove == 0) {
            if (IsValid(str)) {
                res.Add(str);
            }
            return;
        }

        for (int i = start; i < str.Length; i++) {
            if (i != start && str[i] == str[i - 1]) {
                continue;
            }
            // 如果剩余的字符无法满足去掉的数量要求，直接返回
            if (lremove + rremove > str.Length - i) {
                return;
            }
            // 尝试去掉一个左括号
            if (lremove > 0 && str[i] == '(') {
                Helper(str.Substring(0, i) + str.Substring(i + 1), i, lremove - 1, rremove);
            }
            // 尝试去掉一个右括号
            if (rremove > 0 && str[i] == ')') {
                Helper(str.Substring(0, i) + str.Substring(i + 1), i, lremove, rremove - 1);
            }
        }
    }

    private bool IsValid(String str) {
        int cnt = 0;
        for (int i = 0; i < str.Length; i++) {
            if (str[i] == '(') {
                cnt++;
            } else if (str[i] == ')') {
                cnt--;
                if (cnt < 0) {
                    return false;
                }
            }
        }

        return cnt == 0;
    }
}

作者：LeetCode-Solution
链接：https://leetcode-cn.com/problems/remove-invalid-parentheses/solution/shan-chu-wu-xiao-de-gua-hao-by-leetcode-9w8au/
来源：力扣（LeetCode）
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```