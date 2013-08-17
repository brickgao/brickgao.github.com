---
layout: post
title: Codeforces Round 170 (Div. 2)
disqus: y
tags: [Codeforces, Div2, DFS, 图, 模拟, 计算几何]
---

###A. Circle Line

模拟。

###B. New Problem

暴力判断即可。

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
#define out(v) cerr << #v << ": " << (v) << endl
#define SZ(v) ((int)(v).size())
const int maxint = -1u>>1;
template <class T> bool get_max(T& a, const T &b) {return b > a? a = b, 1: 0;}
template <class T> bool get_min(T& a, const T &b) {return b < a? a = b, 1: 0;}

bool flag;
int n, now, t;
string rec[40], ans;

int main() {
    ans = "a";
    cin >> n;
    for(int i = 0; i < n; i ++) {
        cin >> rec[i];
    }
    while(!flag) {
        flag = true;
        for(int i = 0; i < n; i ++) {
            if(rec[i].find(ans) != string::npos)
                flag = false;
        }
        if(!flag) {
            bool flag2 = true;
            for(int i = 0; i < SZ(ans); i ++) {
                if(ans[i] != 'z') {
                    flag2 = false;
                    break;
                }
            }
            if(flag2) {
                ans += 'a';
                for(int i = 0; i < SZ(ans); i ++) {
                    ans[i] = 'a';
                }
            }
            else {
                if(ans[SZ(ans) - 1] == 'z') {
                    t = 1;
                    now = SZ(ans) - 2;
                    ans[SZ(ans) - 1] = 'a';
                    while(t >= 1 && now >= 0) {
                        t --;
                        if(ans[now] == 'z') {
                            t ++;
                            ans[now] = 'a';
                        }
                        now --;
                    }
                    ans[now + 1] ++;
                }
                else {
                    ans[SZ(ans) - 1] ++;
                }
            }
        }
        else {
            break;
        }
    }
    cout << ans << endl;
    return 0;
}
{% endhighlight %}

###C. Learning Languages

建图，对能互相理解的人建立一条边。

dfs来得出有多少堆互相理解的人，答案是总堆数 - 1。

这里有一个trick，如果所有人都不懂语言，答案就是总人数。

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
#define out(v) cerr << #v << ": " << (v) << endl
#define SZ(v) ((int)(v).size())
const int maxint = -1u>>1;
template <class T> bool get_max(T& a, const T &b) {return b > a? a = b, 1: 0;}
template <class T> bool get_min(T& a, const T &b) {return b < a? a = b, 1: 0;}

int n, m, num, tmp, ans;
bool vis[400];
vector <int> rec[400];
vector <int> p[400];

void dfs(int u) {
    for(int i = 0; i < SZ(p[u]); i ++) {
        int v = p[u][i];
        if(!vis[v]) {
            vis[v] = true;
            dfs(v);
        }
    }
    return;
}

int main() {
    bool flag = true;
    ans = -1;
    memset(vis, false, sizeof(vis));
    for(int i = 0; i < 400; i ++) {
        rec[i].clear();
        p[i].clear();
    }
    memset(rec, 0, sizeof(rec));
    cin >> n >> m;
    for(int i = 0; i < n; i ++) {
        cin >> num;
        if(num > 0) {
            flag = false;
        }
        for(int j = 0; j < num; j ++) {
            cin >> tmp;
            rec[tmp].push_back(i);
        }
    }
    if(flag) {
        cout << n << endl;
        return 0;
    }
    for(int i = 1; i <= m; i ++) {
        for(int j = 0; j < SZ(rec[i]); j ++) {
            for(int k = 0; k < j; k ++) {
                //cout << rec[i][j] << "->" << rec[i][k] << endl;
                p[rec[i][j]].push_back(rec[i][k]);
                p[rec[i][k]].push_back(rec[i][j]);
            }
        }
    }
    for(int i = 0; i < n; i ++) {
        if(!vis[i]) {
            ans ++;
            vis[i] = true;
            dfs(i);
        }
    }
    cout << ans << endl;
    return 0;
}
{% endhighlight %}

###D. Set of Points

看到别人代码才想到的做法...y = x ^ 2这个函数大于0一侧的点能围成凸包，就将m个点分配在函数图像上，剩下的是n - m个点放图像y = - x ^ 2 + k的图像上。

这样只要求三点不共线就行，由(m, m ^ 2)以及(m - 1, (m - 1) ^ 2)两点，得y = x ^ 2最远能达到的点是(1, m ^ 2 - m)，所以k = - m ^ 2 + m。

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
#define out(v) cerr << #v << ": " << (v) << endl
#define SZ(v) ((int)(v).size())
const int maxint = -1u>>1;
template <class T> bool get_max(T& a, const T &b) {return b > a? a = b, 1: 0;}
template <class T> bool get_min(T& a, const T &b) {return b < a? a = b, 1: 0;}

int n, m;

int main() {
    cin >> n >> m;
    if(m == 3 && n > 4) {
        cout << -1 << endl;
        return 0;
    }
    for(int i = 1; i <= m; i ++) {
        cout << i << " " << i * i << endl;
    }
    for(int i = 1; i <= n - m; i ++) {
        cout << i << " " << 0 - m * m + m - i * i << endl;
    }
    return 0;
}
{% endhighlight %}
