+++
title = "【poj 1694】An Old Stone Game"
date = 2019-12-19 17:51:23
slug = "201912191751"

[taxonomies]
tags = ["动态规划", "树形 DP"]
+++

一题简单的树形 DP

<!-- more -->

## 题目描述

There is an old stone game, played on an arbitrary general tree `T`. The goal is to put one stone on the root of `T` observing the following rules:

1.  At the beginning of the game, the player picks `K` stones and puts them all in one bucket.
2.  At each step of the game, the player can pick one stone from the bucket and put it on any empty leaf.
3.  When all of the `r` immediate children of a node p each has one stone, the player may remove all of these `r` stones, and put one of the stones on `p`. The other `r - 1` stones are put back into the bucket, and can be used in the later steps of the game.

The player wins the game if by following the above rules, he succeeds to put one stone on the root of the tree.<br>
You are to write a program to determine the least number of stones to be picked at the beginning of the game (`K`), so that the player can win the game on the given input tree.

## 输入格式

The input describes several trees. The first line of this file is `M`, the number of trees (`1 <= M <= 10`). Description of these `M` trees comes next in the file. Each tree has `N < 200` nodes, labeled `1, 2, ... N`, and each node can have any possible number of children. Root has label `1`. Description of each tree starts with `N` in a separate line. The following `N` lines describe the children of all nodes in order of their labels. Each line starts with a number `p` (`1 <= p <= N`, the label of one of the nodes), `r` the number of the immediate children of `p`, and then the labels of these `r` children.

## 输出格式

One line for each input tree showing the minimum number of stones to be picked in step 1 above, in order to win the game on that input tree.

## 输入样例

```txt
2
7
1 2 2 3
2 2 5 4
3 2 6 7
4 0
5 0
6 0
7 0
12
1 3 2 3 4
2 0
3 2 5 6
4 3 7 8 9
5 3 10 11 12
6 0
7 0
8 0
9 0
10 0
11 0
12 0
```

## 输出样例

```txt
3
4
```

## 题解

树形 DP<br>
`F(p)` 表示 `p` 结点最少需要的石子数

```cpp
#include <cstdio>
#include <cstring>
using namespace std;
const int MAXN = 205;
int T, N;
int Head[MAXN];
int A[MAXN];
int To[MAXN], Next[MAXN], EL;
inline void AddEdge(int p, int to) {
    To[++EL] = to;
    Next[EL] = Head[p];
    Head[p] = EL;
}
int F[MAXN];
void DP(int p) {
    if (!A[p]) { F[p] = 1; return; }
    int a[MAXN];
    for (int i = Head[p], j = 1; i; i = Next[i], j++) {
        DP(To[i]);
        a[j] = F[To[i]];
    }
    sort(a + 1, a + A[p] + 1);
    for (int i = 2; i <= A[p]; i++)
        a[i] = max (a[i], a[i - 1] + 1);
    F[p] = a[A[p]];
}
int main() {
    scanf("%d", &T);
    while (T--) {
        scanf("%d", &N);
        memset(Head, 0, sizeof(Head));
        EL = 0;
        for (int i = 1; i <= N; i++) {
            int p, n;
            scanf("%d%d", &p, &n);
            A[p] = n;
            while (n--) {
                int to;
                scanf("%d", &to);
                AddEdge(p, to);
            }
        }
        memset(F, 0x0f, sizeof(F));
        DP(1);
        printf("%d\n", F[1]);
    }
    return 0;
}
```
