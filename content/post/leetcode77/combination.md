---
title: "Combination"
date: 2020-09-08T10:25:55+08:00
draft: false
summary: "leetcode 77组合数"
tags: [leetcode]
---

```C++
/*
给定两个整数 n 和 k，返回 1 ... n 中所有可能的 k 个数的组合。

示例:

输入: n = 4, k = 2
输出:
[
  [2,4],
  [3,4],
  [2,3],
  [1,2],
  [1,3],
  [1,4],
]
思路： 典型的DFS题目，不需要建立vis数组进行访问控制，数组的按序遍历就是vis数组访问控制
*/

class Solution {
public:
    // DFS 
    vector<vector<int>> combine(int n, int k) {
        if (n < k) {
            return vector<vector<int>>();
        }
        vector<vector<int>> res;
        vector<int> tmp;
        DFS(res, tmp, 0, n, k);
        return res;
    }

    void DFS(vector<vector<int>>& res, vector<int>& tmp, int start, int n, int k) {
        if (k == tmp.size()) {
            res.push_back(tmp);
            return;
        }
        if (tmp.size()+n-start+1 < k) {
            return;
        }
        for(int i = start; i < n; i++) {
            tmp.push_back(i+1);
            DFS(res, tmp, i+1, n, k);
            tmp.pop_back();
        }
    }
};
```

