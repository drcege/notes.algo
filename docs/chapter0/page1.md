# 框架思维

## 数据结构的存储方式

只有两种：
- 数组：顺序存储
- 链表：链式存储

用递归思想分析问题，自顶向下，从抽象到具体。
数组和链表是【结构基础】，散列表、堆、栈、队列、树、图等，都是【上层建筑】。

- 队列、栈，既可以用链表也可以用数组实现：
    - 数组需要处理扩容、缩容
    - 链表不需要，但需要额外的空间存储指针
- 图，也有两种表示方法：
    - 邻接矩阵是二维数组，可以快速判断连通性，适合矩阵运算，但稀疏图浪费空间
    - 邻接表是链表，节省存储空间，但很多操作不方便
- 散列表，通过散列函数将键映射到数组，需要解决散列冲突：
    - 线性探查法，基于数组连续寻址，但操作稍微复杂
    - 拉链法，基于链表特性，操作简单，但需要额外空间存储指针
- 树
    - 数组实现为【堆】，堆是【完全二叉树】；不需要存储指针，操作简单
    - 链表实现为常规【树】，不一定是完全二叉树，不适合数组存储
        - 在【链表树】结构上衍生出二叉搜索树、AVL 树、红黑树、区间树、B 树等

总结：各种数据结构的底层存储只有两种，数组、链表。



|     | 优点 | 缺点 |
| --- | --- | --- |
| 树组 | 随机访问、节省存储 | 扩容时间复杂度 O(N)、中间插入删除 O(N) |
| 链表 | 不存在扩容问题、给定前驱插入删除 O(1) | 不能随机访问、额外存储指针 |



## 数据结构的基本操作

遍历及【增删改查】

遍历方式：
- 线性：for/while 迭代
- 非线性：递归

数组遍历，线性迭代：
```python
def traverse(arr: List[int]):
    for i in range(len(arr)):
        print(arr[i])
```

链表遍历，两种：
```python
class ListNode:
    def __init__(self, val):
        self.val = val
        self.next = None

# 迭代
def traverse(head: ListNode):
    p = head
    while p is not None:
        print(p.val)
        p = p.next

# 递归
def traverse(head: ListNode):
    if head is not None:
        print(head.val)
        traverse(head.next)
```

二叉树，递归：
```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right
 
 def traverse(root: TreeNode):
    traverse(root.left)
    traverse(root.right)
```

N 叉树，递归：
```python
class TreeNode:
    def __init__(self, val=0, children=None):
        self.val = val
        self.children = children
 
 def traverse(root: TreeNode):
    for child in root.children:
        traverse(child)
```

图遍历实际上是 N 叉树遍历的扩展，通过 visited 标记避免环。

## 算法刷题指南

数据结构是工具，算法是通过合适的工具解决特定问题的方法。先了解常用的数据结构，熟悉各自的特性和缺陷。

学习顺序：
1. 首先学习数组、链表等基本数据结构的常用算法，如单链表翻转、前缀和树组、二分搜索等
2. 其次二叉树，培养框架思维；特别是递归算法技巧，本质上都是树的遍历
3. 之后再看回溯、动态规划、分治等算法专题
