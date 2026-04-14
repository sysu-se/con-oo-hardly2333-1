奇怪的问题：
1. 如果将sudoku改成immutable，没法跑通测试，里面对guess的检测是基于原来的sudoku的
A. 如果采用svelte结构，ui是订阅的状态层$store，其实反而不应该用immutable的形式。
和这种订阅机制没有那么吻合。


修改上次作业：
1.修改注释为jsdoc风格
2.Game添加puzzle初始棋盘，添加方法getViewState提供快照，便于UI接入
3.Sudoku添加getInvalidCells方法，将棋盘相关操作整合回类中。