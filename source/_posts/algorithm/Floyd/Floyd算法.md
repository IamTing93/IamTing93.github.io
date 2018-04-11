---
title: Floyd算法
date: 2018-04-12 16:09:47
tags: algorithm
---

本文将简单介绍求带权的多源最短路径长度的Floyd算法

<!-- more -->
其实关于Floyd算法，网上就有一篇很通俗易懂，生动有趣的博文《[坐在马桶上看算法：只有五行的Floyd最短路算法](http://developer.51cto.com/art/201403/433874.htm)》，所以这里就不打算重复造轮子了。

还是谈谈对这个算法的理解。

这个算法，说白了，就是对每两个点使用规则：中转点b能否使得`dist(a->z) < dist(a->b) + dist(b->z)`成立,成立的话，那么a和z之间的距离就为dist(a->b) + dist(b->z)，然后再遍历其它中转点c,d,e,f等，看上述规则是否成立。这个与[Djistra算法](https://iamting93.github.io/2018/04/10/algorithm/Djikstra/Djikstra%E7%AE%97%E6%B3%95/)有点类似，不过Djistra算法是单源的。

所以真的很好理解，代码很好写，不过代价就是O(n^3)的时间复杂度。

以下是核心代码。
```cpp
for(k = 1; k <= n; k++) { // k是用来遍历中转点的，要放在最外层循环
    for(i = 1; i <= n; i++) {
        for(j = 1; j <= n; j++) {
            if(dist[i][j] > dist[i][k] + dist[k][j]) {
                dist[i][j] = dist[i][k] + dist[k][j];
            }
        }
    }
}
```

***
文中有疏漏欢迎指出。
