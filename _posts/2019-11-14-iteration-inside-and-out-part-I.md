---
layout:     post
title:      "[译]深入浅出Iteration(上)"
subtitle:   "两种迭代模式"
date:       2019-11-14
author:     "Zhx"
tags:
    - 技术
---

# [译]深入浅出Iteration(上)

> 原文：https://journal.stuffwithstuff.com/2013/01/13/iteration-inside-and-out/
>
> 作者：**Bob Nystrom**
>
> External Iterator VS Internal Iterator

你知道迭代，知道 *循环* 的所有知识，认为这些都是编程语言中已经解决的问题。认真地说，下面的 *FORTRAN* 代码执行一个循环并且可以在50年前的计算机上执行：

```fortran
do i=1,10
	print i
end do
```

所以当我开始为我的小编程语言 [Magpie](http://magpie-lang.org/) 设计循环时，我发现这是一件做起来非常直接的事情：

1. 看一堆其他的编程语言
2. 看看最厉害的设计是什么
3. 照着做

当然，现在的第一个难题是，这不只是要循环一定次数，或者只是根据一系列数字进行循环。那是小孩都能做的事情，即使 C 语言都可以做。

这关系到 *迭代* ：可以生成和使用任意序列的东西。它不只是“列表中的每一个元素”，它是“一个树的所有叶子节点”，或者“文件中的所有行”，或者“质数”。因此，这里有一个隐含的抽象层：你需要能够定义“迭代”对自己用途的含义。

我的发现使我感到惊讶。事实证明，在其他语言里，存在两种完全独立不相关的迭代形式。 [Gafter and the Gang of Four](http://gafter.blogspot.com/2007/07/internal-versus-external-iterators.html) (同时也是一个杰出的乐队名) 把他们称为“内部(internal)”或“外部(external)”迭代器，听起来很花哨。

每一种形式都只是在一些使用场景非常完美，另外的场景则令人生厌。他们就像阴阳，或者小孩和玩耍。

### External iterators: OOPs, I did it again.

硬币的第一面是 *外部* 迭代器。如果你使用 C++，Java，C#，Python，PHP 或者几乎任何 [single-dispatch](http://en.wikipedia.org/wiki/Dynamic_dispatch#Single_and_multiple_dispatch) 的面向对象的语言都有这样的迭代器。这些语言一般提供 `for` 或者 `foreach` 语句，就像这样：

```dart
var elements = [1, 2, 3, 4, 5];
for (var i in elements) print(i);
```

（如果你好奇是什么语言，[Dart](http://www.dartlang.org/)）

编译器看到的会有一点不一样。如果你有尼奥的能力，那么像上面的循环真实样子是：

```dart
var elements = [1, 2, 3, 4, 5];
var __iterator = elements.iterator();
while (__iterator.moveNext()) {
  var i = __iterator.current;
  print(i);
}
```

`.iterator()`，`.moveNext()`，和 `.current` 这三个统称为*迭代协议*。如果你要定义你自己可以迭代的东西，你的类型需要支持这个协议。因为编译器对 `for` 语句会把它编译成这样(如果你喜欢 PL nerd 的术语也可以称为“[desugars](http://en.wikipedia.org/wiki/Syntactic_sugar)”)，支持这个协议可以让你的类型完美融合到循环中。

在静态类型语言中，这个"协议"实际上是显式的接口：

- Java: [`Iterable`](http://docs.oracle.com/javase/1.5.0/docs/api/java/lang/Iterable.html)
- C#: [`IEnumerable`](http://msdn.microsoft.com/en-us/library/system.collections.ienumerable.aspx)
- Dart: [`Iterable`](http://api.dartlang.org/docs/bleeding_edge/dart_core/Iterator.html)

在动态类型语言中，它更加非正式，就像 Python 的 [iterator protocol](http://docs.python.org/2/library/stdtypes.html#iterator-types)。

##### *Beautiful example 1: Finding an item*

这是一个这种迭代器运行不错的例子。写一个函数当序列中有给定的项目时返回 `true` ，没有则返回 `false`。我将继续使用 Dart 因为我认为 Dart 语言能工作的很好并且大多数程序员都能理解：

```dart
find(Iterable haystack, needle) {
  for (var item in haystack) {
    if (item == needle) return true;
  }
  
  return false;
}
```

十分简单。这段代码的一个关键点是*短路*：只要你发现了目标则停止循环。这不只是一个优化，在考虑某些序列(例如读取文件中的行)可能有副作用或可能有无限序列时也很关键。

##### *Beautiful example 2: Interleaving two sequences*

让我们做一些更复杂的事情，写一个函数，该函数接受两个序列并返回一个序列，这个序列的元素为每个序列的元素交替。例如[1, 2, 3]和['a', 'b', 'c']，结果为`1, 'a', 2, 'b', 3, 'c'`。

```dart
interleave(Iterable a, Iterable b) {
  return new InterleaveIterable(a, b);
}
```

这只是一个对象的委托，因为你需要一个类型去完成迭代器协议，这个类型：

```dart
class InterleaveIterable {
  Iterable a;
  Iterable b;
  InterleaveIterable(this.a, this.b);

  Iterator get iterator() {
    return new InterleaveIterator(a.iterator(), b.iterator());
  }
}
```

好了，又是另一个委托。这是因为许多迭代器协议分割为“可以被迭代的类”与代表*当前*迭代状态的对象分开。前者不能通过迭代修改，而后者可以。现在让我完成真正的代码：

```dart
class InterleaveIterator {
  Iterator a;
  Iterator b;
  InterleaveIterator(this.a, this.b);

  bool moveNext() {
    // Stop if we're done.
    if (!a.moveNext()) return false;

    // Swap them so we'll pull from the other one next time.
    var temp = a;
    a = b;
    b = temp;
    return true;
  }

  get current => a.current;
}
```

这段代码虽然有一点冗长，但还是比较直接的。每一次调用 `moveNext()` ，它从一个迭代器中读取，然后交换迭代器，直到其中一个迭代器完成再停止，很时髦。

##### *Kitten-punch example: Walking a tree*

现在让我们看一个使用这种迭代器会很差的例子。这里有一个简单的二叉树的类：

```dart
class Tree {
  Tree left;
  String label;
  Tree right;
}
```

我们想 *中序* 遍历这个树，意味着我们会递归地打印出每一个左边节点，然后是中间节点，最后是右边节点。实现代码就像下面描述的那样简单：

```dart
printTree(Tree tree) {
  if (tree.left != null) printTree(tree.left);
  print(tree.label);
  if (tree.left != null) printTree(tree.right);
}
```

接着，我们发现需要在遍历过程中做一些事情。或许是将它转化为 JSON ，或者是记录节点的数量或其他事情。我们真正想要的是能够依次遍历节点，然后对每个项目执行我们想做的任何事情，所以上面的功能变成：

```dart
printTree(Tree tree) {
  for (var node in tree) {
    print(node.label);
  }
}
```

为了让上面的代码能工作，`Tree` 需要实现迭代协议。这看起来会像什么样，最好一次吞掉整个苦药：

```dart
class Tree implements Iterable<Tree> {
  Tree left;
  String label;
  Tree right;
  Tree(this.left, this.label, this.right);
  Iterator get iterator => new TreeIterator(this);
}

class IterateState {
  Tree tree;
  int step = 0;
  IteratorState(this.tree);
}

class TreeIterator implements Iterator<Tree> {
  var stack = [];
  TreeIterator(Tree tree) {
    stack.add(new IteratorState(tree));
  }
  
  bool moveNext() {
    var hasValue = false;
    while (stack.length > 0 && !hasValue) {
      var state = stack.last;
      switch (state.step) {
        case 0:
          state.step = 1;
          if (state.tree.left != null) {
            stack.add(new IterateState(state.tree.left));
          }
          break;
          
        case 1:
          state.step = 2;
          current = state.tree;
          hasValue = true;
          break;
          
        case 2:
          state.removeList();
          if (state.tree.right != null) {
            stack.add(new IterateState(state.tree.right));
          }
          break;
      }
    }
    
    return hasValue;
  }
  
  Tree current;
}
```

Sweet Mother of Turing (译:这就不翻译了，笑哭)，这里到底发生了什么，完全相同的行为是三行递归函数，现在是五十行怪物。

一会儿我们将回过头来看这里到底出了什么问题，这里毫无疑问不是一种好的抽象中序遍历的方法。现在让我们换一下口味。

### Internal iterators: Don't Call Me, I'll Call You

现在，Ruby 使用者在笑，Smalltalk 使用者疯狂地挥舞着双手试图获取老师的注意，Lisp 使用者一如既往在后排自鸣得意地点头。或许他们知道一些你不知道的东西：

这些语言(Smalltalk, Ruby, 和 Lisp)使用了*内部* 迭代器，进行迭代时，有两个代码块在起作用：

1. 负责生成一系列值的代码
2. 使用这些值并对它进行处理的代码

在外部迭代器中，(1)是实现迭代器协议的类型，(2)是`for`循环的循环体，这种风格中，(2)决定何时调用(1)以获取下一个值，并且可以随时停止。

内部迭代器则逆转了这样的责任。在内部迭代器中，生成值的代码决定什么时候去调用使用这些值的代码。举个例子，在 Ruby 中，你可以这样打印披头士乐队：

```ruby
beatles = ['George', 'John', 'Paul', 'Ringo']
beatles.each {|beatle| puts beatle}
```

`Array` 的 `each` 方法就是迭代器。它的功能是遍历数组中的每一个元素。`{|beatle| puts beatle}` 是我们想要每一个元素要执行的代码。大括号定义了一个 Ruby 中的 *块* ：你可以传递的 first-class 代码块。 

所以这段代码就是将 `puts` 表达式打包放入对象并将它传给 `each` 方法。然后 `each` 方法遍历数组中的每一个元素并将元素传给代码块。

##### *Beautiful example 1: Walking a tree*

让我们瞧一瞧不优雅的外部迭代器例子在 Ruby 中是什么样的。首先，定义一个树：

```ruby
class Tree
  attr_accessor :left, :label, :right
  
  def initialize(left, label, right)
    @left = left
    @label = label
    @right = right
  end
end
```

要使用内部迭代器风格来遍历这个树，我们希望它可以神奇地工作：

```ruby
tree.in_order {|node| puts node.label}
```

实现这样的迭代器在 Dart (或者 Java，C#)需要大概50行代码。这是 Ruby 的代码：

```ruby
class Tree
  def in_order(&code)
    @left.in_order &code if @left
    code.call(self)
    @right.in_order &code if @right
  end
end
```

这样就完成了，它看起来很像一开始的递归函数，因为它就是很像。唯一的差别是 Dart 函数式硬编码调用 `print()` ，这里使用了 *块* ，基本上是对每一个值调用了回调。事实上，我们可以在任意支持匿名函数的语言上实现相同的东西，这是 Dart 版本的：

```dart
inOrder(Tree tree, callback(Tree tree)) {
  if (tree.left != null) inOrder(tree.left);
  callback(tree);
  if (tree.right != null) inOrder(tree.right);
}
```

你(…[至此](http://openjdk.java.net/projects/lambda/))不能在 Java 中做到这样，但是在大多数 OOP 语言中你可以假冒内部迭代器的风格。它只是在这些语言中不常用。

在这个遍历树的例子中，内部迭代肯定优于外部迭代器风格。让我们看看其他例子。

##### *Beautiful example 2: Finding an item*

好的，假设我们正在使用 Ruby，并且想编写一个方法，该方法在给定任何可迭代对象的情况下，查看它是否包含某些对象。所谓“任意可迭代对象”，是“具有 `each` ”方法的对象，这个方法是迭代的规范方法。就像这样：

```ruby
def container(haystack, needle)
  haystack.each { |item| return true if item == needle }
  false
end
```

让我们把它写成 Dart：

```dart
contains(Iterable haystack, needle) {
  haystack.forEach((item) {
    if (item == needle) return true;
  });
  return false;
}
```

仍然十分简洁！除了一个问题：它事实上不能工作。

差别在哪？在两个例子中，都有这句代码：`return false`。它的意思是使`contains()`方法返回`true`。但是在 Dart 的例子中，`return` 语句包含在一个 lambda 表达式中，一个小的匿名函数：

```dart
(item) {
	if (item == needle) return true;
}
```

它起的作用就是让 *函数* 返回。当他结束时，它返回到 `forEach()` 里，然后继续处理下一个项目。在 Ruby 中，`return` 语句不能返回到包含它的 *块* 里，它返回到包含它的 *方法* 里。`return` 语句将沿着封闭的块，从所有封闭的块返回，直到遇见一个 *方法* ，然后让这个*方法* 返回。

这种特性被称为“[non-local returns](http://yehudakatz.com/2010/02/07/the-building-blocks-of-ruby/)”。Smalltalk 和 Ruby 都有这种特性。如果你想使用内部迭代器并且希望他们能像我们做的这样尽早的终止，你真的需要 `non-local returns`。

这是内部迭代器不是其他语言常用手段的主要原因。如果你的 `each` 和 `forEach` 函数不能提前返回，这确实是一个限制。

##### *Kitten-punching example: Interleaving two sequences*

另一个与外部迭代器配合良好的示例是将两个序列交替出输出在一起。代码有点冗长，但是效果很好，可以与任何一对序列一起使用。让我将风格转换为内部迭代器。这篇文章已经足够长了，所以这个留作练习，快去做完再回来。

...

这么快回来吗？怎么样了？浪费了你多少时间？

是的，据我所知，除非你愿意使用一些重型武器(例如线程或者 `continuations` )，否则根本无法使用内部迭代器解决此问题。你在一条小溪的 *无* 桨船上。

我认为，这是大多数主流语言都使用外部迭代器的重要原因。没错，遍历树的代码确实冗长，但至少是 *可用* 的。(这也可能是为什么有内部迭代器的语言也有`continuations`的原因。)

### 问题出在哪?

我们现在陷入了僵局。外部迭代器在某些方面表现良好，内部迭代器则在另外方面表现良好。为什么没有一个对所有情况都适用的解决方法？问题归结为一个原因：*调用栈(callstack)*。

你可能不会这样考虑，*调用栈是一个数据结构*。每一个帧(例如你现在所在的方法)就像一个对象。局部变量就是这个对象中的字段。

你可以免费获得另外一点额外的数据：当前执行指针。调用栈会跟踪你在方法中的位置。例如：

```dart
lameExample() {
  print("I'm at the top");
  doSomething();
  print("I'm in the middle");
  doSomething();
  print("Dead last like a chump");
}
```

我们认为这是理所当然的，但是每次 `doSometing()` 返回到 `lameExample()` 时，都会回到它离开的地方。这很方便，还记得我们是怎么递归地遍历树：

```dart
printTree(Tree tree) {
  if (tree.left != null) printTree(tree.left);
  print(tree.label);
  if (tree.right != null) printTree(tree.right);
}
```

在左子树上调用 `printTree()` 后，它会恢复到它离开的地方，打印标签，然后去下一个子树。进行递归后，你还是可以看出这个隐式的栈。调用栈本身(也因此得名)将跟踪我们正在遍历的父分支。

当我们将函数转换为外部迭代器风格时，那50行样板代码只是在 [reifying](http://en.wikipedia.org/wiki/Reification_(computer_science)) 调用栈免费提供给我们的数据结构。`IterateState` 类正是每个调用帧存储的内容。其中 `tree` 字段是 `printTree` 函数的 `tree` 参数。`step` 字段是执行指针。`TreeIterator`中的 `stack` 就是调用栈。

这里的经验是调用栈帧是一种非常简洁的状态存储方式。你必须手动将所有内容写下来，才能意识到它对你有多大的作用。如果有人问我你最喜欢的数据结构是什么，我的答案一直是：调用栈。

### 谁控制调用栈？

我们需要了解为什么每种迭代风格适用于处理不同问题，关键就是谁在控制调用栈。前面我说过迭代涉及两个代码块：生成迭代对象的代码和使用迭代对象的代码。在外部迭代器中，调用栈看起来像：

```text
+------------+
| moveNext() |
+------------+
| loop body  |
+------------+
  ...

main()
```

包含循环的方法调用 `moveNext()` ，将它推入栈顶，它可以依次调用所需的任何内容，因此可以 *暂时* 在调用栈上进行控制。但是它必须返回，展开和丢弃所有状态，才能返回到循环体，然后生成下一个值。

这就是为什么在遍历树的例子如此冗长。因为所有存储在调用帧内的状态都会被清除，在这里将 `iterateState` 构成的栈存储在了迭代对象中。这样下次调用 `moveNext()` 时，它仍然存在。

在内部迭代中，调用栈看起来是这样：

```text
+------------------------+
| each									 |
+------------------------+
| method containing loop |
+------------------------+
  ...
  
main()
```

现在迭代对象在栈顶。它可以建立所需要的任何堆栈内容，然后在方便时调用：

```text
+------------------------+
| block                  |
+------------------------+
  stuff...
+------------------------+
| each                   |
+------------------------+
| method containing loop |
+------------------------+
  ...

  main()
```

*block* 代码可以返回到 `each` 中。所以迭代器可以返回任何调用栈需要的状态，一切都在掌握中。但是，正如你看到的，你需要 `non-local return` 特性来保证这一点。因为，当 *block* 想停止迭代，它需要一种方法从头到尾地通过 `stuff` 到 `each`，返回到 `method` 里。

这就是问题所在。在栈顶位置的代码处于较弱的位置，因为它必须在每个生成的值之间一路返回到另一个。在某些用例中，生成值的代码需要这种能力(递归地遍历一棵树)，内部迭代器就工作的非常好。另一些用例中，消费值的代码需要这种能力(交织两组迭代器)，外部迭代器就胜出了。

因为你只有一个调用栈，这是我们能做到的最好情况吗，还是？

查看 [第二部分](https://journal.stuffwithstuff.com/2013/02/24/iteration-inside-and-out-part-2/) 来了解一些语言做了什么尝试来解决这个问题。

