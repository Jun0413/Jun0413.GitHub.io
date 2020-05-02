---
title: "Tarjan算法"
layout: post
date: 2020-05-01 21:19
tag:
- algorithm
- graph
headerImage: false
category: blog
author: junhao
description: N.A.
---

## 介绍

第一次接触Tarjan算法的思想是通过这道Leetcode题：[1192. Critical Connections in a Network
](https://leetcode.com/problems/critical-connections-in-a-network)，问的是如何在一个无向图中找到所有的critical connection，a.k.a. 桥。凭借Tarjan算法的思维框架，通过一次DFS就可以求解，时间复杂度O(V+E)。在展开之前，需要先了解这个算法引入的`id`和`lowlink`的概念。`id`很好理解，就是跑一遍DFS给每个点的先后顺序。而一个点的**lowlink value**指的是这个点可以到达的最小`id`。有了这两个概念，对于某条边`(u, v)`，如果满足了`u.id < v.lowlink`，那么这条边就是我们要找的桥之一。

## 应用1：找桥

下面给出LC1192的C++代码。

{% highlight C++ %}
class Solution {
public:
    vector<vector<int>> criticalConnections(int n, vector<vector<int>>& connections) {
        // build the graph
        vector<vector<int>> graph(n, vector<int>());
        for (auto& e : connections) {
            graph[e[0]].push_back(e[1]);
            graph[e[1]].push_back(e[0]);
        }
        // tarjan algorithm
        vector<int> id(n, 0);
        vector<int> ll(n, 0);
        vector<vector<int>> res;
        vector<bool> visited(n, false);
        dfs(0, graph, 0, id, ll, 0, visited, res);
        return res;
    }
    
    void dfs(int u,
             vector<vector<int>>& graph,
             int prev,
             vector<int>& id,
             vector<int>& ll,
             int ts,
             vector<bool>& visited,
             vector<vector<int>>& res
            ) {
        visited[u] = true;
        id[u] = ll[u] = ts + 1;
        for (int v : graph[u]) {
            // #1 (below line)
            if (v == prev) { continue; }
            if (!visited[v]) {
                dfs(v, graph, u, id, ll, ts + 1, visited, res);
                ll[u] = min(ll[u], ll[v]); // #1.5
                if (id[u] < ll[v]) {
                    res.emplace_back(initializer_list<int>({u, v}));
                }
            } else {
                // #2 (below line)
                ll[u] = min(ll[u], id[v]);                            
            }
        }
    }
};
{% endhighlight %}

*注意点*

#1：如果DFS的下个点`v`是父节点`prev`的话，我们应该跳过这种情况，否则`v`的`lowlink value`有可能和`prev`的`id`一样，因此不满足`prev.id < v.lowlink`，尽管`(prev, v)`可能是critical connection。可以参考下图的例子。

<div style="display: flex; flex-direction: column; justify-content: center; align-items: center">
    <img class="image" src="{{ site.url }}/assets/images/tarjan_1.png" alt="tarjan_1">
    <figcaption class="caption">(prev, v)是critical connection，但prev.id >= prev.lowlink == v.lowlink</figcaption>
</div>

#2：对于#1的情况，我们可能会觉得，`prev`应该是已经访问过的节点，那么访问过的节点都跳过不就行了吗？但在tarjan算法中，对于访问过的非父节点，我们还需要用这个节点的`id`来更新当前的`lowlink value`。如果感觉难以理解，请这么想象：在一条首尾相连的链上进行DFS到尾节点的时候，因为下个点——首节点已经被访问过了，而首节点的`id`最小，这个`id`就应该是末尾节点的`lowlink value`，这也是对于已经访问过的非父节点我们所进行的操作。有点类似于`lowlink values`的base case，而#1.5就是在unwind DFS stack的时候一层一层往之前传递并更新`lowlink value`。

## 应用2：找割点

相应的，我们可以把tarjan的思想用于找割点(Critical Node)的问题上。Leetcode Discussion上有这样的[题目](https://leetcode.com/discuss/interview-question/436073/)。一个Critical Node `u`，需要满足：

1. Critical Connection (u, v)  ==> `u.id < v.lowlink`
2. Entry of Cycle              ==> `u.id = next.lowlink`
3. If u is a starter node, it connects to more than 1 connected component.

要注意的是上面第三点中的starter node指的是启动一次DFS的starter node。容易想到的是对于非starter node的Critical Connection `(u, v)`，`u`必然是一个割点。除此之外，还有第二种情况。比如下图就是这样一个例子。这种情况出现于环的入口，且该入口除了连接这个环，还连接了不和这个环连通的点。为此，对于一条从`u`出来的在环上的边`(u, v)`，需要满足`u.id = next.lowlink`。最后，对于前两种情况都不考虑的starter node，容易想到如果是割点，必定是连接两个及以上的连通部分。

<div style="display: flex; flex-direction: column; justify-content: center; align-items: center">
    <img class="image" src="{{ site.url }}/assets/images/tarjan_2.png" alt="tarjan_2">
    <figcaption class="caption">不存在Critical Connection却存在Critical Node</figcaption>
</div>

下面给出C++代码。

{% highlight C++ %}
void dfs(vector<vector<int>>& g,
         int start_node,
         int u,
         int prev,
         vector<int>& id,
         vector<int>& ll,
         int& ts,
         vector<bool>& visited,
         vector<bool>& is_critical,
         int& out_edges) {
    
    if (prev == start_node) { ++out_edges; }
    visited[u] = true;
    id[u] = ll[u] = ++ts;
    for (int v : g[u]) {
        if (!visited[v]) {
            dfs(g, start_node, v, u, id, ll, ts, visited, is_critical, out_edges);
            ll[u] = min(ll[u], ll[v]);
            if (id[u] <= ll[v]) { is_critical[u] = true; }
        } else {
            ll[u] = min(ll[u], id[v]);
        }
    }
}


vector<int> findCriticalNodes(vector<vector<int>>& g, int n) {
    vector<int> id(n, 0);
    vector<int> ll(n, n + 1);
    int out_edges, ts = 0;
    vector<bool> is_critical(n, false);
    vector<bool> visited(n, false);
    for (int i = 0; i < n; ++i) {
        if (!visited[i]) {
            out_edges = 0;
            dfs(g, i, i, -1, id, ll, ts, visited, is_critical, out_edges);
            is_critical[i] = (out_edges > 1);
        }
    }
    
    vector<int> res;
    for (int i = 0; i < n; ++i) {
        if (is_critical[i]) { res.push_back(i); }
    }
    return res;
}
{% endhighlight %}

## 应用3：找强连接部件

以上两个应用都是对于无向图的，而强连接(SCC)是种有向图的性质，也是Tarjan算法真正解决的问题。首先意识到，在DFS遍历中，SCC会形成subtree，所以找SCC其实就是在找这些head of subtree，因此一遍DFS是有可能的。同样地，我们在DFS中维护`id`和`lowlink values`。如果一个节点`u`满足`u.id = u.lowlink`，那么这个点就是一个head of subtree，因此我们可以在遍历的时候维护一个栈，用来保存这个subtree的元素。此外，为了更深刻的认识，归纳了这4种边：`tree edge`, `back edge`, `forward edge`, `cross edge`，如下图所示。

<div style="display: flex; flex-direction: column; justify-content: center; align-items: center">
    <img class="image" src="{{ site.url }}/assets/images/tarjan_3.jpg" alt="tarjan_3">
    <figcaption class="caption">Taken from GeeksforGeeks</figcaption>
</div>

对于cross edge，即从当前SCC指向已经访问过的SCC的边，应当忽略。以下是C++代码。

{% highlight C++ %}

void dfs(int u,
         vector<vector<int>>& g,
         vector<int>& id,
         vector<int>& ll,
         vector<bool>& on_stk,
         stack<int>& stk,
         vector<vector<int>>& res,
         int ts
        ) {
    id[u] = ll[u] = ++ts;
    stk.push(u);
    on_stk[u] = true;
    for (int v : g[u]) {
        if (id[v] == 0) { // not visited
            dfs(v, g, id, ll, on_stk, stk, res);
            ll[u] = min(ll[u], ll[v]);
        } else if (on_stk[v]) { // a back edge, not a cross edge
            ll[u] = min(ll[u], id[v]);
        }
    }

    // check if u is a head
    if (low[u] == id[u]) {
        vector<int> scc;
        while (stk.top() != u) {
            int t = stk.top();
            stk.pop();
            on_stk[t] = false;
            scc.push_back(t);
        }
        on_stk[stk.top()] = false;
        scc.push_back(stk.top());
        stk.pop();
    }
}

vector<vector<int>> findSCC(vector<vector<int>>& g, int n) {
    vector<int> id(n, 0);
    vector<int> ll(n, n + 1);
    vector<bool> on_stk(n, false);
    stack<int> stk;
    vector<vector<int>> res; 
    for (int i = 0; i < n; ++i) {
        dfs(i, g, id, ll, on_stk, stk, res, 0);
    }
}

{% endhighlight %}

## 参考以及更多

- [Youtube](https://www.youtube.com/watch?v=aZXi1unBdJA)
- [GeeksforGeeks](https://www.geeksforgeeks.org/tarjan-algorithm-find-strongly-connected-components/)
- [HackerEarth](https://www.hackerearth.com/practice/algorithms/graphs/articulation-points-and-bridges/tutorial/)
