---
layout: post
title: Bellman-Ford算法和Dijkstra算法
category: 算法
comments: true
---
Bellman-Ford算法是通过Relax边来实现的，由于最短无负权回路的路径应该最多有V-1条边，所以一共执行V-1次Relax操作即可，而且注意，每次Relax操作都只是基于上一次Relax操作之后的图，和这次Relax中已经Relax了的节点毫无关系（这个是重点！）.
检查负权回路原理：若有负权回路，则经过V-1次relax之后relax还能减小消耗（路径值不会收敛）
```c++
bool graph::Bellman_Ford(int start,int res_d[],int res_pre[])
{
	int d[1000];//记录到源点的距离
	int pre[1000];//记录其前驱节点
	for (int i = 0; i < Pnum; i++)//初始化，距离初源点外都为无穷大，源点为0
	{
		if (i == start)
		{
			d[i] = 0;
			pre[i] = i;
		}
		else
		{
			d[i] = INF;
			pre[i] = INF;
		}
	}
	for (int k = 0; k < Pnum - 1; k++)
	{
		for (int i = 0; i < Pnum; i++)
		{
			for (int j = 0; j < edge_set[i].size(); j++)
			{
				int end = edge_set[i][j].end;
				if (d[end] > d[i] + edge_set[i][j].value)//每次松弛的时候仅在上一次的图上进行
				{
					int a = d[i] + edge_set[i][j].value;
					res_d[end] = d[i] + edge_set[i][j].value;
					res_pre[end] = i;
				}
			}
		}
		for (int i = 0; i < Pnum; i++)
		{
			d[i] = res_d[i];
			pre[i] = res_pre[i];
		}
	}
	//检查是否有源可达的负权回路
	for (int i = 0; i < Pnum; i++)
	{
		for (int j = 0; j < edge_set[i].size(); j++)
		{
			int end = edge_set[i][j].end;
			if (d[end] > d[i] + edge_set[i][j].value)//证明还可以松弛，证明最短路径无法收敛，有负权回路
				return false;
		}
	}
	return true;
}
```

DIjkstra算法：
其仅对与当前的节点相邻的边进行松弛，而每次选松弛后离源点最近的点作为当前节点，并加入到集合中，每一次加入节点u时，此节点u的d[u] = res[u].(反正，假设u是第一个不满足d[u] = res[u]的)

```c++
void graph::dijkstra(int start_p,int res[][2])
{
	int p_num = this->Pnum;
	int p = start_p;
	//一共有n - 1个点的距离 
	int visit[1000];
	memset(visit, 0, sizeof(visit));
	visit[p] = 1;
	for (int i = 0; i < p_num - 1; i++)
	{
		int min = INF;
		int min_p;
		for (int j = 0; j < p_num; j++)
		{
			if (edge[p][j] && !visit[j])//p到j有边且j未被访问 ，判断是否更新点j的路径 
			{
				if (res[j][0] == 0 || res[j][0] > res[p][0] + edge[p][j])
				{
					res[j][0] = res[p][0] + edge[p][j];
					res[j][1] = p;
				}
			}
			//从所有点中找到到start_p最近的点作为下一个起始点 rea[j][0]证明还没能访问 
			if (!visit[j] && res[j][0] < min && res[j][0] != 0)
			{
				min = res[j][0];
				min_p = j;
			}
		}
		p = min_p;
		visit[p] = 1;
	}
}
```


