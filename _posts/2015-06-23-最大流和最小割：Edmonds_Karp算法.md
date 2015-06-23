---
layout: post
title: 最大流和最小割：Edmonds_Karp算法
category: 科研
comments: true
---

最大流：从s->t可以通过的最大流量
最小割:将图用一条直线切断，并将s和t分到不同的2边，此为一个割，割会经过图中的边，这些边的容量（只算从s->t方向的）总和为c，当c最小时，此割即为最小割.可以证明，最大流=最小割的容量！（这就是Ford_Fulkerson定理）.

Ford_Fulkerson方法
----------------
此为求最大流的基本方法，通过在残留网络中找增广路径不断迭代来求解最大流，在寻找增广路径的时候会有不同的方式，当选择BFS方式的时候则即为Edmonds_Karp算法。
![](http://img.blog.csdn.net/20150623194904366)
										左边为每次的残留网络，右边为current-graph



**Edmonds_Karp算法**
通过BFS的方式来进行增广路径的查找，即每次找步数最少的路径进行残留网络的修改，此方法的复杂度为O(VE^2).

```c++
int graph::Edmonds_Karp(int start, int end, int remain_g[][1000], int current_g[][1000])//按BFS找增广路径(即每次找边数最短的)实现Ford_Fulkerson方法
{
	for (int i = 0; i < Pnum; i++)
		for (int j = 0; j < Pnum; j++)
		{
			if (edge[i][j] == INF)
				remain_g[i][j] = 0;
			else
				remain_g[i][j] = edge[i][j];
		}
	int max_flow = 0;
	while (1)
	{
		int visit[1000];
		int pre[1000];//记录节点的前驱，好找路径,其实这里的visit可以不要,就用pre
		memset(visit, 0, sizeof(visit));
		memset(pre, 0, sizeof(pre));
		//bfs在残留网络中找增广路径
		queue<int> q;
		q.push(start);
		while (!q.empty())
		{
			int temp_point = q.front();
			q.pop();
			if (temp_point == end)
				break;
			for (int i = 0; i < Pnum; i++)
			{
				if (remain_g[temp_point][i] > 0 && !visit[i])
				{
					q.push(i);
					pre[i] = temp_point;
					visit[i] = 1;
				}
			}
		}
		//更新残留网络
		if (pre[end] == 0)//end的前驱没有更新，证明没有增广路径了
			break;
		int min = INF;
		for (int temp_point = end; temp_point != start; temp_point = pre[temp_point])
			min = zzMin(remain_g[pre[temp_point]][temp_point], min);
		for (int temp_point = end; temp_point != start; temp_point = pre[temp_point])
		{
			remain_g[pre[temp_point]][temp_point] -= min;
			remain_g[temp_point][pre[temp_point]] = min;//此为负向边
		}
		max_flow += min;
	}
	return max_flow;
}
```


在找到最大流后如何寻找最小割：
在求得最大流后得到残留网络，从s开始进行bfs，并将所有经过的点标记，然后在残留网络中找到连接标记的点和非标记的点的边，经过这些边的直线即为一个最小割.(这些边事实上就已经是满流量了,即流量=容量).

```c++
//remain_g为找到最大流后得到的残留网络
bool graph::Find_Min_Cut(int start,int remain_g[][1000], vector<int> &s_set, vector<int> &t_set)
{
	int visit[1000];
	memset(visit, 0, sizeof(visit));
	queue<int> s;
	s.push(start);
	while (!s.empty())
	{
		int temp_point = s.front();
		s.pop();
		for (int i = 0; i < Pnum; i++)
			if (remain_g[temp_point][i] > 0 && !visit[i])
			{
				s.push(i);
				visit[i] = 1;
				s_set.push_back(i);
			}
	}
	for (int i = 0; i < Pnum; i++)
		if (!visit[i])
			t_set.push_back(i);
	return true;
}
```

此方法为Graph cut的基础，非常重要！