---
layout: post
title: Codeforces Round 139 (Div. 2)
disqus: y
tags: [Codeforces, Div2]
---

A了两题 =A=。。。

###A. Dice Tower

喜闻乐见的骰子题，将骰子累起来，两个接触的面值不同，一个骰子两个相对的面和为7，每一个骰子判断就行了。

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


int main()
{
    int n, x, a, b, tmp;
    bool flag;
    while(cin >> n >> x)
    {
        flag = true;
        cin >> a >> b;
        tmp = 7 - x;
        for(int i = 2; i <= n; i++)
        {
            cin >> a >> b;
            if(a == tmp || b == tmp || 7 - a == tmp || 7 - b == tmp || a == x || b == x || 7 - a == x || 7 - b == x)
            {
                flag = false;
            }
        }
        if(flag) printf("YES\n");
        else printf("NO\n");
    }
}
{% endhighlight %}

###B. Well-known Numbers

一个类fibonacci数列,求出序列，然后倒序贪心求和位s就行。

注意求出序列的每个数不同O_o

{% highlight cpp linenos %}
//By Brickgao
#include <iostream>
#include <cstdio>
#include <cstring>
#include <cmath>
#include <cstdlib>
#include <algorithm>
#include <vector>
#define LL long long
using namespace std;

LL f[1000], k, s;

int main()
{
    vector <LL> rec;
    LL i;
    cin >> s >> k;
    f[0] = 0;
    f[1] = 1;
    for(i = 2; f[i - 1] < s; i++)
        for(LL j = i - 1; j >= 0 && j >= i - k; --j)
            f[i] += f[j];
    for(; i > 0 && s > 0; i--)
        if(f[i] <= s)
        {
            s -= f[i];
            rec.push_back(f[i]);
        }
    LL leng = rec.size();
    cout << leng << endl;
    for(i = 0; i < leng - 1; i++)
        cout << rec[i] << " ";
    cout << rec[leng - 1] << endl;
    return 0;
}
{% endhighlight %}

