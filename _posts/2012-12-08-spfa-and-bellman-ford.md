---
layout: post
title: 图的spfa和Bellman_Ford小结
category: tech
tags: [cut point, bridge, point biconnected, edge biconnected]
---

两个都是最短路算法，相比于Dijkstra ,可以判断负权边，spfa比Bellman_Ford效率高,算是对Bellman_Ford的优化。。。

###Lightoj 1074

判断负环...

####Code

{% highlight cpp %}
//By Brickgao
#include <iostream>
#include <cstdio>
#include <cstring>
#include <cmath>
#include <cstdlib>
#include <algorithm>
#include <vector>
using namespace std;

const int Maxn = 210;
const long long maxint = 1073741824;

typedef struct Edge{
	int u, v;
	long long weight;
}Edge;

Edge edge[Maxn * Maxn * 2];
long long dist[Maxn], busy[Maxn], mark[Maxn], roll[Maxn];

int n, m, q;

int caseno = 1;

void init()
{
	scanf("%d", &n);
	for(int i = 1; i <= n; i ++)
		scanf("%lld", &busy[i]);
	for(int i = 1; i <= n; i ++)
	{
		dist[i] = maxint;
		roll[i] = false;
		mark[i] = false;
	}
	dist[1] = 0;
	scanf("%d", &m);
	for(int i = 0; i < m; i ++)
	{
		scanf("%d%d", &edge[i].u, &edge[i].v);
		edge[i].weight = pow(busy[edge[i].v] - busy[edge[i].u], 3);
		if(edge[i].u == 1)
		{
			dist[edge[i].v] = edge[i].weight;
		}
	}
}

void relax(int u, int v, long long weight)
{
	if(dist[v] > dist[u] + weight)
		dist[v] = dist[u] + weight;
}

bool Bellman_Ford()
{
	for(int i = 0; i < n; i ++)
		for(int j = 0; j < m; j ++)
			relax(edge[j].u, edge[j].v, edge[j].weight);
	bool flag = true;
	for(int i = 0; i < m; i ++)
		if(dist[edge[i].v] > dist[edge[i].u] + edge[i].weight)
		{
			flag = false;
			roll[edge[i].u] = roll[edge[i].v] = true;
		}
	return flag;
}

int dfs(int u)
{
    for(int i = 0; i < m; i ++)
    {
        if(edge[i].u == u && mark[edge[i].v] == false)
        {
            mark[edge[i].v] = true;
            dfs(edge[i].v);
        }
    }
    return 0;
}

int main()
{
	int t, req;
	scanf("%d", &t);
	while(t --)
	{
		init();
		Bellman_Ford();
		dfs(1);
		scanf("%d", &q);
		printf("Case %d:\n", caseno ++);
		while(q --)
		{
			scanf("%d", &req);
			if(dist[req] < 3 || !mark[req] || roll[req])
				printf("?\n");
			else
				printf("%lld\n", dist[req]);
		}
	}
    return 0;
}
{% endhighlight %}

###Lightoj 1063

判断负环上的点以及可以到达负环的点。

{% highlight cpp %}
//By Brickgao
#include <iostream>
#include <cstdio>
#include <cstring>
#include <cmath>
#include <queue>
#include <cstdlib>
#include <algorithm>
#include <vector>
using namespace std;

const int Maxn = 1010;
const int maxint = 1073741824;

typedef struct Edge{
	int to, nxt, dis;
}Edge;

Edge rec[2010];
int head[Maxn], dist[Maxn], cou[Maxn], que[Maxn * Maxn];
bool inque[Maxn], cir[Maxn];
int n, m, cnt;
int caseno = 1;

void add(int u, int v, int d)
{
    rec[cnt].nxt = head[u];
    rec[cnt].to = v;
    rec[cnt].dis = d;
    head[u] = cnt ++;
}

void init()
{
    int u, v, d;
    cnt = 0;
	scanf("%d%d", &n, &m);
	memset(dist, 0, sizeof(dist));
	memset(inque, false, sizeof(inque));
	memset(cir, false, sizeof(cir));
	memset(head, -1, sizeof(head));
	memset(cou, 0, sizeof(cou));
	for(int i = 0; i < m; i ++)
	{
	    scanf("%d%d%d", &v, &u, &d);
	    add(u, v, d);
	}
}

void dfs(int u)
{
    cir[u] = true;
    for(int i = head[u]; i != -1; i = rec[i].nxt)
    {
        if(!cir[rec[i].to])
            dfs(rec[i].to);
    }
}

void spfa()
{
    int front = 0, rear = n;
	for(int i = 0; i < n; i ++)
        que[i] = i;
	while(rear > front)
	{
	    int e = que[front ++];
	    inque[e] = false;
	    cou[e] ++;
        if(cou[e] > n)
        {
            dfs(e);
            continue;
        }
		for(int i = head[e]; i != -1; i = rec[i].nxt)
        {
            int to = rec[i].to;
            if(cir[to]) continue;
            if(dist[to] > dist[e] + rec[i].dis)
            {
                dist[to] = dist[e] + rec[i].dis;
                if(!inque[to])
                {
                    if(cou[to] > (n>>1) && front > 0)
                        que[-- front] = to;
                    else
                        que[rear ++] = to;
                    inque[to] = true;
                }
            }
        }
	}
}

void solve()
{
    spfa();
    int f = 0;
    for(int i = 0; i < n; i ++)
        if(cir[i])
            dfs(i), f = 1;
    printf("Case %d:", caseno ++);
    if(f == 0)
        printf(" impossible");
    else
    {
        for(int i = 0; i < n; i ++)
        if(cir[i])
            printf(" %d", i);
    }
    printf("\n");
}

int main()
{
	int t;
	scanf("%d", &t);
	while(t --)
	{
		init();
		solve();
	}
    return 0;
}
{% endhighlight %}

###Lightoj 1291

变向判断负环，要求增长比率为p，那么我们定义边为I * P - E就行了。。。

{% highlight cpp %}
//By Brickgao
#include <iostream>
#include <cstdio>
#include <cstring>
#include <cmath>
#include <queue>
#include <cstdlib>
#include <algorithm>
#include <vector>
using namespace std;

const int Maxn = 1010;
const int maxint = 1073741824;

typedef struct Edge{
	int to, nxt, dis;
}Edge;

Edge rec[10000];
int head[Maxn], dist[Maxn], cou[Maxn], que[Maxn * Maxn];
bool inque[Maxn], flag;
int n, m, cnt, p;
int caseno = 1;

void add(int u, int v, int d)
{
    rec[cnt].nxt = head[u];
    rec[cnt].to = v;
    rec[cnt].dis = d;
    head[u] = cnt ++;
}

void init()
{
    int u, v, cost, e;
    cnt = 0;
	flag = false;
	scanf("%d%d%d", &n, &m, &p);
	memset(dist, 0, sizeof(dist));
	memset(inque, false, sizeof(inque));
	memset(head, -1, sizeof(head));
	memset(cou, 0, sizeof(cou));
	for(int i = 0; i < m; i ++)
	{
	    scanf("%d%d%d%d", &u, &v, &e, &cost);
	    add(u, v, cost * p - e);
	}
}

void spfa()
{
    int front = 0, rear = n;
	for(int i = 0; i < n; i ++)
        que[i] = i;
	while(rear > front)
	{
	    int e = que[front ++];
	    inque[e] = false;
	    cou[e] ++;
        if(cou[e] > n)
        {
			flag = true;
			break;
        }
		for(int i = head[e]; i != -1; i = rec[i].nxt)
        {
            int to = rec[i].to;
            if(dist[to] > dist[e] + rec[i].dis)
            {
                dist[to] = dist[e] + rec[i].dis;
                if(!inque[to])
                {
                    if(cou[to] > (n>>1) && front > 0)
                        que[-- front] = to;
                    else
                        que[rear ++] = to;
                    inque[to] = true;
                }
            }
        }
	}
}

void solve()
{
    spfa();
	if(flag)
		printf("Case %d: YES\n", caseno ++);
	else
		printf("Case %d: NO\n", caseno ++);
}

int main()
{
	int t;
	scanf("%d", &t);
	while(t --)
	{
		init();
		solve();
	}
    return 0;
}
{% endhighlight %}

