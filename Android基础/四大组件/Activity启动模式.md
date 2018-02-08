### 概述

上篇我稍微介绍了关于activity的生命周期的知识点，今天我们来谈一下activity的启动模式，这也是一个面试常客。下面容我慢慢道来。



###  什么情况下会用到启动模式？

举个简单的例子，来推送通知之后，我们点击推送跳转页面，如果我们一次来了10个推送，那么我们会先后打开10个页面。但是这10个页面是同一个。这种情况下，我们就需要通过启动模式去解决。



### Activity的LaunchMode

关于activity的启动模式，目前总共有四种：standard、singleTop、singleTask和singleInstance，下面我们来简单介绍下，四个启动模式的具体含义是什么：

- standard：标准模式。这是Android模式的启动模式，所以对这个词陌生也算正常。每次启动一个activity都会创建该实例，不管当前activity是否存在。也就是说可以有多个相同的activity在栈中重叠。也就是上述例子的情况下。
- singleTop：栈顶复用模式。在这种模式，如果当前activity已经位于栈顶，此时，当前activity将不再重新创建。同时它的onNewIntent方法会被调用。此时，当前activity的生命周期不会重置。如果当前activity已经存在，但是不位于栈顶，此时会重新创建。假设目前栈内是ABCD，我将D设为singleTop属性，再次启动D的时候是ABCD而不是ABCDD。
- singleTask：栈内复用模式。在这种模式，如果当前activity已经位于栈内，此时，当前activity将不再重新创建。同时它的onNewIntent方法也会被调用。和singleTop一样。但是和singleTop的区别是重新启动时，它会移除在当前页面上面的所有activity。因为singleTask有clearTop效果。假设目前栈内是ABCD，将B设为singleTask属性，再次启动B的时候是AB而不是ABCDB。
- singleInstance：单实例模式。这个模式除了单独给当前activity开一个栈，它具备了所有singleTask的属性。具体什么地方用到。不明。。。



### Activity的flags

- FLAG_ACTIVITY_CLEAR_TOP：例如现在的栈情况为：A B C D 。D此时通过intent跳转到B，如果这个intent添加FLAG_ACTIVITY_CLEAR_TOP 标记，则栈情况变为：A B。如果没有添加这个标记，则栈情况将会变成：A B C D B。也就是说，如果添加了FLAG_ACTIVITY_CLEAR_TOP 标记，并且目标Activity在栈中已经存在，则将会把位于该目标activity之上的activity从栈中弹出销毁。这跟上面把B的Launch mode设置成singleTask类似。

- FLAG_ACTIVITY_NEW_TASK：例如现在栈1的情况是：A B C。C通过intent跳转到D，并且这个intent添加了FLAG_ACTIVITY_NEW_TASK 标记，如果D这个Activity在Manifest.xml中的声明中添加了Task affinity，并且和栈1的affinity不同，系统首先会查找有没有和D的Task affinity相同的task栈存在，如果有存在，将D压入那个栈，如果不存在则会新建一个D的affinity的栈将其压入。如果D的Task affinity默认没有设置，或者和栈1的affinity相同，则会把其压入栈1，变成：A B C D，这样就和不加FLAG_ACTIVITY_NEW_TASK 标记效果是一样的了。      

  注意如果试图从非activity的非正常途径启动一个activity，比如从一个service中启动一个activity，则intent比如要添加FLAG_ACTIVITY_NEW_TASK 标记。

- FLAG_ACTIVITY_NO_HISTORY：例如现在栈情况为：A B C。C通过intent跳转到D，这个intent添加FLAG_ACTIVITY_NO_HISTORY标志，则此时界面显示D的内容，但是它并不会压入栈中。如果按返回键，返回到C，栈的情况还是：A B C。如果此时D中又跳转到E，栈的情况变为：A B C E，此时按返回键会回到C，因为D根本就没有被压入栈中。

- FLAG_ACTIVITY_SINGLE_TOP：和上面Activity的 Launch mode的singleTop类似。如果某个intent添加了这个标志，并且这个intent的目标activity就是栈顶的activity，那么将不会新建一个实例压入栈中。

