## HW 问题收集

列举在HW 1、HW1.1过程里，你所遇到的2\~3个通过自己学习已经解决的问题，和2\~3个尚未解决的问题与挑战

### 已解决

1. 为什么组件中能在import grid后既用这个名字访问棋盘，又能够调用明明是function里面的非这个名字变量的其他函数？
   1. **上下文**：
   一开始我在看 Actions.svelte 和 Keyboard.svelte 时，发现组件里虽然只是 import { grid, userGrid, invalidCells } from ... ，但却既能通过 $grid 读取棋盘，又能调用 grid.undo()、grid.generate() 这类方法。很奇怪于实际上这些状态方法明明是在function createGrid中实现的，却没有出现其函数名。
   2. **解决手段**：
   通过询问CA，我了解到此处import的grid不是普通数组，而是一个已经通过createGame实例化后的自定义store对象，它可以通过$grid方式读取其中的状态容器，也可以通过grid.xx()来调用。如果是import方法，每个组件里都是新的一个grid实例，反而没法同步了。

2. 为什么既可以通过键盘输入数字，上下左右移动光标，又可以通过鼠标点击提供的数字来输入和移动？
   1. **上下文**：
   一开始我在看 Keyboard.svelte 时，发现页面上既有键盘输入，又有鼠标点击按钮输入，而且两种方式最后都能改变棋盘和光标。我一开始不太清楚这两种交互是如何统一到同一套逻辑里的，也不清楚是怎么识别键盘输入和分类的。
   2. **解决手段**：
   将keyboard.svelte的代码要求CA来解析后，我了解到键鼠输入实际不是分别实现两套逻辑，而是都收敛到一个入口handleKeyButton，通过svelte架构提供的svelte window on:keydown的语法监听输入，按照类型区分不同功能。而鼠标按钮本质上是在on:click后调用键盘逻辑的handleKeyButton(num)函数，从而完成了：按键检测 -> 统一入口 -> 修改状态 的处理方式。

### 未解决

1. 我尝试用一个统一的gameStore取代旧的userGrid, grid等状态容器，但在状态的职责和组件的依赖上遇到了问题
   1. **上下文**：
   一开始我想把 gameStore 直接作为唯一状态容器，取代原来的 userGridState，希望让 UI 只订阅一个更统一的 store，并由它统一管理 grid、userGrid、invalidCells 和操作命令。这样看起来结构更集中，但在实际接入时，组件里原本依赖 userGrid 的地方很多很杂，我大致替换后出现了游戏无法正常初始化的问题。

   2. **尝试解决手段**：
   我一开始选择反复询问CA，反馈我的需要，根据提示将一些可能有关的组件代码里面的userGrid, grid等都用gameStore的相关状态取代，如Welcome.svelte, game.js中的startNew函数等，但最后都会出现在选择难度弹窗后无法正常结束弹窗开始游戏。
   我之后又尝试询问CA取消弹窗到显示棋盘游戏 的stores层到UI层 的逻辑路线，了解到从弹窗到棋盘显示的过程是 app.svelte打开欢迎弹窗， welcome.svelte里点Start后调用startNew等函数，然后执行hideModal()关闭弹窗。而其中startNew在game.js中实现，会调用原先grid的一些方法。 自己在以上文件相关步骤都一一筛查后仍未果，遂改用最小实现方法，利用adapt层作为中间层处理。
   现在考虑来，应该是没有处理与userGrid, grid派生出来一些其他变量及其相关逻辑，尤其是gameWon这种散落在外面的要订阅的量，从而导致半成品的各种奇怪bug。

2. canRedo/canUndo等Game类功能 还没有能与UI层接上
   1. **上下文**：
   我在 grid.js 里加了 canUndo() 和 canRedo()，但实际 Actions.svelte 里按钮的禁用条件目前主要还是 gamePaused，不会因为栈空而禁止按钮禁用。
   2. **尝试解决手段**：
   我尝试将按钮禁用逻辑也和canUndo canRedo关联起来，但和问题1的情况一一，由于按钮、暂停、弹窗、启动流程耦合比较紧，改动一处又会出现各种奇怪的bug，多次问CA修复无果，决定先保留现有行为，没有额外暴露多的状态容器。