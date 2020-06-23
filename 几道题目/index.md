# 几道题目


### Codeforces 1153C(Round 551 Div2) Serval and Parenthesis Sequence ###

题目链接：http://codeforces.com/problemset/problem/1153/C

题目大意：给定一个由'(', ')', '?'组成的字符串s，要求填充'?'为'('或')'使得从s[0]开始的任意长度的前缀不是一个正确的序列（空串和s本身不算前缀），而s是一个正确的序列。

思路：要满足任意前缀不是正确的序列，第一个括号要跟最后一个括号匹配，也就是填充'?'使s[1]~s[n-2]是一个正确的序列。记'('对平衡程度的贡献是+1，')'为-1，一个序列只要满足任意前缀的平衡程度始终大于或等于0且序列本身的平衡程度为0，那么这个序列就是正确的。为了满足条件，我们可以使前面的'('尽可能的多。

代码比较简单就不贴了~

### Codeforces 1151C(Round 553 Div2) Problem for Nazar ###

题目链接：http://codeforces.com/problemset/problem/1151/C

题目大意：将从1开始的整数按照{1},{2,4},{3,5,7,9},...的顺序排列（第n个序列的长度是2<sup>n</sup>，n从0开始，奇偶奇偶奇偶...）。现在给定l和r(1≤l≤r≤10<sup>18</sup>) ，要求第l个数到第r个数之和，结果对1000000007取余。
    
思路：简单的数学题，没啥好说的。。。就是代码写起来有点麻烦。规律比较好找，分析一下可以求出$l$到$r$上奇数的个数和偶数的个数，直接用求和公式就行了。
    
代码：代码写得素质极差就不贴了。。。

### Codeforces 1151D(Round 553 Div2) Stas and the Queue at the Buffet ###

题目链接：http://codeforces.com/problemset/problem/1151/D

题目大意：有n个学生，对于第i个学生有两个属性：a<sub>i</sub>和b<sub>i</sub>。假设第i个学生排在第j个位置，他的不满程度是a<sub>i</sub> * 站在他左边的人数 + b<sub>i</sub> * 站在他右边的人数。现在要求安排每个学生的位置使总的不满程度最低。

思路：还是数学题，设第i个学生的位置是p<sub>i</sub>，则总的不满程度：$$s = \sum_{i=0}^{n-1} a_i * p_i + b_i * (n-1-p_j) = \sum_{i=0}^{n-1} (a_i - b_i) * p_i + b_i * (n-1)$$
s只跟(a<sub>i</sub> - b<sub>i</sub>) * p<sub>i</sub>相关，所以直接把(a<sub>i</sub> - b<sub>i</sub>)按从大到小的顺序排列就行了。

代码：
```cpp
#include <iostream>
#include <stdio.h>
#include <algorithm>
#include <string.h>
#include <math.h>

#define MAXN 100005

using namespace std;

//long long

typedef struct {
    int a, b;
    int sub;
}S;

S s[MAXN];

bool cmp(S a, S b)
{
    return a.sub > b.sub;
}

int main()
{
    int n;
    cin>>n;
    for(int i = 0; i < n; i++)
    {
        cin>>s[i].a>>s[i].b;
        s[i].sub = s[i].a - s[i].b;
    }
    sort(s, s + n, cmp);
    long long ret = 0;
    for(int i = 0; i < n; i++)
        ret += (long long)s[i].a * i + (long long)s[i].b * (n - i - 1);
    cout<<ret<<endl;
    return 0;
}
```

### Codeforces 1151E(Round 553 #Div2) Number of Components ###

题目链接：http://codeforces.com/problemset/problem/1151/E

题目大意：一个无环无向连通图有n个顶点($1\leq{n}\leq10^5$)，第i个顶点与第i+1个顶点相连,每个顶点都有对应的属性值。给定$l$和$r$，用$f(l,r)$表示删去属性值不在$l$到$r$的范围内的顶点后连通块的个数。要求计算$\sum_{l=1}^n\sum_{r=l}^n f(l,r)$的值。


