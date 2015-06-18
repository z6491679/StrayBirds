---
layout: post
title: Vertex cover problem（NP）算法是多项式的？
category: 算法
comments: true
---

3.1. Procedure. Given a simple graph G with n vertices and a vertex cover C of G, if C has no removable vertices, output C. Else, for each removable vertex v of C, find the number ρ(C−{v}) of removable vertices of the vertex cover C−{v}. Let vmax denote a removable vertex such that ρ(C−{vmax}) is a maximum and obtain the vertex cover C−{vmax}. Repeat until the vertex cover has no removable vertices.

3.2. Procedure. Given a simple graph G with n vertices and a minimal vertex cover C of G, if there is no vertex v in C such that v has exactly one neighbor w outside C, output C. Else, find a vertex v in C such that v has exactly one neighbor w outside C. Define Cv,w by removing v from C and adding w to C. Perform procedure 3.1 on Cv,w and output the resulting vertex cover.

3.3. Algorithm. Given as input a simple graph G with n vertices labeled 1, 2, …, n, search for a vertex cover of size at most k. At each stage, if the vertex cover obtained has size at most k, then stop.

Part I. For i = 1, 2, ..., n in turn
		1.Initialize the vertex cover Ci = V−{i}.
		2.Perform procedure 3.1 on Ci.
		3.For r = 1, 2, ..., n−k perform procedure 3.2 repeated r times.
		4.The result is a minimal vertex cover Ci.


Part II. For each pair of minimal vertex covers Ci, Cj found in Part I
		1.Initialize the vertex cover Ci, j = Ci∪Cj .
		2.Perform procedure 3.1 on Ci, j.
		3.For r = 1, 2, ..., n−k perform procedure 3.2 repeated r times.
		4.The result is a minimal vertex cover Ci, j.


```
#include <iostream>

#include <vector>

#include<algorithm>

using namespace std;


class graph
{
	vector<int> p_set[101];
	vector<vector<int> > covers;
	vector<int> allcover;
	int Max_V;
	int Vnum;
	int Enum;

public:

	vector<int> res;
	int min;
	void initial();
	bool removable(vector<int> neighbor, vector<int> cover);
	int max_removable(vector<int> cover);
	vector<int> procedure_1(vector<int> cover);
	vector<int> procedure_2(vector<int> cover, int k);
	int cover_size(vector<int> cover);
	bool findvertexcorver();
	bool pairwiseUnions();
};

void graph::initial()
{
	cin >> Vnum >> Enum >> Max_V;
	for (int j = 0; j < Enum; j++)
	{
		int a, b;
		cin >> a >> b;
		p_set[a - 1].push_back(b - 1);
		p_set[b - 1].push_back(a - 1);
	}
	for (int i = 0; i < Vnum; i++)
		allcover.push_back(1);
	min = Vnum + 1;
	res.clear();
}


bool graph::removable(vector<int> neighbor, vector<int> cover)
{
	bool check = true;
	for (int i = 0; i < neighbor.size(); i++)
		if (cover[neighbor[i]] == 0)
		{
			check = false;
			break;
		}
	return check;
}

int graph::max_removable(vector<int> cover)
{
	int r = -1, max = -1;
	for (int i = 0; i < cover.size(); i++)
	{
		if (cover[i] == 1 && removable(p_set[i], cover) == true)
		{
			vector<int> temp_cover = cover;
			temp_cover[i] = 0;
			int sum = 0;
			for (int j = 0; j < temp_cover.size(); j++)
				if (temp_cover[j] == 1 && removable(p_set[j], temp_cover) == true)
					sum++;
			if (sum>max)
			{
				max = sum;
				r = i;
			}
		}
	}
	return r;
}

vector<int> graph::procedure_1(vector<int> cover)
{
	vector<int> temp_cover = cover;
	int r = 0;
	while (r != -1)
	{
		r = max_removable(temp_cover);
		if (r != -1) temp_cover[r] = 0;
	}
	return temp_cover;
}

vector<int> graph::procedure_2(vector<int> cover, int k)
{
	int count = 0;
	vector<int> temp_cover = cover;
	int i = 0;
	for (int i = 0; i < temp_cover.size(); i++)
	{
		if (temp_cover[i] == 1)
		{
			int sum = 0, index;
			for (int j = 0; j<p_set[i].size(); j++)
				if (temp_cover[p_set[i][j]] == 0) { index = j; sum++; }
			if (sum == 1 && cover[p_set[i][index]] == 0)
			{
				temp_cover[p_set[i][index]] = 1;
				temp_cover[i] = 0;
				temp_cover = procedure_1(temp_cover);
				count++;
			}
			if (count>k) 
				break;
		}
	}
	return temp_cover;
}


int graph::cover_size(vector<int> cover)
{
	int count = 0;
	for (int i = 0; i < cover.size(); i++)
		if (cover[i] == 1) count++;
	return count;
}


bool graph::findvertexcorver()
{
	bool found = false;
	int counter = 0;
	int s;
	for (int i = 0; i < allcover.size(); i++)
	{
		if (found) break;
		counter++;
		vector<int> cover = allcover;
		cover[i] = 0;
		cover = procedure_1(cover);
		s = cover_size(cover);
		if (s < min) min = s;
		if (s <= Max_V)
		{
			for (int j = 0; j < cover.size(); j++) if (cover[j] == 1) res.push_back(j + 1);
			covers.push_back(cover);
			found = true;
			break;
		}
		for (int j = 0; j < Vnum - Max_V; j++)
			cover = procedure_2(cover, j);
		s = cover_size(cover);
		if (s < min) min = s;
		for (int j = 0; j < cover.size(); j++) if (cover[j] == 1) res.push_back(j + 1);
		covers.push_back(cover);
		if (s <= Max_V)
		{ 
			found = true; 
			break; 
		}
		res.clear();
	}
	return found;
}


bool graph::pairwiseUnions()
{
	bool found = false;
	int counter = 0;
	int s;
	for (int p = 0; p < covers.size(); p++)
	{
		if (found) break;
		for (int q = p + 1; q < covers.size(); q++)
		{
			if (found) break;
			counter++;
			vector<int> cover = allcover;
			for (int r = 0; r < cover.size(); r++)
				if (covers[p][r] == 0 && covers[q][r] == 0) cover[r] = 0;
			cover = procedure_1(cover);
			s = cover_size(cover);
			if (s<min) min = s;
			if (s <= Max_V)
			{
				for (int j = 0; j < cover.size(); j++) if (cover[j] == 1) res.push_back(j + 1);
				found = true;
				break;
			}
			for (int j = 0; j < Max_V; j++)
				cover = procedure_2(cover, j);
			s = cover_size(cover);
			if (s<min) min = s;
			for (int j = 0; j < cover.size(); j++) if (cover[j] == 1) res.push_back(j + 1);
			if (s <= Max_V){ found = true; break; }
		}
		res.clear();
	}
	return found;
}


int main()
{
	freopen("E:\\study\\算法复杂度\\vetex cover\\in.txt", "rb", stdin);
	int n;
	cin >> n;
	for (int m = 0; m < n; m++)
	{
		graph g;
		g.initial();
		if (g.findvertexcorver())
		{
			cout << g.min << endl;
			for (int i = 0; i < g.res.size(); i++)
				cout << g.res[i] << " ";
			cout << endl;
			continue;
		}
		if (g.pairwiseUnions())
		{
			cout << g.min << endl;
			for (int i = 0; i < g.res.size(); i++)
				cout << g.res[i] << " ";
			cout << endl;
			continue;
		}
		cout << -1 << endl;
	}
	return 0;
}
```
引用

> http://www.dharwadker.org/vertex_cover/

上面给出的是准确的算法，其实不用算那么久，将	`bool findvertexcorver();bool pairwiseUnions();`
中的循环次数从allcover.size()改为10就行，一般前几次就算出来最小值了。

复杂度：
Given a simple graph G with n vertices, the algorithm takes less than n8+2n7+n6+n5+n4+n3+n2 steps to terminate.