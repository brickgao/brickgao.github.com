---
layout: post
title: 图的割点、桥与双连通分支小结
disqus: y
tags: [Cut point, Bridge, Point biconnected, Edge biconnected]
---

###基本知识

具体知识点请看byvoid大大的文章(<a href = "http://www.byvoid.com/blog/biconnect/">传送门</a>)。

对于这一部份主要就是以Tarjan算法为核心。

该算法是R.Tarjan发明的。对图深度优先搜索，定义DFS(u)为u在搜索树（以下简称为树）中被遍历到的次序号。定义Low(u)为u或u的子树中能通过非父子边追溯到的最早的节点，即DFS序号最小的节点。根据定义，则有：


{% highlight cpp %}
Low(u)=Min
{
	DFS(u)
	DFS(v) /*(u,v)为后向边(返祖边) 等价于 DFS(v)<DFS(u)且v不为u的父亲节点*/	
	Low(v) /*(u,v)为树枝边(父子边)*/
}
{% endhighlight %}
一个顶点u是割点，当且仅当满足(1)或(2)

(1) u为树根，且u有多于一个子树。

(2) u不为树根，且满足存在(u,v)为树枝边(或称父子边，即u为v在搜索树中的父亲)，使得DFS(u)<=Low(v)。

一条无向边(u,v)是桥，当且仅当(u,v)为树枝边，且满足DFS(u)<Low(v)。

###题目

####Lightoj 1026

求割边

####Lightoj 1063

求割点

####Lightoj 1291

构建边双连通分量

####Lightoj 1300

求图中路程为奇数，走过的边不重复，且能返回起点的路径（类似一笔画的要求）。

具体做法就是求出所有的边双连通分量，求出若此边双连通分量中存在边数为奇数的环，则此双连通分量中的所有点都可一作为起点。

【实在想说一下自己写的tarjan真难看。。。

#####Code

{% highlight cpp linenos %}
//By Brickgao
#include <iostream>
#include <cstdio>
#include <cstring>
#include <cmath>
#include <cstdlib>
#include <algorithm>
#include <vector>
#include <stack>
#define pb push_back
#define cl clear

using namespace std;

typedef struct edge{
    int u, v;
} edge;

typedef struct record{
    vector <int> v;
} record;

int dfn[10010], low[10010], cnt = 0, key;
int n, m, num;
int invis[10010];
record rec[10010];
vector <edge> ans;
bool brige[10010], found;

int dfs(int u, int father)
{
	invis[u] = 1;
	dfn[u] = low[u] = ++ cnt;
	for(int i = 0; i < rec[u].v.size(); i ++)
	{
		int v = rec[u].v[i];
		if(!dfn[v])
		{
			dfs(v, u);
			if(dfn[u] < low[v])
            {
                edge tmp;
                tmp.u = u; tmp.v = v;
                ans.pb(tmp);
            }
			low[u] = min(low[u], low[v]);
		}
		else if(v != father && invis[v])
			low[u] = min(low[u], dfn[v]);
	}
	invis[u] = 0;
	return 0;
}

int dfs_find(int u, int D)
{
    num ++;
    dfn[u] = D;
    invis[u] = 1;
    for(int i = 0; i < rec[u].v.size(); i ++)
    {
        int v = rec[u].v[i];
        if(!invis[v])
            dfs_find(v, D + 1);
        else
            if(invis[v] == 1 && (dfn[u] - dfn[v]) % 2 == 0)
                found = true;
    }
    return 0;
}

int main()
{
    int t, caseno = 1, u, v;
    scanf("%d", &t);
    while(t --)
    {
        key = num = cnt = 0;
        scanf("%d%d", &n, &m);
        ans.cl();
        for(int i = 0; i <= n; i ++)
        {
            dfn[i] = invis[i] = 0;
            rec[i].v.cl();
        }
        for(int i = 0; i < m; i ++)
        {
            scanf("%d%d", &u, &v);
            rec[u].v.pb(v);
            rec[v].v.pb(u);
        }
        for(int i = 0; i < n; i ++)
            if(!dfn[i])
                dfs(i, i);
        for(int i = 0; i < ans.size(); i ++)
        {
            int tmp;
            for(int j = 0; j < rec[ans[i].u].v.size(); j ++)
                if(rec[ans[i].u].v[j] == ans[i].v)
                    tmp = j;
            rec[ans[i].u].v.erase(rec[ans[i].u].v.begin() + tmp);
            for(int j = 0; j < rec[ans[i].v].v.size(); j ++)
                if(rec[ans[i].v].v[j] == ans[i].u)
                    tmp = j;
            rec[ans[i].v].v.erase(rec[ans[i].v].v.begin() + tmp);
        }
        for(int i = 0; i < n; i ++)
            invis[i] = dfn[i] = 0;
        for(int i = 0; i < n; i ++)
        {
            if(invis[i] == 0)
            {
                num = 0;
                found = false;
                dfs_find(i, 1);
                if(found)   key += num;
            }
        }
        printf("Case %d: %d\n", caseno ++, key);
    }
    return 0;
}
{% endhighlight %}

####Lightoj 1308

灾难可以摧毁图中的任意一个点，为了能让图中的蚂蚁能够逃生，蚂蚁会在点上建立逃到地面的通道，求最少建立多少通道以及建立最少通道的方法数。

卡了一阵子的题。

先是要注意这组数据：

{% highlight html %}
Input:
1

5 5
0 1
1 2
2 3
3 4
4 0

Output:
Case 1: 2 10
{% endhighlight %}

就是说如果一个图是点双连通图，至少建立两个通道，因为可能摧毁的是建立通道的点= =。

具体做法是求出所有点双连通分量，然后以割点和点双连通分量为新的点来建立图，所有叶子节点都是需要建立通道的，总方法数就是叶子节点中除了割点的点数乘积。

顺便长一下姿势，是用ull来处理 mod 1 >> 64啊...

#####Code
{% highlight cpp linenos %}
//By Bricgao
#include <iostream>
#include <cstdio>
#include <cstring>
#include <cmath>
#include <cstdlib>
#include <algorithm>
#include <vector>
#include <set>
#include <stack>
#define pb push_back
#define cl clear

using namespace std;

typedef long long LL;
#define MP make_pair
#define pb push_back
typedef pair<int, int> PII;

const int V = 33086;
const int E = 400040;

struct Graph
{
    int head[V], nxt[E], cur[E], ne;
    void init(int V)
    {
        ne = 0, memset(head, -1, sizeof(head));
    }
    void addEdge(int u, int v)
    {
        cur[ne] = v, nxt[ne] = head[u];
        head[u] = ne++;
    }
}g1, g2;
int dfn[V], low[V], sta[V], top, leaf = 0;
unsigned long long ans = 1;
vector<vector<int> > id;
int id_num, cnt[V];

void init(int V)
{
    top = 0, memset(dfn, -1, sizeof(dfn));
    id.clear();
    id.resize(V);
    id_num = -1;
}

void dump(const vector<int> &a)
{
    cnt[++ id_num] = a.size();
    //cout << id_num << ":";
    for(size_t i = 0; i < a.size(); i ++)
    {
        id[a[i]].pb(id_num);
        //cout << a[i] << " ";
    }
    //cout << endl;
}

void dfs(int u, int fa, int d)
{
    sta[top ++] = u;
    dfn[u] = low[u] = d;
    for(int i = g1.head[u]; i != -1; i = g1.nxt[i])
    {
        int v = g1.cur[i];
        if(dfn[v] == -1)
        {
            dfs(v, u, d + 1);
            low[u] = min(low[u], low[v]);
            if(dfn[u] <= low[v])
            {
                vector<int> a;
                a.pb(u);
                for(sta[top] = 0; sta[top] != v; a.push_back(sta[-- top]));
                dump(a);
            }
        }
        else if(v != fa)
            {
                low[u] = min(low[u], dfn[v]);
            }
    }
}

void dfs2(int u, int fa)
{
    int c = 0;
    for(int i = g2.head[u]; i != -1; i = g2.nxt[i])
    {
        int v = g2.cur[i];
        if(v == fa) continue;
        dfs2(v, u);
        c ++;
    }
    if(!c || (fa == -1 && c == 1))
    {
        if(u < id.size())
        {
            ans *= (cnt[u] - 1);
        }
        ++ leaf;
    }
}

int main()
{
    int t, n, m, u, v, caseno = 1;
    scanf("%d", &t);
	while(t --)
	{
    	scanf("%d%d", &n, &m);
		g1.init(n), g2.init(n);
		for(int i = 0; i < m; i ++)
		{
        	scanf("%d%d", &u, &v);
       		g1.addEdge(u, v);
        	g1.addEdge(v, u);
    	}
    	init(n);
    	dfs(0, -1, 0);
    	for(int i = 0; i < n; i ++)
    	{
        	if(id[i].size() < 2)    continue;
        	id_num ++;
        	for(size_t j = 0; j < id[i].size(); j ++)
        	{
            	//cout << id_num << " - " << id[i][j] << endl;
            	g2.addEdge(id_num, id[i][j]);
            	g2.addEdge(id[i][j], id_num);
        	}
    	}
    	leaf = 0;
    	ans = 1;
    	dfs2(0, -1);
    	if(leaf == 1)
    	{
        	leaf ++;
        	ans = (unsigned long long)n * (unsigned long long)(n - 1);
        	ans >>= 1;
   	 	}
    	printf("Case %d: %d %llu\n", caseno ++, leaf, ans);
	}
	return 0;
}
{% endhighlight %}

