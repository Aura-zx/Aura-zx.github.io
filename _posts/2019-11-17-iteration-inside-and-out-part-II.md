---
layout:     post
title:      "[译]深入浅出Iteration(下)"
subtitle:   "从迭代到并发"
date:       2019-11-17
author:     "Zhx"
tags:
    - 技术
---

# [译]深入浅出Iteration(下)

> 原文：https://journal.stuffwithstuff.com/2013/01/13/iteration-inside-and-out/
>
> 作者：**Bob Nystrom**
>
> 你以为在聊迭代，其实我在说并发

除非你觉得自己很勇敢，否则可能会想先阅读[第一部分](https://journal.stuffwithstuff.com/2013/01/13/iteration-inside-and-out/)。

在上篇中，我们知道了迭代包括两部分代码：一部分生产值，另一部分消费值。在循环中，这两部分在生产，消费，生产，消费的循环中轮流旋转，就像某种怪异的增量衔尾蛇。这意味着一个块必须完全返回并展开它的调用栈帧，然后才能移交给下一个。

我们也知道了内部迭代器和外部迭代器能解决一部分问题但是解决不了另一部分问题。这其中的差异归结为哪些代码块中有更多有用的东西存储在调用栈中。

对于外部迭代器，消费值的代码可以控制调用栈，所以它可以很好地解决大多数复杂性在于消费值时的问题。例如，短路或者交织多组迭代器在处理起来就很轻松。

反过来，内部迭代器控制了生成值的过程，能很好处理复杂性在于生成值的问题，例如，遍历一个树的节点。

现在，我们将了解一些解决这些问题的技术。核心思路是 [*reification*](http://en.wikipedia.org/wiki/Reification_(computer_science))。如果你要保留一些在调用栈上的数据，需要找到一个存储它们的地方。

### Iterators and generators

假设我们要定义一个连接两个序列的方法。但实际上不想创建包含两个元素的数据结构，只需要返回一个遍历第一个序列然后遍历第二个序列的迭代器。下面是做这些事情的 C# 代码：

```c#
IEnumerable Concat(IEnumerable, IEnumerable b) {
  return new ConcatEnumerable(a, b)
}

class ConcatEnumerable
{
  IEnmuerable a;
  IEnmuerable b;
	ConcatEnumerable(IEnumerable a, IEnumerable b)
  {
    this.a = a
    this.b = b
  }
  
  IEnumerable GetEnumerator()
  {
    return new ConcatEnumerator(
      a.GetEnumerator(), b.GetEnumerator());
  }
}

class ConcatEnumerator
{
  IEnumerator a;
  IEnumerator b;
  bool onFirst = true;
  ConcatEnumerator(IEnumerator a, IEnumerator b)
  {
    this.a = a;
    this.b = b;
  }
  
  bool MoveNext()
  {
    // which sequence are we on?
    if (onFirst)
    {
      // Stay on first
      if (a.MoveNext()) return true;
      
      // Move to the next sequence
      onFirst = false
      return b.MoveNext();
    }
    
    // On second
    return b.MoveNext();
  }
  
  Object Current
  {
    get { return onFirst ? a.current : b.current; }
  }
}
```

哎，对于这么简单的目标来说，似乎是一堆代码。问题出在我们需要保持所有这些状态：正在迭代的两个序列，我们在其中的序列以及我们在其中的位置。这是一个 *外部* 迭代器，不能只将它存储为堆栈中的局部变量，因为必须从每个项目之间的 `MoveNext` 返回。

但是！但是！但是！ C# 中有一个叫`iterators` 的东西。(使人迷惑的名字。其他语言中叫`iterators` 的东西在 C# 中叫 `enumerators`。`iterator` 在 C# 中是一个特殊的东西。)上面的代码也能写成下面那样：

```C#
IEnumerable Concat(IEnumerable a, IEnumerable b)
{
  foreach (var item in a) yield return item;
  foreach (var item in b) yield return item;
}
```

上面的代码有什么改善呢？(备注：如果你的老板按代码行付费给你，则需要避免这种情况。)关键在于 `yield return`，一个方法有这个语句时，编译器会把这个方法转换为 `iterator`。可以认为它是一个“可恢复的方法”，调用 `Concat()` 时，会执行到第一个 `yield` 停止，恢复时会从 `yield` 之后开始。

在上面的代码中，遍历第一个序列，在每个项目处停止并返回。然后对第二个序列做同样的事情。但什么是“恢复”方法？

返回类型说明了这个问题，调用 `Concat()` 返回了 `IEnumerable` 类型，它是 C# 中的 [可迭代序列类型](https://docs.microsoft.com/en-us/dotnet/api/system.collections.ienumerable?redirectedfrom=MSDN&view=netframework-4.8)，所以你得到的返回值是一个“集合”，“恢复方法”意味着“获取集合中的下一个方法”。

如上所述，我们得到了一个很好的解决方法，可以像下面这样使用上面的方法：

```c#
foreach (var item in Concat(stuff, moreStuff))
{
  Console.WriteLine(item);
}
```

使用 `yield` 让我们可以存储所有需要的状态，两个序列和在序列中的当前位置，作为 `Concat()` 方法中的局部变量。C# 会 *reify* 这变量，以便在 `Concat()` 方法"返回"时使用，数据被安全的储存起来了。

你可能会好奇这是如何做到的，有点可笑，在 C# 中，答案是编译器本身将自动生成一个小隐藏类，就像 `ConcatEnumerator` 一样。runtime([CLR](http://en.wikipedia.org/wiki/Common_Language_Runtime)) 并不支持 `yield`，纯粹是通过自动生成代码来完成的，它一块很甜的[语法糖](https://en.wikipedia.org/wiki/Syntactic_sugar)。

在上篇中，我精心制作了一些精美的 ASCII 艺术，以显示所有状态的存储位置。主要问题是生成值的代码的状态和使用它们的代码的状态都存储在堆栈中。使用 iterators 就像这样：

```
 stack                       heap
+---------------------+
| iterator.MoveNext() |
+---------------------+     +-------------------+
| loop body           | --> | DesugaredIterator |
+---------------------+     +-------------------+
  ...

  main()
```

在栈中有消费值的代码状态，但是生成值所需要的状态存储在 *堆* 中。编译器为我们创建了这个小类的实例。当 `MoveNext()` 返回时不会清除状态，因为这些状态存储在堆中，在下次调用 `MoveNext()` 时可以继续使用。

还有其他几种语言会(或将要)以这种方式工作。Python 称为"[generators](https://www.python.org/dev/peps/pep-0255/)"，使用了相似的 `yield` 语法；下个版本的 JavaScript 也会有[类似的东西](http://wiki.ecmascript.org/doku.php?id=harmony:generators)。发明 generators 和 `yield` 关键词的语言是 Barbara Liskov 的 [CLU](http://en.wikipedia.org/wiki/CLU_(programming_language))，在 [Tron](https://en.wikipedia.org/wiki/List_of_Tron_characters#CLU) 中永垂不朽。

不过这里有一点限制，你只能从方法自身 `yield` 。假设我们(出于某种原因)想要组织 C# 代码，例如：

```c#
IEnumerable Concat(IEnumerable a, IEnumerable b)
{
  WalkFirst(a);
  WalkSecond(b);
}

IEnumerable WalkFirst(IEnumerable a)
{
  foreach (var item in a) yield return item;
}

IEnumerable WalkSecond(IEnumerable a)
{
  foreach (var item in a) yield return item;
}
```

我们 *想要* 的效果是 `WalkFirst()` 和 `WalkSecond()` 的 `yield return` 使 `Concat()` 也 `yield return`，但这是行不通的。iterator/生成器会 reify 堆栈框架，但是只会 reify 一个，如果你想让迭代的调用逻辑也支持 yield，需要通过遍历每个级别的序列来手动地 reify 。就像这样：

```c#
IEnumerable Concat(IEnumerable a, IEnumerable b)
{
  foreach (var item in WalkFirst(a)) yield return item;
  foreach (var item in WalkSecond(a)) yield return item;
}

IEnumerable WalkFirst(IEnumerable a)
{
  foreach (var item in a) yield return item;
}

IEnumerable WalkSecond(IEnumerable a)
{
  foreach (var item in a) yield return item;
}

```

你看到我在所有的 `Walk_` 方法和 `Concat` 中都使用了 `foreach` 和 `yield return` 了吗？我们显式地让栈的*每个层级*都成为 iterator。这样可以确保每个调用帧都可以成为我们需要的那样。有更好的解决方案吗？

### Python 3.3: Delegating generators

上面的例子翻译成 Python 会是这样：

```python
def concat(a, b):
  for item in walkFirst(a): yield item
  for item in walkSecond(b): yield item
    
def walkFirst(a):
  for item in a: yield item
    
def walkSecond(a):
  for item in b: yield item
```

除了更简洁之外，这就是一行行地从C# 代码映射过来的。但是 Python 3.3 增加了[新东西](http://www.python.org/dev/peps/pep-0380/)在这里可以用到，让我们用一下：

```python
def concat(a, b):
  yield from walkFirst(a)
  yield from walkSecond(b)
  
def walkFirst(a):
  for item in a: yield item

def walkSecond(b):
  for item in b: yield item
```

`concat()`中显式的循环换成了新的 `yield from` 语法。不错，生成器的构成更加简洁了。但是没有任何魔法的地方，依旧需要在迭代代码中的每一层都使用 `yield`。

在大多数情况下，这只是有点冗余。但是如果是在高阶函数中会妨碍代码重用，如果你的函数需要回调，它很难知道那个回调是否需要 yield，最终可能不得不有两种实现，需要 yield 和 不需要 yield。

所以，`yield from` 只是一个小小的提升，还能做得更好吗？

### Ruby: Enumerables, Enumerators, and Fibers

如果你在玩“哪种语言比 X 有更好的特性”，Ruby 通常是一个安全的选项。Matz 精选了 Smalltalk 和 Lisp 的众多功能。(也别忘了 Perl，在 Ruby 灵感池旁嬉戏的丑小鸭。)

在 Ruby 中，迭代通常是内部迭代器。遍历集合的惯用方式是传一个 [block](https://eli.thegreenplace.net/2006/04/18/understanding-ruby-blocks-procs-and-methods/) (如果你不熟悉 Ruby / Smalltalk，多少可以理解成一个回调) 给集合的 `each` 方法。当然它也支持外部迭代器和 `for` 表达式。令人印象深刻的是，它可以将前者转换为后者。(在任何语言中，反过来都是不重要的。)

让我们深入挖掘之前上篇中那个遍历树的例子。下面是 Ruby 中定义树并进行中序遍历的代码：

```ruby
class Tree
  attr_accessor :left, :label, :right

  def initialize(left, label, right)
    @left = left
    @label = label
    @right = right
  end

  def each(&code)
    @left.each &code if @left
    code.call(self)
    @right.each &code if @right
  end
end
```

可以这样使用：

```ruby
tree.each { |node| puts node.label }
```

这就在用内部迭代，传了 `{ |node| ...}` 块，树自己递归地遍历节点并在每个节点上调用回调。

现在假设我们希望它成为一个外部迭代器。也许我们想平行地遍历两棵树来判断是否有相同的标签，大概是这样的代码：

```ruby
class Tree
  # Mixin all of the enumerable methods to our class.
  include Enumerable
end

a = some tree...
b = another tree...

if a.zip(b).each.all? { |pair| pair[0] == pair[1] }
  puts "Equal!"
end
```

`zip` 方法从左边取一个枚举，再右边取另一个，一次将它们“压缩”在一起。结果是成对的元素数组。例如，zip `[1, 2, 3]` 和 `['a', 'b', 'c']` ，结果是 `[[1, 'a'], [2, 'b'], [3, 'c']]`。

接下来的 `all` 方法呢？遍历一个数组，对每一个元素使用传进去的 block。如果每一个元素上的 block 都返回 `true`，则 `all?` 也返回 `true`。

但是这有一个微妙的问题。`zip` 方法在执行任何操作之前将它的参数转换为 *数组*。同时，在树上调用 `each` 方法可以不浪费内存的情况下递增生成值，但是接下来将它放在 `zip` 方法中会分配一个大数组来存储这些值，如果对比的两个树很大，则会浪费很多内存。

我们想要的是一种在不创建任何中间数组的情况下*迭代*遍历这两棵树的方法。树的 `each` 方法

可以做到，但是是内部迭代。外部迭代完美适合这种情况，能转换成它吗？在 Ruby 中，可以很简单的这样做：

```ruby
a = some tree...
b = another tree...

a_enum = a.to_enum
b_enum = b.to_enum
```

`to_enum` 方法接受一个实现了 `each` 方法的对象并返回一个外部迭代器。可以像下面这样使用：

```ruby
loop do 
  if a_enum.next != b.enum.next
    puts "Not equal!"
  end
end
```

这里的约定是 `next` 返回序列中的下一个元素。如果没有下一个元素，会抛出 `StopIteration` 错误，(`loop`可以很方便的处理这个错误。)

特别好，现在对外部迭代和内部迭代都支持很好，并可以轻松地在它们之间来回转换。就像吃蛋糕和派作为甜点。剩下的一个问题是，这是怎么做到的，看看我们现在有什么：

1. 内部迭代器(树上的每个 `each` 方法)递归地调用自身并构建一个深层的调用栈。在此期间的任何时候，它都可以生成值。
2. `to_enum` 方法接收1中的对象并返回一个可枚举对象，调用 `next` 方法它运行这个递归代码，然后无论何时生成值都会挂起。
3. 下一次调用 `next` 方法它会从上一次挂起的地方开始，整个栈会被冻结，在每次调用 `next` 时解冻。

一定有某种表示整个调用栈的数据结构。它不会像生成器一样对栈中的帧做调整，而是处理整个栈。

### A fiber by any other name

这个神秘的数据结构，Ruby 称之为 *fiber* (纤程)。它有点像线程，因为它表示正在进行的计算。它有调用栈，局部变量等等。但是，不像“真正”的线程，它不涉及操作系统，内核调度以及其它所有重量级的东西。

它是非抢占式并且纤程之间可以很好的共处。如果你要运行一个纤程，则必须对其进行控制，不能反过来。

这些结构的名称与支持他们的语言一样多。Lua 称为“[coroutines](http://www.lua.org/pil/9.html)”(这个我认为是这个点子最老的名字了)。 [Stackless Python](http://www.stackless.com/) 称之为“tasklets”。Go 语言的“goroutines”也是类似的东西，尽管有一些有趣的差异。

这些是 `to_enum` 需要的特殊调味料。当你调用它时就启动了一个新的纤程，然后在纤程上运行内部迭代器。当你在枚举器上调用 `next` 时，它将运行该纤程，直到生成一个值。然后纤程挂起，返回这个值，再运行主纤程。当我们需要下一个值时，它挂起主纤程并再次恢复生成值的纤程。

换句话说，`to_enum`的一个简单实现是这样的：

```ruby
class Object
  def to_enum
    MyEnumerator.new self
  end
end
```

MyEnumerator 类(简化自 [这个优秀的 StackOverflow 回答](http://stackoverflow.com/a/1437678/9457))：

```ruby
class MyEnumerator
  include Enumerable

  def initialize(obj)
    @fiber = Fiber.new do  # Spin up a new fiber.
      obj.each do |value|  # Run the internal iterator on it.
        Fiber.yield(value) # When it yields a value, suspend
                           # the fiber and emit the value.
      end
      raise StopIteration  # Then signal that we're done.
    end
  end

  def next
    @fiber.resume          # When the next value is requested,
                           # resume the fiber.
  end
end
```

### Iteration or concurrency?

我开始这两篇文章时，提出了一个有关如何使迭代变得美观且易于使用的问题。这导致我们既需要内部迭代又需要外部迭代，并且想要从一种迭代到另一种迭代的能力。真正完成这项工作所需的部分是创建新的调用栈的简便方法，纤程是一个很好的答案。

但我发现有趣的是我们到达的地方。一开始讨论的是*迭代*，最基础的控制流结构。但是看看我们在哪里结束，纤程是*并发*的机制，并发是比较深层的语言特性了。

我不认为这是一个巧合。如果你仔细看一下迭代，它实际上是关于并发的。你有两个“线程”行为：一个是生产值，另一个是消费值。你需要同时运行这两个线程并进行协调，这就是并发。我们已经习惯了，所以不会这样想。

### Wait, what about Magpie?

我自己在这里搞砸了。这两篇文章的秘密使命是通过承诺向你介绍一些你可能在实际生活中使用的其他语言，以欺骗你阅读我的语言。

如果我们都很幸运，你确实学到一些东西，但是我还没有搞定我的语言。我已经输入了5000个单词，我相信这已经耗尽你的耐心。我猜 Magpie 不得不等待之后的帖子。不过请相信我，它会*很棒*，我保证。

