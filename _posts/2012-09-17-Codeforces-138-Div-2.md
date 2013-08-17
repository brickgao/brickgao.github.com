---
layout: post
title: Codeforces Round 138 (Div. 2)
disqus: y
tags: [Codeforces, Div2]
---

目前A了四道

###A. Parallelepiped

简单模拟，给平行六面体三个面的面积，求出所有的棱长和。

{% highlight cpp linenos %}
//By Brickgao
#include <iostream>
#include <cstdio>
#include <cstring>
#include <cmath>

using namespace std;

int main()
{
    __int64 a, b, c, ans;
    while(cin >> a >> b >> c)
    {
        int tmp = sqrt(a * b * c);
        ans = (tmp / a + tmp / b + tmp / c) * 4;
        printf("%I64d\n", ans);
    }
    return 0;
}
{% endhighlight %}

###B. Array

求一个字串，使他的字串中有k个不同的数字，并且字串不可再化简，输出这个字串是从哪里到哪里的，找不到输出“-1 -1”，模拟。

{% highlight cpp linenos %}
//By Brickgao
#include <iostream>
#include <cstdio>
#include <cstring>
#include <cmath>

using namespace std;

int hap[100010];
int rec[100010];
int dif[100010];

int main()
{
    bool flag;
    int n, k, tol, ans, min, max;
    while(cin >> n >> k)
    {
        flag = false;
        tol = 0;
        memset(dif, -1, sizeof(dif));
        for(int i = 1; i <= n; i++)
            scanf("%d", &rec[i]);
        for(int i = 1; i <= n; i++)
        {
            if(dif[rec[i]] != -1)
            {
                dif[rec[i]] = i;
            }
            if(dif[rec[i]] == -1)
            {
                tol++;
                hap[tol] = rec[i];
                dif[rec[i]] = i;
            }
            if(tol == k)
            {
                flag = true;
                ans = i;
                break;
            }
        }
        if(flag)
        {
            max = -1;
            min = 100010;
            for(int i = 1; i <= k; i++)
            {
                if(dif[hap[i]] < min)
                    min = dif[hap[i]];
                if(dif[hap[i]] > max)
                    max = dif[hap[i]];
            }
            printf("%d %d\n", min, max);
        }
        else
        {
            printf("-1 -1\n");
        }
    }
    return 0;
}
{% endhighlight %}

###C. Bracket Sequence

求一个最大的字串，其中所有括号能匹配，并且方括号的数目最多，用栈来模拟。

{% highlight cpp linenos %}
//By Brickgao
#include <iostream>
#include <cstdio>
#include <cstring>
#include <cmath>
#include <string>
#include <stack>

using namespace std;

char s[100010];
typedef struct record {
        char ch;
        int po;
} rec;

int main()
{
        int leng, ans, l, r;
        rec tmp1, tmp2;
        int tmp3;
        bool flag;
        stack <rec> st;
        while(~scanf("%s", s))
        {
                flag = false;
                while(!st.empty())
                        st.pop();
                leng = strlen(s);
                for(int i = 0; i < leng; i++) 
                {
                    if(!st.empty())
                        tmp2 = st.top();
                    if(!st.empty() && ((tmp2.ch == '(' && s[i] == ')') || (tmp2.ch == '[' && s[i] == ']'))) 
                    {
                        st.pop();
                    } 
                    else 
                    {
                        tmp1.ch = s[i];
                        tmp1.po = i;
                        st.push(tmp1);
                    }
                }
                ans = 0;
                if(st.empty()) 
                {
                        for(int i = 0; i < leng; i++)
                                if(s[i] == '[') ans ++;
                        printf("%d\n%s\n", ans, s);
                }
                else
                {
                        bool once = true;
                        flag = false;
                        ans = -1;
                        int tmpr, tmpl;
                        while(st.size() != 0)
                        {
                                tmp3 = 0;
                                if(once)
                                {
                                    tmpr = leng - 1;
                                    tmpl = st.top().po + 1;
                                    once = false;
                                }
                                else
                                {
                                    if(st.size() != 1)
                                    {
                                        tmpr = st.top().po - 1;
                                        st.pop();
                                        tmpl = st.top().po + 1;
                                    }
                                    else
                                    {
                                        tmpr = st.top().po  - 1;
                                        tmpl = 0;
                                        st.pop();
                                    }
                                }
                                if(tmpl < tmpr)
                                {
                                    flag = true;
                                    for(int j = tmpl; j <= tmpr; j++)
                                        if(s[j] == '[') tmp3 ++;
                                }
                                if(tmp3 > ans)
                                {
                                        ans = tmp3;
                                        l = tmpl;
                                        r = tmpr;
                                }
                        }
                        if(flag)
                        {
                            printf("%d\n", ans);
                            for(int i = l; i <= r; i++)
                                    printf("%c", s[i]);
                            printf("\n");
                        }
                        else
                            printf("0\n");
                }

        }
        return 0;
}
{% endhighlight %}

###E. Partial Sums

= =完全没有找出规律...后来看到<a href = "http://blog.csdn.net/ACM_cxlove/">ACM_cxlove的Blog</a>才知道是找规律的...

举例：
N = 4的时候， 随着K不断增加

a1   a2                    a3                                        a4

a1   a1 + a2          a1 + a2 + a3                    a1 + a2 + a3 + a4

a1   2 * a1 + a2   3 * a1 + 2 * a2 + a3      4 * a1 + 3 * a2 + 2 * a3 + a4

a1   3 * a1 + a2   6 * a1 + 3 * a2 + a3     10 * a1 + 6 * a2 + 3 * a3 + a4

a1   4 * a1 + a2   10 * a1 + 4 * a2 + a3   20 * a1 + 10 * a2 + 4 * a3 + a4

a1   5 * a1 + a2   15 * a1 + 5 * a2 + a3   35 * a1 + 15 * a2 + 5 * a3 + a4
......

可以得出所有数由C(i + k - 2, i - 1) * ai得到...

inv[i]表示 i 对于 MOD 的逆元

则inv[i] = (- MOD / i * inv[MOD % i]);

这样通过递推O(1)就能求出每一个的逆元

证明这个的正确性

两边同时乘以i

inv[i] * i = i * ( - MOD / i * inv[MOD % i]);   (由逆元性质我们知道，a * inv[a] % MOD = 1)

i*(- MOD / i) == (MOD % i) % MOD      

-(MOD - MOD % i) == (MOD % i) % MOD

这是显然成立的，得证

貌似还有一种方法是快速幂来着，之后再看看。

=w=长姿势了...顺便这两天补补信安数学...

{% highlight cpp linenos %}
//By Brickgao
#include <iostream>
#include <cstdio>
#include <cstring>
#include <cmath>
#include <cstdlib>
#include <algorithm>
#include <vector>
using namespace std;
typedef long long LL;
#define mod 1000000007
#define N 2005

LL inv[N], c[N], a[N], n, k;

LL getc(int n, int m)
{
	LL ans = 1;
	for(int i = 1; i <= m; i++)
		ans = ((ans * inv[i]) % mod * (n - i + 1) % mod);
	return ans;
}	

int main()
{
	cin >> n >> k;
	for(int i = 0; i < n; i++) cin >> a[i];
	if(k == 0)
	{
		cout << a[0];
		for(int i = 1; i < n; i++)
			cout << " " << a[i];
		cout << endl;
		return 0;
	}
	inv[0] = inv[1] = 1;
	for(int i = 2; i < N; i++)
		inv[i] = (((- mod / i * inv[mod % i]) % mod) + mod) % mod;
	for(int i = 0; i < n; i++)
		c[i] = getc(i + k - 1, i);
	for(int i = 0; i < n; i++)
	{
		LL ans = 0;
		for(int j = 0; j <= i; j++)
			ans = (ans + a[j] * c[i - j]) % mod;
		if(!i)
			cout << ans;
		else
			cout << " " << ans;
	}
	cout << endl;
    return 0;
}
{% endhighlight %}
