
A.领域对象如何被消费

    1.
    View层直接消费的是 store adapter, adapter内部持有Game对象，通过subscribe给各组件订阅。
      `具体来说，我们在grid.js中创建只属于该adapter的game实例，并新建了几个writable的状态，它们会通过syncFromGame()在每次操作后从game处更新。如此，我们就可以利用createGrid(),将gridState和相关方法包装成一个obj，我们用grid的名字来向外暴露。这样，View层就可以通过$grid订阅gridState或者grid.xxx()来调用我们提供的方法，每当调用方法时，会针对内部的game实例修改，再同步到几个状态中，进而被View层的组件订阅，消费。其他状态容器也是类似。
    
    2.
    View层拿到的数据就是我们在grid.js提供的几个状态容器：gridState, userGridState, invalidCells，以及在game.js中提供的gameWon(尽管实现了isWon的方法，但是相关组件牵连比较多，还没有一一修改好，于是先不整合回grid.js中统一提供。)

    3.用户操作如何进入领域对象？
     a. 用户选中方格后键盘输入数字/点击数字 -> Keyboard.svelte :若是点击数字,识别on:click后调用handleKeyButton；若是键盘输入，直接通过svelte中的keydown检测，确定为数字调用handleKeyButton。 之后 handleKeyButton调用userGrid.set -> grid.js :在createUserGrid()中的set中调用 game.guess 。如此就进入了领域对象game中。
     b. 用户点击Undo/Redo按钮 -> Actions.svelte: 识别on:click后调用grid.undo/redo -> grid.js: 在createGrid()中调用game.undo/redo 。如此就进入了领域对象game中。

    4.
    组件通过使用暴露的$grid,$userGrid,$invalidCells，订阅了对应的状态容器 gridState, userGridState, invalidCellsState, 当组件调用 这些对象提供的方法后， createGrid(), createUserGrid()内部调用相应操作game实例的函数，然后执行syncFromGame函数，而syncFromGame函数会对这三个writable的状态容器调用set，利用game里的数据更新这些状态容器，这样就会触发subscribe机制，让组件中所有使用了$grid ...的这些地方重新渲染更新。

B.响应式机制说明

    1.我主要依赖的是 store机制：
    通过writable的对象, store adapter层，UI组件调用$store后进行的自动subscribe机制，以及每次调用修改领域对象的方法后对writable的set, 实现UI的自动更新。
    我少量依赖的是 $:，这是在原项目中一些组件内部自动重新计算用到的。如Actions.svelte中$: hintsAvailable = $hints >0。但这不是领域对象接入Ui的核心。

    2.
    (grid.js) : grid, userGrid, invalidCells, 
    (game.js) : gameWon, gamePaused
    
    3.sudoku, puzzle, undoStack, redoStack, 领域方法guess/undo/redo/check等。

    4.如果直接修改内部对象而没有经过grid.set，可能回因为没有调用syncFromGame，导致同步链条断开，UI可能不刷新或者存在不一致的地方。

C.改进说明
    
    1.
    (1)保留下来的改进：
      a.Game类里面添加了puzzle初始棋盘，又添加了getViewState快照输出方法，便于syncFromGame的实现。
      b.Sudoku类里添加了getInvalidCells方法，把散落在grid.js中的逻辑操作相关方法整合进类里。
      c.在grid.js中将UI交互入口统一为store adapter.

    (2)未保留下来的改进：
      a.我一开始想将sudoku改成immutable的对象，按理说在同步时都是直接set()的，没啥影响，但是貌似上次的test在检测guess时是按照直接查看原先sudoku某一地方是否更改作为评判标准，故作罢。

    (3)针对HW1中review做的调整：
      a.Game直接暴露可变Sudoku:
        这次改进将UI入口移到了store adapter,不会直接拿game对象修改。此外，getSudoku()也改成了返回clone()。
      b.核心规则没有被强制执行：
        部分改善，将很多规则相关逻辑添加进类里了，如getInvalidCells，但guess还是直接写格子，没有内部强制check，但是Ui层似乎不限制不合规数字，它会自行check后给数字标红或者标白
      c.未建模题面固定格与玩家输入的区别
        部分改善，在Game中添加了puzzle，以及getViewState来表达区别，但是固定格还是主要体现在视图中，没有在领域层做出完整不可编辑对象。
      d.保存与恢复API契约不统一
        这次改进在Game里也统一用toJSON和fromJSON了，不再用save/recover风格
      e.构造函数没有守住棋盘形状和数值范围
        没有显式验证二维数组的形状、数值范围等，但有在注释里标注出输入数据要求。
      f.领域对象混入控制台输出副作用
        将Sudoku中的print方法删去了。
      g.冲突坐标使用字符串协议
        仍保留了原协议，但是将职责迁移到了sudoku类中，实现了职责回收。

    2.
      a.HW1中只是提供了可以测试的领域对象，但真实UI组件实际没有订阅该领域，还是执行原先的逻辑
      b.HW1中没有把getInvalidCells方法添加进Sudoku类内，让HW1中的类无法提供UI层所需的足够的状态。

    3.
    优点是接入了Redo/Undo操作，将散落的逻辑整合进领域对象中，使得边界较为清晰。
    缺点是多了一层adapter，同时syncFromGame函数需要维护，在领域对象改变后需要修改，在每个修改game的入口都要调用，较为繁琐。