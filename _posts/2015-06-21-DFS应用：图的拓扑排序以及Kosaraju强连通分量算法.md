---
layout: post
title: DFS应用：图的拓扑排序以及Kosaraju强连通分量算法
category: 科研
comments: true
---

图的DFS算法对于每一个图节点u有2个重要的时间d[u]和f[u],分别代表找到u的时间(其变为灰色节点的时间)以及u完成的时间（其变为黑色节点的时间）.
f[u]越大可能其在最上面（树根处），所以要等其所有子孙都完成其才能完成.
## 1、拓扑排序 ##
对于一个图，找拓扑排序的方法就是通过按照完成时间f[u]的递减顺序即为一个拓扑排序:

```c++
void graph::topology(int start,queue<int> &tpmap)
{
	int Pnum = this->Pnum;
	int visit[1000];
	memset(visit, 0, sizeof(visit));
	stack<int> s;
loop:	s.push(start);
	visit[start] = 1;
	int step;
	int h = 0;
	while (!s.empty())
	{
		int find = 0;
		int p = s.top();
		for (int i = 0; i < Pnum; i++)
		{
			if (!visit[i] && edge[p][i] != INF)
			{
				find = 1;
				s.push(i);
				visit[i] = 1;
				h++;
			}
		}
		if (!find)
		{
			int temp = s.top();
			s.pop();
			tpmap.push(temp);
			h--;
		}
	}
	for (int i = 0; i < Pnum; i++)
		if (!visit[i])
		{
			start = i;
			goto loop;
		}

	for (int i = 0; i < Pnum; i++)//将队列元素颠倒
	{
		s.push(tpmap.front());
		tpmap.pop();
	}
	for (int i = 0; i < Pnum; i++)
	{
		tpmap.push(s.top());
		s.pop();
	}
}

```

## 2、Kosaraju算法 ##
此法用了2此DFS，第一次在原图，第二次在其转置图，此算法主要利用下面定理：
原图的DFS完成时间晚的连通分量，其在转置图中不会有边指向在原图DFS时完成时间早的连通分量.

算法：
	step1：对原图G进行深度优先遍历，记录每个节点的离开时间。
	step2：选择具有最晚离开时间的顶点，对反图GT进行遍历，删除能够遍历到的顶点，这些顶点构成一个强连通分量。
	step3：如果还有顶点没有删除，继续step2，否则算法结束。

```c++
void graph::strongly_connected_conponets(int start)
{
	queue<int> tpmap;
	topology(start, tpmap);//step1
	graph reverse_g;
	for (int i = 0; i < Pnum; i++)//step2、构造图G的转置图
		for (int j = 0; j < Pnum; j++)
		{
			if (edge[i][j])
				reverse_g.edge[j][i] = edge[i][j];
		}
	int visit[100];
	memset(visit, 0, sizeof(visit));
	stack<int> s;
	while (!tpmap.empty())//step3
	{
		
		if (!visit[tpmap.front()])
		{
			s.push(tpmap.front());
			visit[tpmap.front()] = 1;
			tpmap.pop();
		}
		else
		{
			tpmap.pop();
			continue;
		}
		while (!s.empty())//每棵树都完成了即为一个连通分量
		{
			int find = 0;
			int now_point = s.top();
			for (int i = 0; i < Pnum; i++)
			{
				if (reverse_g.edge[now_point][i] != INF && !visit[i])
				{
					find = 1;
					s.push(i);
					visit[i] = 1;
				}
			}
			if (!find)
			{
				cout << now_point;
				s.pop();
			}
		}
		cout << endl;
	}
}
```