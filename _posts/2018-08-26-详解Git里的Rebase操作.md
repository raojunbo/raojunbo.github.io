---
layout: post
title: 详解Git里的Rebase操作
date: 2018-08-10 23:34:15.000000000 +09:00
tags: iOS技术
---

在命令行里git rebase --help 可以查看英文版本的解释
下面将对这个英文的文档做输出

假定下面历史记录
![Xnip2018-08-27_19-14-08](http://7nj246.com1.z0.glb.clouddn.com/Xnip2018-08-27_19-14-08.png)

当前在topic分支上
执行 
```git rebase master``` 
或者
 ```git rebase master topic ```
 
将得到如下结果
![Xnip2018-08-27_19-14-18](http://7nj246.com1.z0.glb.clouddn.com/Xnip2018-08-27_19-14-18.png)

NOTE:```git rebase master topic``` 形式是```git checkout topic ,git rebase master```两条命令的相继执行。

如果upstream分支(在这里举例的就是master分支,因为topic从master分支开分支而来)已经包含一个改变的提交，这个提交将会跳过。（A,A`是相同的改变，不同的提交信息）
![Xnip2018-08-27_19-14-28](http://7nj246.com1.z0.glb.clouddn.com/Xnip2018-08-27_19-14-28.png)

执行 ```git rebase master```

将会得到
![Xnip2018-08-27_19-14-36](http://7nj246.com1.z0.glb.clouddn.com/Xnip2018-08-27_19-14-36.png)

下面将介绍如何使用rebase --onto将一个分支的base迁移到另一个分支。
假设现在是如下历史记录
![Xnip2018-08-27_19-14-48](http://7nj246.com1.z0.glb.clouddn.com/Xnip2018-08-27_19-14-48.png)

我们要得到如下结果
![Xnip2018-08-27_19-14-58](http://7nj246.com1.z0.glb.clouddn.com/Xnip2018-08-27_19-14-58.png)

执行 git rebase --onto master next topic 

另一种使用
![Xnip2018-08-28_09-55-15](http://7nj246.com1.z0.glb.clouddn.com/Xnip2018-08-28_09-55-15.png)
执行 git rebase --onto master topicA topicB

（我对这条的理解，checkout到topicB,取topicA到topcB多出来的变更，以master为新的基础）
![Xnip2018-08-27_19-15-17](http://7nj246.com1.z0.glb.clouddn.com/Xnip2018-08-27_19-15-17.png)

另一种使用
假设当前历史记录
![Xnip2018-08-27_19-15-25](http://7nj246.com1.z0.glb.clouddn.com/Xnip2018-08-27_19-15-25.png)
执行 git rebase --onto topicA~5 topicA~3 topicA 
得到如下结果
![Xnip2018-08-27_19-15-33](http://7nj246.com1.z0.glb.clouddn.com/Xnip2018-08-27_19-15-33.png)

在rebase操作时，如果有冲突，git rebase将停止让你解决冲突，解决后git add ,在执行git rebase --continue,直到没有冲突为止。想取消rebase可以执行git rebase --abort;

从上面的总结，可以看出，rebase,其内在含义就是变基础（起点）。有两个参数，一个是”找到变基的提交“。另一个是指“定变基的到哪”。



