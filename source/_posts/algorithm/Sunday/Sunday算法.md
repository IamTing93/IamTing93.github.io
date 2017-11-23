---
title: Sunday算法
date: 2017-11-23 16:09:47
tags: algorithm
---

&ensp;之前刷字符串匹配的题目，都是用最简单暴力的 -- 一个个字符匹配，效率可想而知。

&ensp;当然网上是有一大堆的字符串匹配算法，如经典的KMP算法，但对于我这种没耐性又笨的人来说，KMP是在难以理解，又难实现。所以一直在寻找一个简单又易懂的替代算法，于是就有了这篇博文。

&ensp;Sunday算法的历史什么的就不过多叙述了，也不与其它算法做太多比较，毕竟嘿嘿。不过这个能称得上简答高效还是有它的道理。

&ensp;字符串匹配的核心思想是**做尽量少的比较来提高匹配速度**。

&ensp;Sunday算法的算法如下：有长为sLen源字符串s，以及长为tLen的需匹配的字符串t。

1. 从源字符串s的第i位开始，自左往右匹配字符串；
2. 

	* 当匹配到不同的字符时，查找s的第i + tLen位字符ch在匹配字符串t中的位置，查找方式为自右往左遍历，到遇到第一个ch为止。假设此时找到ch时，其位置与t最右边的字符距离为dis;

	* 当匹配到t后，i要往右位移tLen位
3. 从源字符串s的第i + dis + 1开始，重复1的操作，直到s的第sLen - tLen位。

&ensp;这里有一个情况，就是ch在t中不存在，那么此时dis = tLen，即从源字符串s的第i + tLen + 1开始匹配, 因为i + 1到i + tLen之间的子字符串肯定不匹配，所以直接从i + tLen + 1开始。

&ensp;以下给出例子：

&ensp;s = "YJABCDEBFK", sLen = 10

&ensp;t = "BC", tLen = 2

![pic1](./pic1.png, "pic1")

如上图片所示，一开始从i = 0开始匹配字符，此时s[0] = Y, t[0] = B，两者不匹配，查找s中下标为i + tLen = 2的字符s[2] = A,从右往左查找A在t中第一次出现的下标，但是A并不存在于t中，所以i要右移到下标位i + tLen + 1 = 3处;



![pic2](./pic3.png, "pic2")

图2所示，i = 3,此时按照算法是会在s匹配到一个BC，匹配后，i需要往右位移tLen = 2位。


![pic3](./pic4.png, "pic3")

图3所示，i = 5,此时s[5] = Y, t[6] = B，两者不匹配,查找s中下标为i + tLen = 7的字符s[7] = B,从右往左查找B在t中第一次出现的下标，为7，此时其位置与t最右边的字符距离为dis = 1，所以i需要往右dis + 1 = 2位，即移动到s[7]的位置。


![pic4](./pic5.png, "pic4")

图4所示，循环遍历s,直到s的第sLen - tLen位，算法结束。

以下给出c++的算法实现。

	int sunday(char* s, int sLen, char* t, int tLen) {

		int next[26];

		for (register int i = 0; i < 26; i++) next[i] = tLen + 1;
	
		// 优化，预先计算匹配字符串中每个字符到最右的距离并加1，可以手动模拟计算验证
		for (register int i = 0; i < tLen; i++) next[t[i] - 'a'] = tLen - i;

		int counter = 0;
		register int ptr = 0;
		for (register int i = 0; i <= sLen - tLen; ) {
			if (s[i + ptr] == t[ptr]) {
				ptr++;
				if (ptr == tLen) {
					counter++;
					ptr = 0;
					i += tLen;
				}
			} else if (i + tLen < sLen) { // 这里的判断条件不能够漏掉，不然可能数组越界
				i += next[s[i + tLen] - 'a'];
				ptr = 0;
			} else {
				break;
			}
		}
		return counter;
	}


时间复杂度o(n)

***
文中有疏漏欢迎指出。
