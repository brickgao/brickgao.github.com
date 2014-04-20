layout: post
title: Codeforces Round 196 (Div. 2)
date: 2013-08-24 00:42:53
tags: 
- Codeforces
- Div2
- 排序
- 枚举
- 贪心
- 树形DP
- 搜索
categories:
- Algorithm
---

###A. Puzzles

排序搞之。

###B. Routine Problem

枚举一下就好。

###C. Quiz

求可能的最小分数，贪心一下就好了。

<!-- more -->

``` c++
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
#define LL long long
const int maxint = -1u>>1;
const LL mod = 1000000009;
template <class T> bool get_max(T& a, const T &b) {return b > a? a = b, 1: 0;}
template <class T> bool get_min(T& a, const T &b) {return b < a? a = b, 1: 0;}

LL n, m, k;

LL quickpow(LL a, LL b, LL m) {
    LL ans = 1;
    while(b) {
        if(b & 1) {
            ans = (ans * a) % m;
            b --;
        }
        b /= 2;
        a = a * a % m;
    }
    return ans;
}



int main() {
    cin >> n >> m >> k;
    LL res = n - m;
    LL gro = m / (k - 1);
    LL ans = 0;
    if(m % (k - 1) != 0)    ++ gro;
    -- gro;
    if(gro <= res) {
        ans = m;
    }
    else {
        int tmp = m - res * (k - 1);
        //cout << (quickpow(2, 4, mod) - 2) << endl;
        ans = (quickpow(2, tmp / k + 1, mod) - 2 + mod) * (k % mod) % mod;
        //cout << ans % mod << endl;
        ans += res * (k - 1) + tmp % k;
        ans %= mod;
    }
    cout << ans << endl;
    return 0;
}
```

###D. Book of Evil

树形DP, 求一下对每个点距离最远的发生异变的点的距离。

用DP[u][0]来表示u节点在子树中与距离最远的发生异变的点的距离，一边DFS求出。

用DP[u][1]来表示u节点在非子树中与距离最远的发生异变的点的距离，再DFS一遍，DP[u][1] = max(DP[brother1][0] + 1, DP[brother2][0] + 1, ......, DP[father][1]) + 1。

``` c++
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

int n, m, d;
int ans = 0;
vector <int> p[100010];
bool is[100010];
bool vis[100010];
int f[100010][2];

inline int get_max(int cmp1, int cmp2) {
    if(cmp1 > cmp2) return cmp1;
    else            return cmp2;
}

void dfs(int u) {
    vis[u] = true;
    if(is[u])    f[u][0] = 0;
    else        f[u][0] = -1;
    int ret = -1;
    for(int i = 0; i < SZ(p[u]); ++ i) {
        int v = p[u][i];
        if(!vis[v]) {
            dfs(v);
            if(f[v][0] != -1)
                ret = get_max(ret, f[v][0] + 1);
        }
    }
    f[u][0] = get_max(ret, f[u][0]);
}

void dp(int u) {
    vis[u] = true;
    int max1 = -1, max2 = -1;
    int rec1 = -1;
    if(f[u][1] > max1) {
        max1 = f[u][1];
        rec1 = u;
    }
    vector <int> nxt;
    nxt.clear();
    for(int i = 0; i < SZ(p[u]); ++ i) {
        int v = p[u][i];
        if(!vis[v]) {
            nxt.push_back(v);
            if(f[v][0] != -1 && f[v][0] + 1 > max1) {
                max2 = max1;
                max1 = f[v][0] + 1;
                rec1 = v;
                continue;
            }
            if(f[v][0] != -1 && f[v][0] + 1 > max2) {
                max2 = f[v][0] + 1;
            }
        }
    }
    //cout << u << " " << max1 << " " << max2 << " " << rec1 << endl;
    for(int i = 0; i < SZ(nxt); ++ i) {
        int v = nxt[i];
        if(v != rec1) {
            if(max1 != -1)
                f[v][1] = max1 + 1;
            else
                f[v][1] = -1;
        }
        else {
            if(max2 != -1)
                f[v][1] = max2 + 1;
            else
                f[v][1] = -1;
        }
    }
    for(int i = 0; i < SZ(nxt); ++ i)
        dp(nxt[i]);
}

int main() {
    memset(is, false, sizeof(is));
    scanf("%d%d%d", &n, &m, &d);
    for(int i = 0; i <= n; ++ i)    p[i].clear();
    for(int i = 0; i < m; ++ i) {
        int tmp;
        scanf("%d", &tmp);
        is[tmp] = true;
    }
    for(int i = 0; i < n - 1; ++ i) {
        int u, v;
        scanf("%d%d", &u, &v);
        p[u].push_back(v);
        p[v].push_back(u);
    }
    memset(vis, false, sizeof(vis));
    dfs(1);
    if(is[1])   f[1][1] = 0;
    else        f[1][1] = -1;
    memset(vis, false, sizeof(vis));
    dp(1);
    //for(int i = 1; i <= n; ++ i)    cout << f[i][0] << " " << f[i][1] << endl;
    for(int i = 1; i <= n; ++ i) {
        int tmp = get_max(f[i][1], f[i][0]);
        if(tmp != -1 && tmp <= d)   ++ ans;
    }
    printf("%d\n", ans);
    return 0;
}
```

###E. Divisor Tree

暴力搜索，从根节点向上搜索。

``` c++
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
#define LL long long
const int maxint = -1u>>1;
template <class T> bool get_max(T& a, const T &b) {return b > a? a = b, 1: 0;}
template <class T> bool get_min(T& a, const T &b) {return b < a? a = b, 1: 0;}

int n;
int fa[12], ans = maxint;
LL a[12], used[12], cnt[12];

void gao() {
    int ret = 0;
    int t = 0;
    for(int i = 0; i < n; ++ i) {
        if(fa[i] == -1) {
            ret += cnt[i];
            ++ t;
        }
        if(cnt[i] == 1) -- ret;
    }
    ret += n;
    if(t > 1)    ++ ret;
    ans = min(ans, ret);
}

void dfs(int u) {
    if(u >= n) {
        gao();
        return;
    }
    for(int i = u + 1; i < n; ++ i)
        if((a[i] / used[i]) % a[u] == 0) {
            used[i] *= a[u];
            fa[u] = i;
            dfs(u + 1);
            used[i] /= a[u];
        }
    fa[u] = -1;
    dfs(u + 1);
}

int main() {
    scanf("%d", &n);
    for(int i = 0; i < n; ++ i)
        cin >> a[i];
    memset(fa, -1, sizeof(fa));
    sort(a, a + n);
    for(int i = 0; i < n; ++ i) {
        used[i] = 1;
        LL tmp = a[i];
        for(int j = 2; j <= sqrt(a[i]) + 1; ++ j)
            while(tmp % j == 0) {
                tmp /= j;
                ++ cnt[i];
            }
        if(tmp > 1)
            ++ cnt[i];
    }
    dfs(0);
    cout << ans << endl;
    return 0;
}
```
