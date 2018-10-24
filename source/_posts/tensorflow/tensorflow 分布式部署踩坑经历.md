---
title: tensorflow 分布式部署踩坑经历
date: 2018-10-24 00:01:44
tags: tensorflow
password: 
---

世上本没有坑，挖的人多了，自然就有坑了。
<!-- more -->

公司最近要搭一个分布式集群来训练数据，作为一个无知而又热爱求知的小白，自然被虐得头发都掉了一地。

花了整整2.5个星期后，终于在开源哥们的指导下才大概估计到原因所在，最后才在华为的一个技术贴上找到答案，那时候真是Duang的一声，看着程序终于跑起来的那一刻，真的是想来个夕阳下的奔跑来庆祝一下。这过程真的不容易啊，期间基本把google和百度的资料不管相关和不相关都翻了个遍，也没有很好解决问题，那时候心态是真的爆炸了，最后改了一下关键字，才在谷歌结果的最后一页看到华为的帖子有那么几个字相关，没想到点开后就打开了新世界，激动!!!!!

---
扯淡到此为止，现在来介绍一下部署细节。

本部署是基于ubuntu 16.04 + hadoop + spark + tensorflowOnSpark + tensorflow 1.8

这里集群的环境是1个ps和2个worker

#### hadoop和spark部署
hadoop和spark的部署这里不做叙述，网上的资料一大堆，按照来基本上没有问题，有问题的话再找一个其它靠谱的再配，反正最终能运行就可以了。

#### tensorflow和tensorflowOnSpark安装
这里是采取yarn作为资源管理器，具体安装过程可以参考[官网](https://github.com/yahoo/TensorFlowOnSpark/wiki/GetStarted_YARN)。网上很多文章都说官方说明文档简略，但是其实踩完坑后发觉，其实真的就这么简略，跑不起来，大多是和自身集群的配置有关。所以这些坑得自己填，官方也只能给提示。

一句话，`官方的说明文档是可以跑起来的`！

---
好了，是时候该说一下碰到的坑了。
本人是按官方说明文档，执行到[Run distributed MNIST training (using feed_dict)](https://github.com/yahoo/TensorFlowOnSpark/wiki/GetStarted_YARN#run-distributed-mnist-training-using-feed_dict)这一步就入坑了。

碰到的坑大致如下:

	Timeout while feeding partition

看提示，就是在输入的数据的时候超时了。既然超时了，冒出脑海的第一个想法就是数据能从hadoop上下载下来吗?然后就在代码里加了log来验证，没想到还真的能下载下来，GG，这时候就傻了，既然能下下来，说明和hadoop的通信是没有问题的，那问题的根源在哪里呢？谷歌和百度呗，恭喜，你会发现关于这个问题，搜索的结果要么是不在点上，要么是不相关，要么是广告（你懂的）。这时候就绝望了，log就那么点，搜索的结果都是解决不了问题，只能另辟蹊径，找其它log。

在终端上输入`yarn logs -applicationId <your applicationId>`，然后屏幕上冒出N多的log，若果你运行够久的话，可能会看到类似如下的log

	CreateSession still waiting for response from worker: /job:ps/replica:0/task:0
	CreateSession still waiting for response from worker: /job:worker/replica:0/task:0

没错，这就是本人踩得要死去活来的坑！！！

从log信息来看，当前节点在等ps0和worker0的响应，但是一连串的log说明了这两个节点并没有返回响应，那问题出在哪里呢？goole和baidu呗，网上什么重启，什么worker0完成太快，worker1还没完成所以被挂起，什么`ClusterSpec`配置有问题，什么防火墙没关等等，各种各样的方法都没用，我甚至一度怀疑是ssh有问题(原谅我是小白)，改了也是啥作用都没有啊。总之，反正沾边的链接打开照着做了N次也是没作用，吐了一地的老血。

就这样搞了几天后，在github上看到一个[帖子](https://github.com/yahoo/TensorFlowOnSpark/issues/340)，描述的现象和我还蛮像的，但是提帖子的哥们后面失踪了，也没反馈结果是怎么样。于是自己照着开源哥们提的建议，检查了权限问题，也是没用。这时候实在没办法了，只能自己也跟帖了，开源小哥回复还是蛮快的，指出节点不能相互访问，额，虽然没具体给出解决方法，但起码指明了排查了方向。于是跑了一次原生的tensorflow分布式代码，也是运行失败，至此，问题的根源锁定在tensorflow上，和hadoop什么的一点关系都没有，范围一下缩小了。于是重新谷歌，终于在华为的一个[技术贴](https://bbs.huaweicloud.com/blogs/463145f7a1d111e89fc57ca23e93a89f)上找到原因

	另外一种情况也会导致该问题的发生，从TensorFlow-1.4开始，分布式会自动使用环境变量中的代理去连接，如果运行的节点之间不需要代理互连，那么将代理的环境变量移除即可，在脚本的开始位置添加代码：
	# 注意这段代码必须写在import tensorflow as tf或者import moxing.tensorflow as mox之前

	import os
	os.environ.pop('http_proxy')
	os.environ.pop('https_proxy')


<font color=#0099ff size=7 face="黑体">代理！！！！！！</font>

忽然想起之前写爬虫的时候也被公司的代理坑了一下，但是没放在心上，想不到。。。。。

于是把删除代理配置的代码加入到脚本中，居然<font color=#ff0000 size=7 face="黑体">能运行了</font>。老泪纵横~~

哈哈哈，既然查明原因，接下来就好办了。到每个节点上，把关于代理配置的代码都去掉，重新运行[Run distributed MNIST training (using feed_dict)](https://github.com/yahoo/TensorFlowOnSpark/wiki/GetStarted_YARN#run-distributed-mnist-training-using-feed_dict)的脚本代码, SUCCEED!!!

我的乖乖，网上怎么就没有人碰到这种情况呢.....

除了以上的坑，还遇到过如下的

	waiting for 1 reservations

关于这个坑，我是因为其它节点重启后，没有重新执行启动分布式的脚本，导致重启的节点没有与master联系，master一直在等重启的节点响应。当然[网上](https://blog.csdn.net/jiangpeng59/article/details/72867368)也有说是要`设置spark.cores.max(集群总核数)和spark.task.cpus(worker节点分配核数)满足spark.cores.max/spark.task.cpus=workernumber`，总之，这个检查一下有没出现上述的情况就可以了

	'AutoProxy[get_queue]' object has no attribute 'put'

这个坑也见了好多次，最后是按照这个[帖子](https://github.com/yahoo/TensorFlowOnSpark/issues/248)，在脚本上加上`--executor-cores 1`就ok了，具体原理也没深究...

#### 体会
作为一个毫无经验的小白来说，一开始是按官方说明文档来配置的，发生问题都不知道怎么分析，只能谷歌百度，胡乱配置，所以碰了不少的壁，吃力不讨好。后面还是下决心补了一下基本知识，像tensorflow 分布式里面的ps、worker概念，以及spark，hadoop的基本概念，架构等等，就上手很多了，所谓“工欲善其事必先利其器”，既增长了经验，又解决问题，这点学习时间还是性价比很高的。