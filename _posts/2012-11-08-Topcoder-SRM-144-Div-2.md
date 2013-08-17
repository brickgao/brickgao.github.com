---
layout: post
title: Topcoder SRM 144 (Div. 2)
disqus: y
tags: [Topcoder, Div2]
---

哦呵呵呵呵，这两天开始翻topcoder做。

###200pt Time

给出从一天开始过了多少秒，求现在时间。

{% highlight cpp linenos %}
#include <cstdio>
#include <iostream>
#include <cstring>
#include <algorithm>
#include <cmath>
#include <vector>
using namespace std ;
#define For(i , n) for(int i = 0 ; i < (n) ; ++i)
#define SZ(x)  (int)((x).size())
typedef long long lint ;
const int maxint = -1u>>2 ;
const double eps = 1e-6 ;
 

class Time
{
public:
string whatTime(int seconds)
{
	string ans;
	char tmp[20];
	int h, m, s;
	h = seconds / 3600;
	seconds %= 3600;
	m = seconds /60;
	seconds %= 60;
	s = seconds;
	sprintf(tmp, "%d", h);
	for(int i = 0; i < strlen(tmp); i ++)
		ans += tmp[i];
	ans += ':';
	sprintf(tmp, "%d", m);
	for(int i = 0; i < strlen(tmp); i ++)
		ans += tmp[i];
	ans += ':';
	sprintf(tmp, "%d", s);
	for(int i = 0; i < strlen(tmp); i ++)
		ans += tmp[i];
	return string(ans) ;
}
};
{% endhighlight %}

###550pt BinaryCode

给一种二进制串加密模式，串头部等于串头部数字加后一个数字，串尾部数字等于尾部数字加上前一个数字，其他位置上的数字的前一个数字加上后一个数字。

然后假设第一位是1或者0求是否能解密出串，解密不出输出“NONE”。

{% highlight cpp linenos %}
#include <cstdio>
#include <iostream>
#include <cstring>
#include <algorithm>
#include <cmath>
#include <vector>
using namespace std ;
#define For(i , n) for(int i = 0 ; i < (n) ; ++i)
#define SZ(x)  (int)((x).size())
typedef long long lint ;
const int maxint = -1u>>2 ;
const double eps = 1e-6 ;
 

class BinaryCode
{
public:
vector <string> decode(string message)
{
	vector <string> ans;
	bool flag;
	string st;
	int tmp;
	if(message.size() != 1)
	{
		flag = true;
		st.clear();
		st += '0';
		for(int i = 0; i < message.size() - 1; i ++)
		{
			if(i != 0)
				tmp = message[i] - st[i] - (st[i - 1] - '0');
			else
				tmp = message[i] - st[i];
			if(tmp != 0 && tmp != 1)	flag = false;
			st += '0' + tmp;
		}
		if(message[message.size() - 1] == st[st.size() - 1] + st[st.size() -2] - '0' && flag)
			ans.push_back(st);
		else
			ans.push_back("NONE");
		st.clear();
		flag = true;
		st += '1';
		for(int i = 0; i < message.size() - 1; i ++)
		{
			if(i != 0)
				tmp = message[i] - st[i] - (st[i - 1] - '0');
			else
				tmp = message[i] - st[i];
			if(tmp != 0 && tmp != 1) flag = false;
			st += '0' + tmp;
		}
		if(message[message.size() - 1] == st[st.size() - 1] + st[st.size() -2] - '0' && flag)
			ans.push_back(st);
		else
			ans.push_back("NONE");
	}
	else
	{
		if(message[0] == '0')
			ans.push_back("0");
		else
			ans.push_back("NONE");
		if(message[0] == '1')
			ans.push_back("1");
		else
			ans.push_back("NONE");

	}
	return vector <string>(ans) ;
}
};
{% endhighlight %}

###1100pt PowerOutage

给一个图，求从root遍历所有点的费用。

树形DP...第一个树形dp【dp还没搞清楚 = =。

全部路径的费用 * 2减去最大的从某一点到root的距离。

{% highlight cpp linenos %}
#include <cstdio>
#include <iostream>
#include <cstring>
#include <algorithm>
#include <cmath>
#include <vector>
#include <stack>
using namespace std ;
#define For(i , n) for(int i = 0 ; i < (n) ; ++i)
#define SZ(x)  (int)((x).size())
typedef long long lint ;
const int maxint = -1u>>2 ;
const double eps = 1e-6 ;

class PowerOutage
{
public:
int n;
int map[100][100];

int dfs(int pre, int cur)
{
	int ans = 0;
	for(int i = 0; i <= n; i ++)
		if(map[cur][i] != -1 && i != pre)
		{
			int tmp = dfs(cur, i);
			if(tmp > ans) ans = tmp;
		}
	if(cur != 0) ans += map[pre][cur];
	return ans;
}

int estimateTimeOut(vector <int> fromJunction, vector <int> toJunction, vector <int> ductLength)
{
	int ans = 0;
	n = 0;
	memset(map, -1, sizeof(map));
	for(int i = 0; i < fromJunction.size(); i ++)
	{
		map[fromJunction[i]][toJunction[i]] = map[toJunction[i]][fromJunction[i]] = ductLength[i];
		ans += ductLength[i];
		if(n < max(fromJunction[i], toJunction[i]))
				n = max(fromJunction[i], toJunction[i]);
	}
	ans *= 2;
	ans -= dfs(-1, 0);
	return (int)ans;
}
};
{% endhighlight %}
