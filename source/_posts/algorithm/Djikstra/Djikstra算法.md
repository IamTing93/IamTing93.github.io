---
title: Djikstra算法
date: 2018-04-10 16:09:47
tags: algorithm
---

本文将简单介绍求带权的单源最短路径长度的Djikstra算法

<!-- more -->
<b>问题<b>:假设正在进行一场从广州到北京的旅行，途中会经过很多城市，每两个城市之间所花的时间长短不一。要怎样选择路线才能使得所花的时间最短？

最简单的想法就是每次都从中转城市里挑一个所花时间最短的城市A，去到城市A后就更新一下到其他城市所需要的时间表，即从广州出发，通过城市A到其他城市(如城市B)的时间t是否比从广州直接到城市B的时间u要短，是的话就用时间t更新从广州直接到城市B所花的时间信息，然后从时间表中再挑一个所花时间最短的且未曾到访的城市X，重复以上操作，直到遍历完所有的城市。当遍历完后，就可以很容易知道从广州到北京的所花的最短时间了。

下面对上述的过程进行更抽象的描述。

假设现在有如下的图，我们需要计算从0号点到5号点的最短距离。

![map](/img/algorithm/Djikstra/map.jpg "map")

假设有一个表dist，记录着从出发点(0号点)到其他点的最短距离。还有一个表visited，记录着已经访问过的点。

1. 对表dist和表visited进行初始化。
    与0号点直接邻接的点，它们之间距离就是直接距离；而不相邻的点，它们之间的距离是无限远。所以此时表dis和表visited的状态如下。

    ![step1_表dist](/img/algorithm/Djikstra/step1_dist.jpg "step1_表dist")

    ![step1_表visited](/img/algorithm/Djikstra/step1_visited.jpg "step1_表visited")

2. 从表dist中可以知道，从0号点出发，到2号点的距离是最短的，所以选择2号点作为下一个拓展的点，标记为visited。此时需要更新表dist的信息。更新的原则是`若dist(a->b) + dist(b->c) < dist(a->c)，则选择dist(a->b) + dist(b->c)作为从a到c的距离`，也就是说经过中转点距离比直接去的距离还要少。遍历那些还没有visited的点后，得到的表dist和表visted如下。

    ![step2_表dist](/img/algorithm/Djikstra/step2_dist.jpg "step2_表dist")

    ![step2_表visited](/img/algorithm/Djikstra/step2_visited.jpg "step2_表visited")

	譬如，原本0->3的距离是无穷远，但从0->2->3的距离为6，所以更新0->3的距离为6。

3. 重复上述的操作，直到所有的点都遍历完。这里给出每一步操作后表dist和表visited的状态。

    第三步，拓展1号点。

    ![step3_表dist](/img/algorithm/Djikstra/step3_dist.jpg "step3_表dist")

    ![step3_表visited](/img/algorithm/Djikstra/step3_visited.jpg "step3_表visited")

	第四步，拓展3号点。

    ![step4_表dist](/img/algorithm/Djikstra/step4_dist.jpg "step4_表dist")

    ![step4_表visited](/img/algorithm/Djikstra/step4_visited.jpg "step4_表visited")

	第五步，拓展4号点。

    ![step5_表dist](/img/algorithm/Djikstra/step5_dist.jpg "step5_表dist")

    ![step5_表visited](/img/algorithm/Djikstra/step5_visited.jpg "step5_表visited")

	第六步，拓展5号点。

    ![step6_表dist](/img/algorithm/Djikstra/step6_dist.jpg "step2_表dist")

    ![step6_表visited](/img/algorithm/Djikstra/step6_visited.jpg "step6_表visited")

从上面的描述中可以看出，Djikstra算法是采取了贪心策略：每次都选距离最短的。
当所有距离都相等的时候，Djikstra算法退化为广度优先搜索(BFS)。

下面给出代码

Djikstra.hpp
```cpp
#pragma once
#ifndef TING_DIJKSTRA_CPP
#define TING_DIJKSTRA_CPP

#define MAX 0x2ffffff
#define MAX_SIZE 20

class Djikstra {

public:
    Djikstra();
    virtual ~Djikstra();

    int run(void** map, const int& size, int* out, const int& from, const int& to);

private:
    void init(const int& size);
    void initDistance(void** map, const int& size);
    int selectNearestNode();
    bool isVisited(const int& node);
    void setVisited(const int& node);
    void updateDistance(const int & chooseNode, void** map);

    int* mDistance;
    bool* mVisited;
    int mFrom;
    int mTo;
    int mSize;
};

#endif // !TING_DIJKSTRA_CPP
```

Djikstra.cpp
```cpp
#include "Dijkstra.hpp"

Djikstra::Djikstra() {}

Djikstra::~Djikstra() {
    if (mDistance) {
        delete mDistance;
        mDistance = 0;
    }

    if (mVisited) {
        delete mVisited;
        mVisited = 0;
    }
}

/**
 * @param map   the data matrix
 * @param size  size of matrix
 * @param out   output
 * @param from  the begining node
 * @param to    the ending node
 * @return true if successful
 */

int Djikstra::run(void** map, const int& size, int* out, const int& from, const int& to) {
    if (size > MAX_SIZE) {
        return 0;
    }

    mFrom = from;
    mTo = to;
    mSize = size;

    init(size);
    initDistance(map, size);
    setVisited(mFrom);

    for (register int i = 1; i < mSize; i++) {
        int node = selectNearestNode();
        setVisited(node);
        updateDistance(node, map);
    }

    *out = mDistance[to];

    return 1;
}

/**
 * initialize the data member
 * param size   size of matrix
 */
void Djikstra::init(const int& size) {
    if (mDistance) {
        delete mDistance;
        mDistance = 0;
    }

    if (mVisited) {
        delete mVisited;
        mVisited = 0;
    }

    mDistance = new int[size];
    mVisited = new bool[size];

    for (register int i = 0; i < size; i++) {
        mDistance[i] = MAX;
        mVisited[i] = false;
    }
}

/**
 * initialize the mDistance data member
 * @param map   the data matrix
 * @param size  size of matrix
 */
void Djikstra::initDistance(void** map, const int & size) {
    int* _map = (int*)(map + mFrom);
    for (register int i = 0; i < mSize; i++) {
        if (mFrom == i) {
            mDistance[i] = 0;
            continue;
        }

        if (_map[i]) {
            mDistance[i] = _map[i];
        }
    }
}

/**
* select the closest and un-visited node to the beginning one
*/
int Djikstra::selectNearestNode() {
    int min = MAX;
    int node = -1;
    //choose the nearest node which has been not visited
    for (register int i = 0; i < mSize; i++) {
        if (i != mFrom && !isVisited(i) && mDistance[i] < min) {
            min = mDistance[i];
            node = i;
        }
    }
    return node;
}

/**
* if the giving node is visited
* @param node   the node need to check
* @return if the node is visited
*/
bool Djikstra::isVisited(const int & node) {
    return mVisited[node];
}

/**
 * set the giving node visited
 * @param node  the node which has been visited
 */
void Djikstra::setVisited(const int & node) {
    mVisited[node] = true;
}

/**
 * update the distance for each node after choosing one nearest node
 * @param chooseNode    the node that newly visited
 * @param map           the data matrix
 */
void Djikstra::updateDistance(const int & chooseNode, void** map) {
    int* _map = (int*)(map + chooseNode * mSize);
    for (register int i = 0; i < mSize; i++) {
        // if distance(a->c) > distance(a->b) + distance(b->c), then take distace(a->b) + distance(b->c)
        int tmp = *(_map + i) ? *(_map + i) : MAX;
        if (mDistance[i] > mDistance[chooseNode] + tmp) {
            mDistance[i] = mDistance[chooseNode] + tmp;
        }
    }
}
```

***
文中有疏漏欢迎指出。
