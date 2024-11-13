# 4. 持久的单链栈

好吧，我们已经掌握了可变单链堆栈的艺术。

让我们通过编写一个持久的，也就是不可变的单链栈，从单一所有权转变为共享所有权。这正是函数式程序员所了解和喜爱的列表。您可以获取头部或尾部，并将某人的头部放在其他人的尾部上……并且……基本上就是这样。不变性是一种地狱般的毒品。

在此过程中，我们将主要熟悉`Rc`和`Arc`，但这将为我们准备下一个将改变游戏规则的列表。

让我们添加一个名为`third.rs`的新文件：

```rust
// in lib.rs

pub mod first;
pub mod second;
pub mod third;
```

这次没有复制粘贴。这一次是全新操作。

## 4.1 布局

好吧，回到布局的绘图板上。

持久列表最重要的一点是，你可以基本自由地操作列表的尾部：

例如，在持久列表中，这是一个常见的工作负载：

```txt
list1 = A -> B -> C -> D
list2 = tail(list1) = B -> C -> D
list3 = push(list2, X) = X -> B -> C -> D
```

但最后我们希望内存看起来像这样：

```txt
list1 -> A ---+
              |
              v
list2 ------> B -> C -> D
              ^
              |
list3 -> X ---+
```

这在`Box`中根本行不通，因为节点`B`的所有权是共享的。谁应该释放它？如果我删除了`list2`，它会释放节点`B`吗？对于`Box`，我们当然会期望如此！

函数式语言（实际上几乎所有其他语言）都通过使用垃圾回收来避免这种情况。借助垃圾回收的魔力，只有当每个人都不再查看节点`B`时，它才会被释放。好极了！

Rust没有任何类似这些语言的垃圾收集器。它们有跟踪GC，它会在运行时挖掘所有闲置的内存并自动找出哪些是垃圾。相反，Rust目前只有引用计数。引用计数可以被认为是一种非常简单的GC。对于许多工作负载，它的吞吐量明显低于跟踪收集器，如果您设法构建了循环引用，它就会完全失效。但是嘿，这就是我们所拥有的一切！值得庆幸的是，对于我们的用例，我们永远不会遇到循环引用（请随意尝试向自己证明这一点——我肯定不会）。

那么我们如何进行引用计数垃圾收集？使用`Rc`！`Rc`就像`Box`，但我们可以复制它，并且只有当从它派生的所有`Rc`都被删除时，它的内存才会被释放。不幸的是，这种灵活性的代价是严重的：我们只能对其内部进行共享引用。这意味着我们永远无法真正从我们的列表中获取数据，也无法改变它们。 那么我们的布局会是什么样子？好吧，以前我们的代码是这样的：

```rust
pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

我们可以只将`Box`改为`Rc`吗？

```rust
// in third.rs

pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Rc<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

```txt
> cargo build

error[E0412]: cannot find type `Rc` in this scope
 --> src/third.rs:5:23
  |
5 | type Link<T> = Option<Rc<Node<T>>>;
  |                       ^^ not found in this scope
help: possible candidate is found in another module, you can import it into scope
  |
1 | use std::rc::Rc;
  |
```

哦，该死，太恶心了。与我们用于可变列表的所有东西不同，`Rc`太差劲了，甚至没有被隐式导入到每个Rust程序中。真是个失败者。所以我们必须手动添加导入语句。

```rust
use std::rc::Rc;
```

```txt
> cargo build

warning: field is never used: `head`
 --> src/third.rs:4:5
  |
4 |     head: Link<T>,
  |     ^^^^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: field is never used: `elem`
  --> src/third.rs:10:5
   |
10 |     elem: T,
   |     ^^^^^^^

warning: field is never used: `next`
  --> src/third.rs:11:5
   |
11 |     next: Link<T>,
   |     ^^^^^^^^^^^^^
```

看起来是合法的。Rust的编写仍然非常简单。我敢打赌，我们可以找到`Box`并将其都替换为`Rc`，然后就大功告成了！

...

不。我们不能。

## 4.2 基本操作

我们现在已经了解了很多Rust的基础知识，所以我们可以再次做很多简单的事情。

对于构造函数，我们可以再次复制粘贴：

```rust
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }
}
```

因为是不可变的列表，`push`和`pop`不再有意义。相反，我们可以提供`prepend`和`tail`方法，它们提供的功能大致相同。

让我们从`prepend`开始。就是在前方添加元素。它接受一个列表和一个元素，并返回一个列表。与可变列表的情况一样，我们想要创建一个新节点，该节点的下一个值是旧列表。唯一新颖的事情是如何获取下一个值，因为我们不允许改变任何东西。

我们祈祷的答案是`Clone`特征。`Clone`由几乎所有类型实现，并提供了一种通用方法来获取“另一个与此类似的”逻辑上不相交的，仅给出一个共享引用。它就像C++中的复制构造函数，但它从未被隐式调用，必须总是显式调用。

`Rc`特别使用`Clone`作为增加引用计数的方式。因此，我们只需克隆旧列表的头部`head`，代替将`Box`移动到子列表中。我们甚至不需要对`head`进行匹配，因为`Option`公开了一个`Clone`实现，它正好可以完成我们想要做的事情。

好吧，让我们试一试：

```rust
    pub fn prepend(&self, elem: T) -> List<T> {
        List { head: Some(Rc::new(Node {
            elem: elem,
            next: self.head.clone(),
        }))}
    }
```

```txt
> cargo build

warning: field is never used: `elem`
  --> src/third.rs:10:5
   |
10 |     elem: T,
   |     ^^^^^^^
   |
   = note: #[warn(dead_code)] on by default

warning: field is never used: `next`
  --> src/third.rs:11:5
   |
11 |     next: Link<T>,
   |     ^^^^^^^^^^^^^
```

哇，Rust在实际使用字段方面真的很严格。它可以说没有消费者可以真正观察到这些字段的使用！不过，到目前为止，我们似乎还不错。

`tail`是此操作的逆向逻辑，即获取列表的尾部。它接受一个列表并返回删除第一个元素的整个列表。所有这些都是从列表中的第二个元素（如果存在）开始的克隆。让我们试试这个：

```rust
    pub fn tail(&self) -> List<T> {
        List { head: self.head.as_ref().map(|node| node.next.clone()) }
    }
```

```txt
> cargo build

error[E0308]: mismatched types
  --> src/third.rs:27:22
   |
27 |         List { head: self.head.as_ref().map(|node| node.next.clone()) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `std::rc::Rc`, found enum `std::option::Option`
   |
   = note: expected type `std::option::Option<std::rc::Rc<_>>`
              found type `std::option::Option<std::option::Option<std::rc::Rc<_>>>`
```

嗯，我们搞砸了。同样是返回`Option<Y>`，`map`方法的函数参数需要返回`Y`，但这里我们给出的函数参数（即`clone()`方法）返回的是`Option<Y>`。幸运的是，这是另一种常见的`Option`模式，我们只需使用`and_then`方法即可返回一个`Option`。（而在逻辑上，`and_then`和更常用的`or_else`方法更像是一对反义词，定义如下）

```rust
pub fn and_then<U, F>(self, f: F) -> Option<U>
where
    F: FnOnce(T) -> Option<U>,
```

只改个方法名很简单

```rust
    pub fn tail(&self) -> List<T> {
        List { head: self.head.as_ref().and_then(|node| node.next.clone()) }
    }
```

```txt
> cargo build
```

太棒了。 现在我们有了`tail`，我们可能应该提供`head`，它返回对第一个元素的引用。这只是从可变列表中`peek`转换过来的：

```rust
pub fn head(&self) -> Option<&T> {
    self.head.as_ref().map(|node| &node.elem)
}
```

```txt
> cargo build
```

很好！

这些功能已经足够了，我们可以测试一下：

```rust
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn basics() {
        let list = List::new();
        assert_eq!(list.head(), None);

        let list = list.prepend(1).prepend(2).prepend(3);
        assert_eq!(list.head(), Some(&3));

        let list = list.tail();
        assert_eq!(list.head(), Some(&2));

        let list = list.tail();
        assert_eq!(list.head(), Some(&1));

        let list = list.tail();
        assert_eq!(list.head(), None);

        // Make sure empty tail works
        let list = list.tail();
        assert_eq!(list.head(), None);
    }
}
```

```txt
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 5 tests
test first::test::basics ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test third::test::basics ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured
```

完美！

`Iter`也与我们的可变列表相同：

```rust
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head.as_deref() }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_deref();
            &node.elem
        })
    }
}
```

```rust
    #[test]
    fn iter() {
        let list = List::new().prepend(1).prepend(2).prepend(3);

        let mut iter = list.iter();
        assert_eq!(iter.next(), Some(&3));
        assert_eq!(iter.next(), Some(&2));
        assert_eq!(iter.next(), Some(&1));
    }
```

谁曾说过动态类型更容易？

（笨蛋说过）

请注意，我们无法为这种不可变类型实现`IntoIter`或`IterMut`特征。我们只能共享访问元素。

## 4.3 实现`Drop`特征

与可变列表一样，我们有一个递归析构函数问题。诚然，对于不可变列表来说，这个问题并不那么严重：如果我们在某个地方碰到另一个列表的头部节点，我们不会递归地删除它。然而，这仍然是我们应该关心的事情，如何处理还不太清楚。下面是我们对可变列表解决这个问题的方法：

```rust
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

问题在于循环主体：

```rust
    cur_link = boxed_node.next.take();
```

这是在`Box`内改变`Node`，但我们不能用`Rc`做到这一点；它只给我们共享访问权限，因为任何数量的其他`Rc`都可能指向它。 但如果我们知道我们是最后一个知道这个节点的列表，那么将`Node`移出`Rc`实际上是可以的。然后我们也可以知道什么时候停止：当我们无法提升出`Node`时。

看看这个，`Rc`有一个方法可以做到这一点：`try_unwrap`：

（原文）
> `pub fn try_unwrap(this: Rc<T>) -> Result<T, Rc<T>>`
>
> Returns the inner value, if the Rc has exactly one strong reference.
>
> Otherwise, an Err is returned with the same Rc that was passed in.
>
> This will succeed even if there are outstanding weak references.
>
> Examples
>
> ```rust
> use std::rc::Rc;
> 
> let x = Rc::new(3);
> assert_eq!(Rc::try_unwrap(x), Ok(3));
> 
> let x = Rc::new(4);
> let _y = Rc::clone(&x);
> assert_eq!(*Rc::try_unwrap(x).unwrap_err(), 4);
> ```rust

（翻译）
> `pub fn try_unwrap(this: Rc<T>) -> Result<T, Rc<T>>`
>
> 如果`Rc`正好有一个强引用，则返回内部值。
>
> 否则，返回的`Err`将与传入的`Rc`相同。
>
> 即使存在突出的弱引用，此操作也将成功。
>
> 示例
>
> ```rust
> use std::rc::Rc;
> 
> let x = Rc::new(3);
> assert_eq!(Rc::try_unwrap(x), Ok(3));
> 
> let x = Rc::new(4);
> let _y = Rc::clone(&x);
> assert_eq!(*Rc::try_unwrap(x).unwrap_err(), 4);
> ```rust

（就用它试一下）

```rust
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut head = self.head.take();
        while let Some(node) = head {
            if let Ok(mut node) = Rc::try_unwrap(node) {
                head = node.next.take();
            } else {
                break;
            }
        }
    }
}
```

```txt
> cargo test
   Compiling lists v0.1.0 (/Users/ADesires/dev/too-many-lists/lists)
    Finished dev [unoptimized + debuginfo] target(s) in 1.10s
     Running /Users/ADesires/dev/too-many-lists/lists/target/debug/deps/lists-86544f1d97438f1f

running 8 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 8 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

太棒了！真棒。

## 4.4 使用`Arc`

使用不可变链表的一个原因是跨线程共享数据。毕竟，共享可变状态是万恶之源，解决这个问题的一种方法是永远杀死可变部分。

但我们的列表根本不是线程安全的。为了实现线程安全，我们需要原子地处理引用计数。否则，两个线程可能会尝试增加引用计数，但只有一个会发生。然后列表可能会过早被释放！

为了获得线程安全，我们必须使用`Arc`。除了引用计数是原子修改的之外，`Arc`与`Rc`完全相同。如果您不需要它，这会产生一些开销，因此Rust会同时公开两者。我们需要做的就是将对`Rc`的每个引用替换为`std::sync::Arc`。就是这样。我们是线程安全的。完成了！

但这提出了一个有趣的问题：我们如何知道类型是否是线程安全的？我们会不小心搞砸吗？

不！你不会在Rust中搞砸线程安全！

之所以如此，是因为Rust以一流的方式通过两个特性来模拟线程安全性：`Send`和`Sync`。

- 实现了`Send`的类型，可以安全地在线程间传递所有权。也就是说，可以跨线程移动。
- 实现了`Sync`的类型，可以安全地在线程间传递（可变或共享）引用。也就是说，可以跨线程共享。

如果可以安全地移动到另一线程，则类型被标记为`Send`。如果可以安全地在多个线程之间共享，则类型被标记为`Sync`。也就是说，如果`T`是`Sync`，则`&T`是`Send`。在这种情况下，安全意味着不会引起数据竞争（不要将其与更普遍的竞争条件问题混淆）。

这些是标记特征，这是一种奇特的说法，即它们是完全不提供任何接口的特征。您要么是`Send`，要么不是。这只是其他API可能需要的一个属性。如果您不是适当的`Send`，那么静态地不可能发送到不同的线程！太棒了！

`Send`和`Sync`也是自动派生的特征，具体取决于您的类型是否完全由`Send`和`Sync`类型组成。这类似于如果您仅由`Copy`类型组成，则只能实现 `Copy`，但是如果您是，我们就会继续自动实现它。

几乎每种类型都是`Send`和`Sync`。大多数类型都是`Send`，因为它们完全拥有自己的数据。大多数类型都是`Sync`，因为跨线程共享数据的唯一方法是将它们放在共享引用后面，这使它们不可变！

然而，有些特殊类型违反了这些属性：具有内部可变性的类型。到目前为止，我们只真正与继承的可变性（又称为外部可变性）进行了交互：值的可变性是从其容器的可变性继承而来的。也就是说，你不能仅仅因为你喜欢就随意改变某个非可变值的字段。

内部可变性类型违反了这一点：它们允许您**通过共享引用进行改变**。内部可变性主要分为两类：单元（主要包括`Cell`和`RefCell`），仅在单线程上下文中工作；锁（主要包括`Mutex`和`RwLock`），在多线程上下文中工作。出于显而易见的原因，当您可以使用它们时，单元额外的开销更小。还有原子（即`std::sync::atomic`包中的一系列结构体），它们是像锁一样工作的原语。

那么所有这些与`Rc`和`Arc`有什么关系？好吧，它们都使用内部可变性来获取引用计数。更糟糕的是，这个引用计数在每个实例之间共享！`Rc`只使用一个`Cell`，这意味着它不是线程安全的。`Arc`使用了原子，这意味着它是线程安全的。当然，您不能通过将类型放入`Arc`中来神奇地使它线程安全。`Arc`只能像任何其他类型一样派生线程安全。

我真的真的真的不想深入了解原子内存模型或非派生`Send`实现的细节。不用说，随着你对Rust线​​程安全故事的深入了解，事情变得越来越复杂。作为高级消费者，一切都很正常，你真的不需要考虑它。

## 4.5 最终代码

这就是我要说的关于不可变堆栈的全部内容。现在我们已经非常擅长实现列表了！

```rust
use std::rc::Rc;

pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Rc<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn prepend(&self, elem: T) -> List<T> {
        List { head: Some(Rc::new(Node {
            elem: elem,
            next: self.head.clone(),
        }))}
    }

    pub fn tail(&self) -> List<T> {
        List { head: self.head.as_ref().and_then(|node| node.next.clone()) }
    }

    pub fn head(&self) -> Option<&T> {
        self.head.as_ref().map(|node| &node.elem)
    }

    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head.as_deref() }
    }
}

impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut head = self.head.take();
        while let Some(node) = head {
            if let Ok(mut node) = Rc::try_unwrap(node) {
                head = node.next.take();
            } else {
                break;
            }
        }
    }
}

pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_deref();
            &node.elem
        })
    }
}

#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn basics() {
        let list = List::new();
        assert_eq!(list.head(), None);

        let list = list.prepend(1).prepend(2).prepend(3);
        assert_eq!(list.head(), Some(&3));

        let list = list.tail();
        assert_eq!(list.head(), Some(&2));

        let list = list.tail();
        assert_eq!(list.head(), Some(&1));

        let list = list.tail();
        assert_eq!(list.head(), None);

        // Make sure empty tail works
        let list = list.tail();
        assert_eq!(list.head(), None);
    }

    #[test]
    fn iter() {
        let list = List::new().prepend(1).prepend(2).prepend(3);

        let mut iter = list.iter();
        assert_eq!(iter.next(), Some(&3));
        assert_eq!(iter.next(), Some(&2));
        assert_eq!(iter.next(), Some(&1));
    }
}
```
