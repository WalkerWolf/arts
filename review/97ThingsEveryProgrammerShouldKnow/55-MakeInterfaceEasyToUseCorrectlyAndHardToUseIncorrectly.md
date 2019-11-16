# 让接口易于正确使用并难以出错

[Make Interfaces Easy to Use Correctly and Hard to Use Incorrectly](https://97-things-every-x-should-know.gitbooks.io/97-things-every-programmer-should-know/content/en/thing_55/)

软件开发中最常见的一项任务就是接口规范。接口可以是顶层抽象(用户接口)，可以是底层抽象(函数接口)，也可以在两者之间(类接口、库接口等等)。不论你是否向最终用户说明过他们该如何与某个系统交互，或是向开发者解释如何调用API，或者给类声明私有函数，接口设计都会是你工作中非常重要的一部分。如果你做得好，接口就会被乐于使用，增加生产率。如果你做得不好，接口就会是抱怨和错误的源泉。

好的接口应该是这样的：

*易于正确使用*
> 人们总能正确地使用优秀的接口设计，因为路径阻力最小。对于图形界面，他们总是可以正确地点击到图标、按钮、或者菜单入口，因为它太明显也太容易了。对于API，开发者总能正确地传递对的参数和对的值，因为它看起来理所当然。面对那些易于被正确使用的接口——直接用就是了。

*难以用错*
> 好的接口可以预测人们可能犯的错，并让它难以朝那方面想，几乎不可能去提交这种错误。一个图形界面可能会禁用或者益处那些在当前上下文中没有意义的命令。例如，API可以通过“允许传递任意顺序的参数”来消除传递参数时的顺序问题。

设计易于正确使用的接口的好办法是在它们出现之前就开始使用它们。模拟一个图形界面——可以是一个白板或者桌上的索引卡片——在你创建任何底层代码之前就先开始用它们。再比如，在函数被声明之前就应该先去写如何调用它的代码。浏览常见的用例，并指定你期望该接口会产生的行为。你想点击什么？你想通过什么？易于使用的接口看起来总那么自然，因为它能让你做你想做的。如果你从用户视角去开发它们，你更有可能做出像样的接口。(这个观点也是测试优先编程(test-first programming)的优势之一)

要让接口难以用错需要做两件事。首先，你必须预测用户可能犯的错，并找到阻止它们的方法。第二，你必须留意在早起发布期间接口是如何被滥用的，然后修改接口——是的，修改接口！——以阻止那些错误。禁止错误使用的最佳途径是让它们不可能这么用。如果用户坚持撤销一个不可能的撤销动作，就试着让这个动作变得可以撤销。如果它们坚持传入一个错误的值到API里，尽力修改API以让用户传递想要的值。

综上，记住，一个接口的存在是为了方便它们用户，而非它们的实现者。