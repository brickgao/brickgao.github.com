---
layout: post
title: Codeforces Round 138 (Div. 2)
category: tech
tags: [Codeforces, div2]
---

目前A了三道

###A. Parallelepiped

简单模拟，给平行六面体三个面的面积，求出所有的棱长和。

{% highlight cpp %}
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

{% highlight cpp %}
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

{% highlight cpp %}
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
