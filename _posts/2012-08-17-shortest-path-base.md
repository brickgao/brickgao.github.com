---
layout: post
title: poj-基本最短路问题
category: tech
tags: [Shortest path problem, Graph theory]
---

### poj 1062

中文题，dijkstra基本运用

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

int M, N, lev[110];
int dis[110], map[110][110];
bool vis[110];

void dijstra(int m, int n)
{
    int i, k;
    int min;
    memset(vis, false, sizeof(vis));
    memset(dis, 999999999, sizeof(dis));
    vis[0] = true;
    for(i = 0; i <= N; i++)
        if(lev[i] >= m && lev[i] <= n)
            dis[i] = map[0][i];
    k = 0;
    while(1)
    {
        min = 999999999;
        for(i = 0; i <= N; i++)
            if(lev[i] >= m && lev[i] <= n && !vis[i] && min > dis[i])
            {
                min = dis[i];
                k = i;
            }
        vis[k] = true;
        if(k == 1)
            return;
        for(i = 0; i <= N; i++)
            if(lev[i] >= m && lev[i] <= n && !vis[i] && dis[i] > dis[k] + map[k][i])
            {
                dis[i] = dis[k] + map[k][i];
            }
    }
}

int main()
{
    int i, j, k, t, p;                                                                                                                                          
    memset(map, 127, sizeof(map));
    cin >> M >> N;
    int ans = 999999999;
    for(i = 1; i <= N; i++)
    {
        scanf("%d%d%d", &map[0][i], &lev[i], &k);
        for(j = 1; j <= k; j++)
        {
            scanf("%d%d", &t, &p);
            map[t][i] = p;
        }
    }
    for(i = lev[1] - M; i <= lev[1]; i++)
    {
        dijstra(i, i + M);
        if(ans > dis[1])
            ans = dis[1];
    }
    cout << ans << endl;
    return 0;
}
{% endhighlight %}

### poj 1125

SPFA过的，如果Floyed写的简单一些

{% highlight cpp %}
//By Brickgao
#include <iostream>
#include <cstdio>
#include <cstring>
#include <cmath>
#include <cstdlib>
#include <algorithm>
#include <vector>
#include <queue>

using namespace std;
const int INF=0X1fffffff;
int map[120][120];
int dist[120];
int n;

void spfa(int start)
{
	queue <int> que;
	int visit[110], temp, i;
	while(!que.empty())
		que.pop();
	for(i = 1; i <= n; i++) visit[i] = 0;
	for(i = 1; i <= n; i++)
		dist[i] = INF;
	dist[start] = 0;
	que.push(start);
	visit[start] = 1;
	while(!que.empty())
	{
		temp = que.front(),que.pop();
		visit[temp] = 1;
		for(i = 1; i <= n; i++)
		{
			if(dist[temp] + map[temp][i] < dist[i])
			{
				dist[i] = dist[temp] + map[temp][i];
				if(!visit[i])
				{
					que.push(i);
				}
			}
		}
	}
	return;
}

int main()
{
	int i, j, num, tmp, total, min, u, v;
	bool flag;
	while(~scanf("%d", &n))
	{
		for(i = 1; i <= n; i++)
			for(j = 1; j <= n; j++)
				map[i][j] = INF;
		for(i = 1; i <= n; i++)
			map[i][i] = 0;
		if(n == 0) break;
		for(i = 1; i <= n; i++)
		{
			scanf("%d", &num);
			for(j = 1; j <= num; j++)
			{
				scanf("%d%d", &v, &tmp);
				map[i][v] = tmp;
			}
		}
		min = 99999999;
		u = -1;
		for(i = 1; i <= n; i++)
		{
			flag = false;
			total = 0;
			spfa(i);
			for(j = 1; j <= n; j++)
			{
				if(dist[j] == INF && j != i)
				{
					flag = true;
					break;
				}
				else
				{
					if(dist[j] > total)
						total = dist[j];
				}
			}
			if(!flag)
			{
				if(total < min)
				{
					u = i;
					min = total;
				}
			}
		}
		if(u == -1) printf("disjoint\n");
		else
		{
			printf("%d %d\n", u, min);
		}
	}
    return 0;
}
{% endhighlight %}

### poj 1860

BellmanFord基本运用

{% highlight cpp %}
//By Brickgao
#include <iostream>
#include <cstdio>
#include <cstring>
#include <cmath>
#include <cstdlib>
#include <algorithm>
#include <vector>
const int maxnum = 500;
using namespace std;

typedef struct Edge{
	int u, v;
	double weight, cost;
}Edge;

int n, m, s;
double v;

Edge edge[maxnum];
double dist[maxnum];

bool Bellman_Ford()
{
	int i, j;
	bool flag;
	for(i = 1; i <= n; i++)
		dist[i] = -1<<30;
	dist[s] = v;
	for(i = 1; i <= n; i++)
	{
	    flag = true;
		for(j = 1; j <= 2 * m; j++)
		{
		    if(dist[edge[j].u] > 0 && dist[edge[j].v] < (dist[edge[j].u] - edge[j].cost) * edge[j].weight)
		    {
                flag = false;
                dist[edge[j].v] = (dist[edge[j].u] - edge[j].cost) * edge[j].weight;
		    }
		}
		if(flag)   return false;
	}
    return true;
}

int main()
{
	int A, B, i;
	while(~scanf("%d%d%d%lf", &n, &m, &s, &v))
	{
		for(i = 1; i <= m; i++)
		{
		    scanf("%d%d", &A, &B);
			edge[i].u = A; edge[i].v = B;
			edge[i + m].u = B; edge[i + m].v = A;
			scanf("%lf%lf%lf%lf", &edge[i].weight, &edge[i].cost, &edge[i + m].weight, &edge[i + m].cost);
		}
		if( Bellman_Ford() ) printf("YES\n");
		else
            printf("NO\n");
    }
    return 0;
}
{% endhighlight %}

### poj 2240

BellmanFord基本运用，注意数据中存在自己成环的情况。

{% highlight cpp %}
//By Brickgao
#include <iostream>
#include <cstdio>
#include <cstring>
#include <cmath>
#include <cstdlib>
#include <algorithm>
#include <vector>
#include <string>
#include <queue>
using namespace std;

typedef struct Edge{
	int u, v;
	double weight;
}Edge;

Edge edge[10000];
const double eps = 1e-8;
string name[40];
double dist[40];
int n, num;
bool flag;

void relax(int u, int v, double weight)
{
	if(dist[v] < dist[u] * weight)
		dist[v] = dist[u] * weight;
}

bool Bellman_Ford(int source)
{
	for(int i = 1; i <= n; i++)
		dist[i] = 0.0;
    dist[source] = 1.0;
	for(int i = 1; i <= n - 1; i++)
		for(int j = 1; j <= num; j++)
			relax(edge[j].u, edge[j].v, edge[j].weight);
	flag = false;
	for(int i = 1; i <= num; i++)
		if(dist[edge[i]. v] < dist[edge[i]. u] * edge[i]. weight)
		{
			flag = true;
			break;
		}
	return flag;
}

int main()
{
	string stmp1, stmp2;
	int tmp1, tmp2, i, j, cas, t;
	double tmp;
	cas = 0;
	while(~scanf("%d", &n))
	{
		num = 0;
		flag = false;
		cas ++;
		if(n == 0) break;
		for(i = 1; i <= n; i++)
			cin >> name[i];
		scanf("%d", &t);
		for(i = 1; i <= t; i++)
		{
			num ++;
			cin >> stmp1;
			scanf("%lf", &tmp);
			cin >> stmp2;
			for(j = 1; j <= n; j++)
			{
				if(name[j] == stmp1) tmp1 = j;
				if(name[j] == stmp2) tmp2 = j;
				if(stmp1 == stmp2 && (tmp - eps) > 1.0) flag = true;
			}
		    edge[num].u = tmp1;
			edge[num].v = tmp2;
			edge[num].weight = tmp;
		}
		for(i = 1; i <= n; i++)
		{
			if(flag == true) break;
			Bellman_Ford(i);
			//for(i = 1; i <= n; i++)
				//printf("%lf ", dist[i]);
		    //printf("\n");
		}
		printf("Case %d: ", cas);
		if(flag) printf("Yes\n");
		else printf("No\n");
	}
    return 0;
}
{% endhighlight %}

### poj 2253

Floyed基本应用，在poj上注意输出格式

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

typedef struct point{
   double x, y;
}point;

point a[210];
double map[210][210];

int main()
{
	int n, i, j, k, num = 0;
	while(cin >> n)
	{
		if(n == 0) break;
		num++;
		for(i = 1; i <= n; i++)
		{
			scanf("%lf%lf", &a[i].x, &a[i].y);
		}
		for(i = 1; i <= n - 1; i++)
			for(j = i + 1; j <= n; j++)
			{
				double x = a[i].x - a[j].x;
				double y = a[i].y - a[j].y;
				map[i][j] = map[j][i] = sqrt(x * x + y * y);
			}

		for(k = 1; k <= n; k++)
			for(i = 1; i <= n; i++)
				for(j = 1; j <= n; j++)
				    if(i != j && i != k && j != k && map[i][k] < map[i][j] && map[k][j] < map[i][j])
					{	
						if(map[i][k] > map[k][j])
						     map[i][j] = map[j][i] = map[i][k];
						else
							 map[i][j] = map[j][i] = map[k][j];
					}
		printf("Scenario #%d\n",num);
		printf("Frog Distance = %.3f\n\n", map[1][2]);
	}
    return 0;
}
{% endhighlight %}

### poj 3259

BellmanFord基本应用

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
const int maxnum = 7000;

typedef struct Edge{
	int u, v;
	long long weight;
}Edge;
int n, m, w, f;
Edge edge[maxnum];
long long dist[maxnum];

bool Bellman_Ford()
{
	bool flag = false;
	int i, j;
	for(i = 1; i <= n; i++)
		dist[i] = 1e13;
	dist[1] = 0;
	for(i = 1; i <= n - 1; i++)
		for(j = 1; j <= 2 * m + w; j++)
		{
			if(dist[edge[j].v] >(dist[edge[j].u] + edge[j].weight))
				dist[edge[j].v] = dist[edge[j].u] + edge[j].weight;
		}
	for(j = 1; j <= 2 * m + w; j++)
	{
		if(dist[edge[j].v] >(dist[edge[j].u] + edge[j].weight))
		{
			flag = true;
			break;
		}	
	}
	return flag;
}


int main()
{
	int i, k;
	long long tmp;
	while(~scanf("%d", &f))
	{
		for(k = 1; k <= f; k++)
		{
			scanf("%d%d%d", &n, &m, &w);
			for(i = 1; i <= m; i++)
			{
				scanf("%d%d%lld", &edge[i].u, &edge[i].v, &edge[i].weight);
				edge[i + m].u = edge[i].v;
				edge[i + m].v = edge[i].u;
				edge[i + m].weight = edge[i].weight;
			}
			for(i = 1; i <= w; i++)
			{
				scanf("%d%d%lld", &edge[i + 2 * m].u, &edge[i + 2 * m].v, &tmp);
				edge[i + 2 * m].weight = 0 - tmp;
			}
			if(Bellman_Ford()) printf("YES\n");
			else
				{
					if(dist[1] < 0) printf("YES\n");
					else printf("NO\n");
				}
		}
	}
    return 0;
}
{% endhighlight %}

