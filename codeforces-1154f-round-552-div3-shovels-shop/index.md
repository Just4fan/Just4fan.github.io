# Codeforces 1154F(Round 552 Div3) - Shovels Shop


先填个坑~ 上次做div3的时候没有做出来，感觉好像不是一道特别难的题目

题目链接：http://codeforces.com/problemset/problem/1154/F

题目大意：
有n把铲子，记第i把的价格为c<sub>i</sub>。同时给你m个special offers，对于优惠j，用(x<sub>j</sub>, y<sub>j</sub>)表示选择x<sub>j</sub>把铲子购买，其中价格最便宜的y<sub>j</sub>把铲子免费，每个优惠可以被使用任意次数（包括0），但是每次购买只能使用一个优惠或不使用优惠。现在要求你买k把铲子（可以分多次购买），问最小花费。

输入：
第一行输入三个整数n,m 和 k (1≤n,m≤2⋅10<sup>5</sup>,1≤k≤min(n,2000)) —— 分别表示有n把铲子、m个优惠、需要购买k把铲子。
第二行输入n个整数a<sub>1</sub>,a<sub>2</sub>,…,a<sub>n</sub>  (1≤a<sub>i</sub>≤2⋅10<sup>5</sup>)，表示n把铲子的价格。
接下来的m行每行输入两个整数x<sub>j</sub>和y<sub>j</sub> (1≤y<sub>j</sub>≤x<sub>j</sub>≤n) 表示第j个优惠。
     
输出：
一个整数，表示购买k把铲子的最小花费。
    
思路：
总体思路是贪心+动态规划，首先可以确定在最小花费的情况下，购买的k把铲子一定是n把铲子中价格最便宜的k把，可以这么理解：任意取k把铲子，先按价格排序，假设通过使用优惠，购买其中的第i1,i2,...,im (m <= k)把铲子达到最小花费c，那么对于k把价格最便宜的铲子，通过使用优惠购买其中的第i1,i2,...,im (m <= k)把铲子，花费必定小于等于c。
这样问题就变成对于k把铲子，怎么组合使用优惠可以使花费最小。首先明确一个优惠怎么使用是最优的：当免费的价格最大时最优，所以我们可以把k把铲子按价格从高到低排序。用dp[i]表示购买前i把铲子的最小花费，显然dp[i]可以从dp[i - 1]（不使用优惠）或dp[i - offer[j].x]得到（使用优惠j），状态转移方程就不写了。对于每个i枚举所有的优惠，时间复杂度是O(k*m)。

代码：
```cpp
#include <iostream>
#include <stdio.h>
#include <algorithm>
#include <string.h>

#define MAXN 200005
#define MAXM 200005
#define MAXK 2001

using namespace std;

typedef struct {
    int x, y;
}Offer;

int a[MAXN];
Offer o[MAXM];
long long dp[MAXN];
long long sum[MAXN];

bool cmp(int a, int b)
{
    return a > b;
}

int main()
{
    int n, m, k;
    cin>>n>>m>>k;
    for(int i = 1; i <= n; i++)
        cin>>a[i];
    for(int i = 0; i < m; i++)
        cin>>o[i].x>>o[i].y;
    sort(a + 1, a + 1 + n, cmp);
    sum[n - k] = 0;
    for(int i = n - k + 1; i <= n; i++)
    {
        sum[i] = sum[i - 1] + a[i];
        dp[i] = sum[i];
    }
    dp[n - k] = 0;
    for(int i = n - k; i <= n; i++)
    {
        for(int j = 0; j < m; j++)
        {
            if(i + o[j].x <= n)
                dp[i + o[j].x] = min(dp[i + o[j].x], dp[i] + sum[i + o[j].x - o[j].y] - sum[i]);
            else
                dp[n] = min(dp[n], dp[i] + sum[n] - sum[i]);
        }
    }
    cout<<dp[n]<<endl;
    return 0;
}

```

跑了一下1965ms，有点极限……

