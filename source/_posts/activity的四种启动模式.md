---
title: activity的四种启动模式
date: 2017-03-30 00:00:00
category: android
tags: android
---


## activity的四种启动模式

本文是学习郭霖《第一行代码》的笔记。

<!-- more -->

**Activity有四种启动模式，分别是**

* **standard**
* **singleTop**
* **singleTask**
* **singleInstance**

***

- #### standard模式
standard模式是默认模式，每当启动一个新的Activity，它就会在返回栈中入栈，并处于栈顶的位置。对于使用standard模式的活动，系统不会在乎这个活动是否已经在返回栈存在，每次启动都会创建Activity的一个新的实例。


- #### singleTop模式
顾名思义，只有栈顶。在启动Activity时如果发现返回栈的栈顶已经是该Activity，则认为可以直接使用它，不会再创建新的Activity实例。

不过当Activity的实例并未处于栈顶位置时，这时再启动Activity，还是会创建新的实例并且压入返回栈。


- #### singleTask模式
singleTop模式可以解决栈顶重复创建Activity实例的问题，但是还是会出现返回栈中存在多个Activity的实例。

若指定singleTask模式，每次启动该Activity时系统首先会在返回栈中检查是否存在该Activity的实例。已存在就直接使用该实例，并把这个实例之上的所有其他Activity的实例统统出栈，没有则会创建一个新的Activity实例。

- #### singleInstance模式
这个模式有点特殊。它会启用一个新的返回栈来管理这个Activity。这样做的意义是，若果程序中有一个Activity允许其他程序调用，要实现共享这个Activity的实例，前面3中启动模式是做不到的，因为每个应用都会有自己的返回栈，同一个Activity在不同的返回栈中入栈时必然是创建了新的实例。而singleInstance模式解决了这个问题，在这种模式下，会有一个单独的返回栈来管理这个Activity，不管事那个应用程序来访问这个Activity，都会共用同一个返回栈，解决了共享Activity实例的问题。


***

<p style="color:red;"><b>To Be Better</b></p>
