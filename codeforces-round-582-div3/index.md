# Codeforces Round 582 (Div 3)


[题目链接](http://codeforces.com/contests)

### A. Chips Moving

题目大意：对一个序列的每一个元素进行±1或±2的操作，其中±2的操作不计算代价，求最小代价使序列的每一个元素都相同。

思路： 求奇数个数和偶数个数的最小值。

代码：

```cpp
#include <stdio.h>
#include <iostream>
#include <algorithm>

#define MAXN 105

using namespace std;

int a[MAXN];

int main()
{
    int n;
    cin >> n;
    int num = 0;
    for (int i = 0; i < n; i++)
    {
        int a;
        cin >> a;
        cout << (a & 1) << " ";
        if ((a & 1) == 0)
            ++num;
    }
    cout << min(num, n - num) << endl;
    return 0;
}
```

### B. Bad Prices

题目大意：给定一个序列，求满足下列条件的元素的个数：在其后面的元素中存在比它小的元素。

思路：逆序遍历序列，记录最小值，跟最小值比较并记录。

代码：

```cpp
#include <stdio.h>
#include <iostream>
#include <algorithm>

#define MAXN 150001

using namespace std;

int a[MAXN];

int main()
{
    int t;
    cin >> t;
    while (t--)
    {
        int n;
        cin >> n;
        for (int i = 0; i < n; i++)
            cin >> a[i];
        int min = 1e6 + 1;
        int cnt = 0;
        for (int i = n - 1; i >= 0; i--)
            if (a[i] <= min)
                min = a[i];
            else
                ++cnt;
        cout << cnt << endl;
    }
    return 0;
}
```

### C. Book Reading

题目大意：给定n和m，对于满足以下条件的所有整数i：i ≤ n 且 i mod m 余0，求所有i的个位数相加的结果。

思路：满足i mod m == 0的整数是m, 2m, 3m, ...，不难发现当km的个位数为0时，(k+1)m, (k+2)m, (k+3)m,...的个位数与m, 2m, 3m, ...相等，所以对于每个m只需要找出这个最小的周期就可以了，由于只需要考虑个位数，可以先把0~9的周期计算一下，例如2的周期是5,3的周期是10，另外再算一下周期内个位数之和。最后求出周期数和处理一下余数就可以了。（代码写复杂了）

代码：

```cpp
#include <stdio.h>
#include <iostream>
#include <algorithm>

using namespace std;

long long table[] = {1, 10, 5, 10, 5, 2, 5, 10, 5, 10};
long long table_sum[10];

int init()
{
    for (int i = 1; i < 10; i++)
        for (int j = 1; j < table[i]; j++)
            table_sum[i] += (j * i) % 10;
    // for (int i = 0; i < 9; i++)
    //     cout << table_sum[i] << " ";
    // cout << endl;
}

int main()
{
    int t;
    init();
    cin >> t;
    while (t--)
    {
        long long n, m;
        cin >> n >> m;
        if (n < m)
        {
            cout << 0 << endl;
            continue;
        }
        int last = m % 10;
        if (last == 0)
        {
            cout << 0 << endl;
            continue;
        }
        long long num = table[last] * m;
        long long ret = n / num * table_sum[last];
        long long mod = n % num;
        for (long long i = m; i <= mod; i += m)
            ret += i % 10;
        cout << ret << endl;
    }
    return 0;
}
```

### D. Equalizing by Division

题目大意：对于元素a<sub>i</sub>,进行如下操作a<sub>i</sub>:=⌊a<sub>i</sub>/2⌋(向下取整)，对于每个元素这种操作可以进行任意次数，问最少需要多少次这样的操作可以让序列中有k个相等的元素。

思路：感觉挺有意思的一道题，首先对于元素a<sub>i</sub>，对2×a<sub>i</sub>和2×a<sub>i</sub>+1进行上述操作都可以得到2×a<sub>i</sub>。可以看成一个二叉树，对于所有结点x，求出以结点x为根的子树中高度最靠近x的k个结点（贪心的思想，实际上子树中子结点跟根结点的高度差就是改子结点变换为根结点所需操作的次数）。找出拥有最小值的结点即可。

代码：

```cpp
#include <stdio.h>
#include <iostream>
#include <algorithm>
#include <math.h>

#define MAXN (int)2e5 + 1
#define MAXD 18
#define INF 0x3f3f3f3f

using namespace std;

int a[MAXN];
int num[MAXN];
int ret = 0;
int dl, dr;

int cal_range(int root, int root_depth, int depth)
{
    int l = root << (depth - root_depth);
    int r = l + (1 << (depth - root_depth)) - 1;
    l = min(MAXN, l);
    r = min(MAXN - 1, r);
    return (num[r] - num[l - 1]);
}

int find_ret(int i, int depth, int k)
{
    int root_depth = depth;
    ++depth;
    k -= a[i];
    int ret = 0;
    while (depth <= MAXD && k > 0)
    {
        int num = cal_range(i, root_depth, depth);
        ret += (depth - root_depth) * (num > k ? k : num);
        k -= num;
        ++depth;
    }
    if (k > 0)
        return INF;
    return ret;
}

void solve(int k)
{
    int ret = INF;
    for (int i = 1; i <= MAXN; i++)
    {
        int depth = (int)log2(i) + 1;
        ret = min(ret, find_ret(i, depth, k));
    }
    cout << ret << endl;
}

int main()
{
    int n, k;
    fill(a, a + MAXN, 0);
    cin >> n >> k;
    for (int i = 0; i < n; i++)
    {
        int x;
        cin >> x;
        ++a[x];
    }
    num[0] = 0;
    for (int i = 1; i <= MAXN; i++)
        num[i] = num[i - 1] + a[i];
    solve(k);
    return 0;
}
```

### E. Two Small Strings

题目大意：给定两个长度为2的子串，每个子串由'a','b','c'组成，可以重复，给定n，要求构造一个字符串由n个‘a',n个'b',n个'c'组成，并且不含给定的子串。

思路：水过去的，没有思路。。。

代码：

```cpp
#include <stdio.h>
#include <iostream>
#include <algorithm>
#include <math.h>
#include <string>

#define MAXN (int)1e5 + 1
#define INF 0x3f3f3f3f

using namespace std;

int main()
{
    int n;
    cin >> n;
    string sub[2];
    for (int i = 0; i < 2; i++)
        cin >> sub[i];
    // n = 2;
    // string sub_list[9];
    string s = "abc";
    // int index = 0;
    // for (int i = 0; i < 3; i++)
    //     for (int j = 0; j < 3; j++)
    //     {
    //         string tmp = string();
    //         tmp += s[i];
    //         tmp += s[j];
    //         sub_list[index++] = tmp;
    //     }
    // for (int i = 0; i < 9; i++)
    //     for (int j = i + 1; j < 9; j++)
    //     {
    //         string sub[2];
    //         sub[0] = sub_list[i];
    //         sub[1] = sub_list[j];
    //         bool find = false;
    //         s = "abc";
    //         do
    //         {
    //             string tmp = string(s);
    //             if (n > 1)
    //                 tmp = tmp + s[0];
    //             bool ok = true;
    //             for (int i = 0; i < 2; i++)
    //             {
    //                 if (tmp.find(sub[i]) != string::npos)
    //                 {
    //                     ok = false;
    //                     break;
    //                 }
    //             }
    //             if (ok)
    //             {
    //                 find = true;
    //                 break;
    //             }
    //         } while (next_permutation(s.begin(), s.end()));
    //         if (find)
    //         {
    //             string sub = string(s);
    //             s = "";
    //             for (int i = 0; i < n; i++)
    //                 s += sub;
    //             // cout << "YES" << endl;
    //             // cout << s << endl;
    //         }
    //         else
    //         {
    //             cout << "NO" << endl;
    //             cout << sub[0] << endl;
    //             cout << sub[1] << endl;
    //         }
    //     }
    bool find = false;
    do
    {
        string tmp = string(s);
        if (n > 1)
            tmp = tmp + s[0];
        bool ok = true;
        for (int i = 0; i < 2; i++)
        {
            if (tmp.find(sub[i]) != string::npos)
            {
                ok = false;
                break;
            }
        }
        if (ok)
        {
            find = true;
            break;
        }
    } while (next_permutation(s.begin(), s.end()));
    if (find)
    {
        string sub = string(s);
        s = "";
        for (int i = 0; i < n; i++)
            s += sub;
        cout << "YES" << endl;
        cout << s << endl;
    }
    else
    {
        string s = "abc";
        bool find = false;
        do
        {
            string tmp = string(s);
            bool ok = true;
            for (int i = 0; i < 2; i++)
            {
                if (tmp.find(sub[i]) != string::npos)
                {
                    ok = false;
                    break;
                }
            }
            if (ok)
            {
                find = true;
                break;
            }
        } while (next_permutation(s.begin(), s.end()));
        if (find)
        {
            string sub = string(s);
            s = "";
            for (int j = 0; j < 3; j++)
                for (int i = 0; i < n; i++)
                    s += sub[j];
            cout << "YES" << endl;
            cout << s << endl;
        }
        else
            cout << "NO" << endl;
    }
    return 0;
}
```
