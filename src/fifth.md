# 6. 不安全的单链队列

好吧，引用计数的内部可变性有点失控了。Rust肯定不希望你做这种事情吧？嗯，是也不是。`Rc`和`Refcell`非常适合处理简单的情况，但它们可能变得笨拙。特别是如果你想隐藏它正在发生的情况。一定有更好的方法！

在本章中，我们将回滚到单链表并实现单链队列，以了解原始指针和不安全的Rust。

> **旁白**：我会指出错误。

我们不会犯任何错误。

让我们添加一个名为`fifth.rs`的新文件：

```rust
// in lib.rs

pub mod first;
pub mod second;
pub mod third;
pub mod fourth;
pub mod fifth;
```

我们的代码主要源自第三章的`second.rs`，因为队列在链表世界中主要是栈的增强。不过，我们还是要从头开始写，因为我们想解决一些与布局等相关的基本问题。

## 6.1 布局

那么单链队列是什么样的呢？好吧，当我们有一个单链栈时，我们将数据押入到列表的一端，然后从同一端弹出。堆栈和队列之间的唯一区别是队列从另一端弹出。因此，从我们的栈实现中，我们有：

```txt
input list:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)

stack push X:
[Some(ptr)] -> (X, Some(ptr)) -> (A, Some(ptr)) -> (B, None)

stack pop:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)
```

要创建队列，我们​​只需决定将哪个操​​作移至列表末尾：押入还是弹出？由于我们的列表是单链接的，因此我们实际上可以用相同的努力将任一操作移至末尾。

要将押入`push`的内容移至末尾，我们只需一直走到`None`并使用新元素将其设置为`Some`。

```txt
input list:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)

flipped push X:
[Some(ptr)] -> (A, Some(ptr)) -> (B, Some(ptr)) -> (X, None)
```

如果是要将末尾的项目弹出`pop`，我们只需一直走到`None`之前的节点，然后取它：

```txt
input list:
[Some(ptr)] -> (A, Some(ptr)) -> (B, Some(ptr)) -> (X, None)

flipped pop:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)
```

我们今天可以这样做然后就此作罢，但那会很糟糕！这两个操作都会遍历整个列表。有些人会认为这样的队列实现确实是队列，因为它公开了正确的接口。但是我相信性能保证是接口的一部分。我不关心精确的渐近界限，只关心“快”与“慢”。队列保证押入和弹出是快速的，而遍历整个列表肯定不快。

一个关键的观察结果是，我们浪费了大量的时间在重复做同样的事情。我们可以“缓存”所有这些工作并重复使用吗？当然可以！我们可以存储指向列表末尾的指针，然后直接跳转到那里！

事实证明，只需一次反转押入和弹出操作就能解决这个问题。要反转弹出，我们必须向后移动“尾部”指针，但由于我们的列表是单向链接的，因此我们无法有效地做到这一点。如果我们反转押入，我们只需向前移动“头”指针，这很容易。

我们来尝试一下：

```rust
use std::mem;

pub struct List<T> {
    head: Link<T>,
    tail: Link<T>, // NEW!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // When you push onto the tail, your next is always None
            next: None,
        });

        // swap the old tail to point to the new tail
        let old_tail = mem::replace(&mut self.tail, Some(new_tail));

        match old_tail {
            Some(mut old_tail) => {
                // If the old tail existed, update it to point to the new tail
                old_tail.next = Some(new_tail);
            }
            None => {
                // Otherwise, update the head to point to it
                self.head = Some(new_tail);
            }
        }
    }
}
```

现在，我将加快实现细节的讲解速度，因为我们应该对这类事情相当熟悉。这并不意味着您就一定要期望第一次尝试就能写出这段代码。我只是跳过了我们之前必须处理的一些反复试验。实际上，我在编写这段代码时犯了很多错误，但我没有展示这些错误，但您只能看到我多次省略了`mut`或`;`，直到它不再具有指导意义。别担心，我们会看到很多其他错误消息！

```txt
> cargo build

error[E0382]: use of moved value: `new_tail`
  --> src/fifth.rs:38:38
   |
26 |         let new_tail = Box::new(Node {
   |             -------- move occurs because `new_tail` has type `std::boxed::Box<fifth::Node<T>>`, which does not implement the `Copy` trait
...
33 |         let old_tail = mem::replace(&mut self.tail, Some(new_tail));
   |                                                          -------- value moved here
...
38 |                 old_tail.next = Some(new_tail);
   |                                      ^^^^^^^^ value used here after move
```

瞄准，射击！

（原文）
> use of moved value: new_tail

（翻译）
> 使用了已经移动的值：new_tail

`Box`没有实现`Copy`，所以我们不能直接将它分配到两个位置。更重要的是，`Box`拥有它指向的东西，并会在它被丢弃时尝试调用析构方法释放它。如果我们的押入实现编译成功，我们会双重释放列表的尾部！实际上，正如所写的那样，我们的代码会在每次押入时释放`old_tail`。哎呀！🙀

好吧，我们知道如何创建一个非拥有指针。那只是一个引用！

```rust
pub struct List<T> {
    head: Link<T>,
    tail: Option<&mut Node<T>>, // NEW!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // When you push onto the tail, your next is always None
            next: None,
        });

        // Put the box in the right place, and then grab a reference to its Node
        let new_tail = match self.tail.take() {
            Some(old_tail) => {
                // If the old tail existed, update it to point to the new tail
                old_tail.next = Some(new_tail);
                old_tail.next.as_deref_mut()
            }
            None => {
                // Otherwise, update the head to point to it
                self.head = Some(new_tail);
                self.head.as_deref_mut()
            }
        };

        self.tail = new_tail;
    }
}
```

这里没什么特别的。基本思路和之前的代码一样，只不过我们使用了一些隐式返回的优点来从实际填充`Box`的地方提取尾部引用。

```txt
> cargo build

error[E0106]: missing lifetime specifier
 --> src/fifth.rs:3:18
  |
3 |     tail: Option<&mut Node<T>>, // NEW!
  |                  ^ expected lifetime parameter
```

哦对了，我们需要为类型中的引用赋予生命周期。嗯...这个引用的生命周期是多少？嗯，这看起来像`IterMut`，对吧？让我们尝试对`IterMut`所做的操作，只需添加一个通用的`'a`：

```rust
pub struct List<'a, T> {
    head: Link<T>,
    tail: Option<&'a mut Node<T>>, // NEW!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<'a, T> List<'a, T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // When you push onto the tail, your next is always None
            next: None,
        });

        // Put the box in the right place, and then grab a reference to its Node
        let new_tail = match self.tail.take() {
            Some(old_tail) => {
                // If the old tail existed, update it to point to the new tail
                old_tail.next = Some(new_tail);
                old_tail.next.as_deref_mut()
            }
            None => {
                // Otherwise, update the head to point to it
                self.head = Some(new_tail);
                self.head.as_deref_mut()
            }
        };

        self.tail = new_tail;
    }
}
```

```txt
> cargo build

error[E0495]: cannot infer an appropriate lifetime for autoref due to conflicting requirements
  --> src/fifth.rs:35:27
   |
35 |                 self.head.as_deref_mut()
   |                           ^^^^^^^^^^^^
   |
note: first, the lifetime cannot outlive the anonymous lifetime #1 defined on the method body at 18:5...
  --> src/fifth.rs:18:5
   |
18 | /     pub fn push(&mut self, elem: T) {
19 | |         let new_tail = Box::new(Node {
20 | |             elem: elem,
21 | |             // When you push onto the tail, your next is always None
...  |
39 | |         self.tail = new_tail;
40 | |     }
   | |_____^
note: ...so that reference does not outlive borrowed content
  --> src/fifth.rs:35:17
   |
35 |                 self.head.as_deref_mut()
   |                 ^^^^^^^^^
note: but, the lifetime must be valid for the lifetime 'a as defined on the impl at 13:6...
  --> src/fifth.rs:13:6
   |
13 | impl<'a, T> List<'a, T> {
   |      ^^
   = note: ...so that the expression is assignable:
           expected std::option::Option<&'a mut fifth::Node<T>>
              found std::option::Option<&mut fifth::Node<T>>
```

哇，这真是一条详细的错误消息。这有点令人担忧，因为它表明我们做的事情真的很糟糕。这里有一个有趣的部分：

（原文）
> the lifetime must be valid for the lifetime 'a as defined on the impl

（翻译）
> 生命周期必须在`impl`块上定义的生命周期`'a`内有效

我们从自身借用，但是编译器希望我们持续与`'a`一样长的时间，如果我们告诉它自身确实持续那么长时间会怎样？

```rust
    pub fn push(&'a mut self, elem: T) {
```

```txt
> cargo build

warning: field is never used: `elem`
 --> src/fifth.rs:9:5
  |
9 |     elem: T,
  |     ^^^^^^^
  |
  = note: #[warn(dead_code)] on by default
```

哦，嘿，成功了！太棒了！

我们也来做`pop`吧：

```rust
    pub fn pop(&'a mut self) -> Option<T> {
        // Grab the list's current head
        self.head.take().map(|head| {
            let head = *head;
            self.head = head.next;

            // If we're out of `head`, make sure to set the tail to `None`.
            if self.head.is_none() {
                self.tail = None;
            }

            head.elem
        })
    }
```

并为此编写一个快速测试：

```rust
#[cfg(test)]
mod test {
    use super::List;
    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop(), None);

        // Populate list
        list.push(1);
        list.push(2);
        list.push(3);

        // Check normal removal
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push(4);
        list.push(5);

        // Check normal removal
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), None);
    }
}
```

```txt
> cargo test

error[E0499]: cannot borrow `list` as mutable more than once at a time
  --> src/fifth.rs:68:9
   |
65 |         assert_eq!(list.pop(), None);
   |                    ---- first mutable borrow occurs here
...
68 |         list.push(1);
   |         ^^^^
   |         |
   |         second mutable borrow occurs here
   |         first borrow later used here

error[E0499]: cannot borrow `list` as mutable more than once at a time
  --> src/fifth.rs:69:9
   |
65 |         assert_eq!(list.pop(), None);
   |                    ---- first mutable borrow occurs here
...
69 |         list.push(2);
   |         ^^^^
   |         |
   |         second mutable borrow occurs here
   |         first borrow later used here

error[E0499]: cannot borrow `list` as mutable more than once at a time
  --> src/fifth.rs:70:9
   |
65 |         assert_eq!(list.pop(), None);
   |                    ---- first mutable borrow occurs here
...
70 |         list.push(3);
   |         ^^^^
   |         |
   |         second mutable borrow occurs here
   |         first borrow later used here


....

** WAY MORE LINES OF ERRORS **

....

error: aborting due to 11 previous errors
```

🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀

天哪。

编译器对我们大发雷霆并没有错。我们只是犯了一个Rust大错：我们在自己内部存储了一个指向自己的引用。不知何故，我们设法说服Rust，这在我们的押入和弹出实现中完全合理（我真的很震惊我们确实这么做了）。​​之所以这样做，是因为Rust实际上根本没有指向自己的指针的概念。代码的每个部分在技术上都是正确的（我们可以调用押入和弹出一次），但随后我们创建的荒谬性就会生效，一切都会锁定。

我确信我们写的东西有些用处，但就我而言，它只是语法上有效的胡言乱语。我们说我们包含具有生命周期`'a`的东西，并且`push`和`pop`借用了该生命周期的`self`。这很奇怪，但Rust可以单独查看我们代码的每个部分，并且它没有发现任何规则被破坏。

但是，一旦我们尝试实际使用列表，编译器就会很快说“是的，您已经可变地为`'a`借用了`self`，因此在`'a`结束之前您不能再使用`self`”，但同时“因为您包含`'a`，所以它必须对整个列表的存在有效”。

这几乎是一个矛盾，但有一个解决方案：只要您`push`或`pop`，列表就会将自己“固定”在原地，并且无法再访问。它吞下了自己众所周知的尾巴，升入了一个梦想的世界。

> **旁白**：这本书刚写出来时还不存在，但Rust实际上[将`pin`的概念形式化为有用的东西](https://doc.rust-lang.org/std/pin/index.html)！这可能是自借阅检查器以来对该语言最复杂的补充。但我们不希望我们的列表被固定！
>
> `pin`对于async-await/future/coroutine来说是必需且有用的，因为编译器需要能够将函数的所有局部变量捆绑到某种结构中并将它们存储在某个地方，直到future/coroutine准备好恢复。由于局部变量可以引用其他局部变量，并且我们希望它能够正常工作，这些结构最终可以包含对自身的引用！
>
> 因此，要等待或产生，Rust需要一种能够正确描述和操作固定值的方法。幸运的是，所有这些东西基本上都隐藏在自动编译器中，在正常情况下，没有人真正需要考虑`pin`（甚至`Future`）。主要的例外是，这些东西对于构建和设计像`tokio`这样的异步运行时的人来说非常重要。
>
> 我们不会在这本书中实现异步运行时。我知道我的朋友知道你可以用`Pin`做的各种“酷”（混乱）技巧，但据我所知，我更乐意不知道它们。我会继续告诉自己，把类型固定（`pin`）不是真实的，它们不会伤害我。

我们的`pop`实现暗示了为什么在我们自己内部存储对我们自己的引用可能非常危险：

```rust
    // ...
    if self.head.is_none() {
        self.tail = None;
    }
```

如果我们忘记这样做会怎么样？那么我们的尾部将指向某个已从列表中删除的节点。这样的节点将立即被释放，并且我们将有一个悬垂指针，而Rust应该保护我们免受这种危险！

事实上，Rust正在保护我们免受这种危险。只是以一种非常……迂回的方式。

那么我们能做什么？回到`Rc<RefCell>>`地狱？

拜托。不要。

不，相反，我们要脱离正轨，使用原始指针。我们的布局将如下所示：

```rust
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>, // DANGER DANGER
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

就是这样。不再有那些懦弱的引用计数、动态借用检查的废话！只有真实的、坚硬的、未经检查的、指针。所以全部代码都要以原始指针的方式重写了。

> **旁白**：事实上，这种做法仍然非常危险，但还不到吸取教训的时候。下一节将像往常一样，以艰难的方式吸取教训。

大家一起成为C语言吧。让我们都成为C语言吧。

我到家了。我准备好了。

你好，不安全的（`unsafe`）Rust。

> **旁白**：哇，作者的狂妄自大真是令人难以置信。

## 6.2 不安全的Rust

这是一个严肃、重大、复杂且危险的话题。它太严重了，我为此专门写了[一本书](https://doc.rust-lang.org/nightly/nomicon/)。

简而言之，只要你允许调用其他语言，每种语言实际上都是不安全的，因为你可以让C随意做坏事。是的：Java、Python、Ruby、Haskell……面对外部函数接口 (FFI)，每种语言都非常不安全。

Rust通过将自身分为两种语言来接受这一事实：安全的（`safe`）Rust和不安全的（`unsafe`）Rust。到目前为止，我们只使用过安全的Rust。它完全100%安全……除了它可以FFI到不安全的Rust。

不安全的Rust是安全的Rust的超集。它的所有语义和规则都与安全的Rust完全相同，只是允许你做一些非常不安全的额外事情，并且可能导致困扰C的可怕的未定义行为。

同样，这是一个非常庞大的话题，有很多有趣的极端情况。我真的不想深入研究它（好吧，我想。我读过。[读过那本书](https://doc.rust-lang.org/nightly/nomicon/)）。没关系，因为有了链表，我们实际上可以忽略几乎所有的东西。

> **旁白**：这是个谎言，但在2015年似乎确实是真的。

我们将使用的主要不安全的Rust工具是原始指针。原始指针基本上是C的指针。它们没有固有的别名规则。它们没有生命周期。它们可以为空。它们可以错位。它们可以是悬空的。它们可以指向未初始化的内存。它们可以转换为整数或从整数转换为指针。它们可以转换为指向其他类型。可变性？转换它。几乎所有事情都会发生，这意味着几乎任何事情都可能出错。

> **旁白*：没有固有的别名规则，是吗？啊，青春的纯真。

这是一些糟糕的事情，老实说，如果你永远不必碰这些东西，你会过上更幸福的生活。不幸的是，我们想写链表，而链表很糟糕。这意味着我们将不得不使用不安全的指针。

有两种原始指针：`*const T`和`*mut T`。它们对应着来自C的`const T*`和`T*`，但我们真的不在乎C认为它们意味着什么。你只能将`*const T`解引用`&T`，但就像变量的可变性一样，这只是对错误使用的检查。最多它只是意味着你必须先将`*const`转换为`*mut`。尽管如果你实际上没有权限改变指针的引用，你就会过得很糟糕。

无论如何，当我们编写一些代码时，我们会对此有更好的感受。现在，`*mut T == &unchecked mut T`！

## 6.3 基本操作

> **旁白**：本节有一个潜在的根本错误，因为这是本书的重点。但是，一旦我们开始使用`unsafe`，就有可能做错事，但一切仍然可以通过编译，并且似乎可以正常工作。下一节将指出根本错误。所以，**不要在生产代码中使用本节的内容！**

好吧，回到基础。我们如何构建列表？

之前我们刚刚做了：

```rust
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }
}
```

但是我们不再使用`Option`作为`tail`的类型了，所以会报错：

```txt
> cargo build

error[E0308]: mismatched types
  --> src/fifth.rs:15:34
   |
15 |         List { head: None, tail: None }
   |                                  ^^^^ expected *-ptr, found 
   |                                       enum `std::option::Option`
   |
   = note: expected type `*mut fifth::Node<T>`
              found type `std::option::Option<_>`
```

我们可以使用`Option`，但与`Box`不同，`*mut`是可空的。这意味着它无法从空指针优化中受益。相反，我们将使用空指针`null`来表示`None`。

那么我们如何获取空指针？有几种方法，但我更喜欢使用`std::ptr::null_mut()`。如果您愿意，也可以使用`0`作为`*mut _`，但这似乎太混乱了。

```rust
use std::ptr;

// defines...

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: ptr::null_mut() }
    }
}
```

```txt
> cargo build

warning: field is never used: `head`
 --> src/fifth.rs:4:5
  |
4 |     head: Link<T>,
  |     ^^^^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: field is never used: `tail`
 --> src/fifth.rs:5:5
  |
5 |     tail: *mut Node<T>,
  |     ^^^^^^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/fifth.rs:11:5
   |
11 |     elem: T,
   |     ^^^^^^^

warning: field is never used: `head`
  --> src/fifth.rs:12:5
   |
12 |     head: Link<T>,
   |     ^^^^^^^^^^^^^
```

编译器又报出了没有使用的值警告。嘘，编译器，我们很快就会用到它们。

好吧，让我们继续再次编写押入。这一次，我们不会在插入后抓取`Option<&mut Node<T>>`，而是立即抓取`Box`内部的`*mut Node<T>`。我们知道我们可以安全地做到这一点，因为`Box`的内容有一个稳定的地址，即使我们移动`Box`。当然，这并不安全，因为如果我们只是丢弃（`drop`）`Box`，我们就会有一个指向释放内存的指针（悬垂指针）。

我们如何从普通指针创建原始指针？强制转换！如果将变量声明为原始指针，则普通引用将强制转换为它：

```rust
let raw_tail: *mut _ = &mut *new_tail;
```

我们拥有了所需的所有信息。我们可以将代码大致翻译成之前的参考版本：

```rust
    pub fn push(&mut self, elem: T) {
        let mut new_tail = Box::new(Node {
            elem: elem,
            next: None,
        });

        let raw_tail: *mut _ = &mut *new_tail;

        // .is_null checks for null, equivalent to checking for None
        if !self.tail.is_null() {
            // If the old tail existed, update it to point to the new tail
            self.tail.next = Some(new_tail);
        } else {
            // Otherwise, update the head to point to it
            self.head = Some(new_tail);
        }

        self.tail = raw_tail;
    }
```

```txt
> cargo build

error[E0609]: no field `next` on type `*mut fifth::Node<T>`
  --> src/fifth.rs:31:23
   |
31 |             self.tail.next = Some(new_tail);
   |             ----------^^^^
   |             |
   |             help: `self.tail` is a raw pointer; 
   |             try dereferencing it: `(*self.tail).next`
```

嗯？我们有一个指向`Node`的指针，为什么我们不能获取`next`字段？

当你使用原始指针时，Rust有点混乱。要访问原始指针的内容，它坚持要求我们手动解引用它们，因为这是一个非常不安全的操作。所以让我们这样做：

```rust
    (*self.tail).next = Some(new_tail);
```

```txt
> cargo build

error[E0133]: dereference of raw pointer is unsafe and requires 
              unsafe function or block

  --> src/fifth.rs:31:13
   |
31 |             (*self.tail).next = Some(new_tail);
   |             ^^^^^^^^^^^^^^^^^ dereference of raw pointer
   |
   = note: raw pointers may be NULL, dangling or unaligned; 
     they can violate aliasing rules and cause data races: 
     all of these are undefined behavior
```

这不应该这么难。

还记得我说过不安全的Rust就像是安全的Rust的FFI语言吗？好吧，编译器希望我们明确界定执行FFI的位置。我们有两个选择。首先，我们可以将整个函数标记为不安全的（`unsafe`），在这种情况下，它将变成一个不安全的Rust函数，并且只能在不安全的（`unsafe`）上下文中调用。这不是很好，因为我们希望我们的列表可以安全使用。其次，我们可以在函数内部编写一个不安全的（`unsafe`）块，以界定FFI边界。这样整个函数被声明为是安全的。让我们这样做吧：

```rust
    pub fn push(&mut self, elem: T) {
        let mut new_tail = Box::new(Node {
            elem: elem,
            next: None,
        });

        let raw_tail: *mut _ = &mut *new_tail;

        if !self.tail.is_null() {
            // Hello Compiler, I Know I Am Doing Something Dangerous And
            // I Promise To Be A Good Programmer Who Never Makes Mistakes.
            unsafe {
                (*self.tail).next = Some(new_tail);
            }
        } else {
            self.head = Some(new_tail);
        }

        self.tail = raw_tail;
    }
```

```txt
> cargo build
warning: field is never used: `elem`
  --> src/fifth.rs:11:5
   |
11 |     elem: T,
   |     ^^^^^^^
   |
   = note: #[warn(dead_code)] on by default
```

耶！

有趣的是，到目前为止，这是我们唯一必须编写不安全的块的地方。我们到处都在做原始指针的事情，这是怎么回事？

事实证明，当涉及到不安全的Rust时，Rust是一个庞大的规则律师学究。我们非常合理地希望最大化安全的Rust程序集，因为这些程序是我们可以更加自信的。为了实现这一点，Rust仔细地为不安全划出了一个最小的表面积。请注意，我们使用原始指针的所有其他地方都是对它们进行分配，或者只是观察它们是否为空。

如果你从未真正解引用原始指针，那么这些都是完全安全的做法。你只是在读取和写入一个整数！你唯一可能遇到原始指针问题的情况是当你真正解引用它时。因此Rust表示只有该操作是不安全的，其他一切都是完全安全的。

太棒了。迂腐。但技术上是正确的。

> **旁白**：在世界的另一端，一位硬件工程师感到脊背一阵发凉——肯定有人又坚持认为指针只是整数。她低头看着自己提出的新硬件指针认证方案，流下了眼泪。隔壁的编译器工程师却毫无感觉——他们早就学会了总是穿着厚厚的毛衣。

只有部分指针操作实际上是不安全的，这引发了一个有趣的问题：尽管我们应该用不安全的块来界定不安全的Rust范围，但它实际上取决于在块之外建立的状态。甚至在函数之外！

这就是我所说的不安全污点。只要在模块中使用了`unsafe`，整个模块就会受到不安全的污染。必须正确编写所有内容，以确保不安全的代码的所有不变量都得到维护。

由于存在隐私控制，这种污点是可控的。在我们的模块之外，我们所有的结构字段都是完全私有的，因此没有其他人可以以任意方式干扰我们的状态。只要我们公开的API组合不会导致坏事发生，就外部观察者而言，我们的所有代码都是安全的！实际上，这与FFI的情况没有什么不同。只要它公开一个安全的接口，就没有人需要关心某个Python数学库是否会变成C。

无论如何，让我们继续讨论`pop`，它几乎逐字逐句地引用了参考版本：

```rust
    pub fn pop(&mut self) -> Option<T> {
        self.head.take().map(|head| {
            let head = *head;
            self.head = head.next;

            if self.head.is_none() {
                self.tail = ptr::null_mut();
            }

            head.elem
        })
    }
```

我们再次看到另一种安全是有状态的情况。如果我们无法在此函数中将尾部指针清零，我们将看不到任何问题。但是，对`push`的后续调用将开始写入悬空尾部！

让我们测试一下：

```rust
#[cfg(test)]
mod test {
    use super::List;
    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop(), None);

        // Populate list
        list.push(1);
        list.push(2);
        list.push(3);

        // Check normal removal
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push(4);
        list.push(5);

        // Check normal removal
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), None);

        // Check the exhaustion case fixed the pointer right
        list.push(6);
        list.push(7);

        // Check normal removal
        assert_eq!(list.pop(), Some(6));
        assert_eq!(list.pop(), Some(7));
        assert_eq!(list.pop(), None);
    }
}
```

这只是堆栈测试，但预期的弹出结果发生了反转。我还在最后添加了一些额外的步骤，以确保弹出时不会发生尾指针损坏的情况。

```txt
> cargo test

running 12 tests
test fifth::test::basics ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test fourth::test::into_iter ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 12 passed; 0 failed; 0 ignored; 0 measured
```

金星！

> **旁白**：来了……

**注意**：再次说明一下，虽然代码通过了测试，但是实际是有问题的，所以**不要在生产环境使用本节的代码！**

## 6.4 `miri`

紧张地笑道：这种不安全的东西太简单了，我不知道为什么每个人都这么说。我们的程序运行完美。

> **旁白**：🙂

...真的正确吗？

> **旁白**：🙂

好吧，我们现在编写的是非安全代码，所以编译器也无法帮助我们发现错误。测试可能碰巧成功了，但实际上却在做一些不确定的事情。一些未定义的行为（`Undefined Behavioury`）。

但我们能做什么？我们撬开窗户，偷偷溜出了`rustc`的教室。现在没人能帮我们了。

...等一下，小巷里那个长相可疑的人是谁？

“嘿，小子，你想解释一些Rust代码吗？”

为什么-不？为什么，

“这太疯狂了，它可以验证你的程序的实际动态执行是否符合Rust内存模型的语义。让你大吃一惊......”

什么？

“它会检查你是否做了未定义的行为。”

我想我可以尝试一次用用解释器。

“你安装了rustup吗？”

当然，它是拥有最新Rust工具链的工具！

```txt
> rustup +nightly-2022-01-21 component add miri

> 注意，我使用了这个版本：2023-02-14，另外我用的操作系统是macOs，而作者是windows
> rustup +nightly-2023-02-14 component add miri

info: syncing channel updates for 'nightly-2023-02-14-x86_64-apple-darwin'
info: latest update on 2023-02-14, rust version 1.69.0-nightly (065852def 2023-02-13)
info: downloading component 'cargo'
info: downloading component 'clippy'
info: downloading component 'rust-docs'
info: downloading component 'rust-std'
info: downloading component 'rustc'
info: downloading component 'rustfmt'
info: installing component 'cargo'
info: installing component 'clippy'
info: installing component 'rust-docs'
info: installing component 'rust-std'
info: installing component 'rustc'
info: installing component 'rustfmt'
info: downloading component 'miri'
info: installing component 'miri'
```

你刚刚在我的计算机上安装了什么！？

“好东西”

> **旁白**：工具链版本发生了一些奇怪的事情：
>
> 我们正在安装的工具`miri`与rustc的内部结构紧密配合，因此它仅适用于夜间工具链。
>
> `+nightly-2022-01-21`（实际是`+nightly-2023-02-14`）告诉`rustup`我们想要在该日期使用rust nightly工具链安装`miri`。我给出一个具体的日期是因为有时`miri`会落后，无法为几个夜间版本构建。如果我们尚未安装，`rustup`将自动下载我们使用`+`指定的任何工具链。
>
> 2022-01-21（实际是2023-02-04）只是我知道有`miri`支持的夜间版本，您可以在此[状态页面](https://rust-lang.github.io/rustup-components-history/)上查看。如果您觉得幸运的话，您可以使用`+nightly`。
>
> 每当我们通过`cargo miri`调用`miri`时，我们还将使用此`+`语法来指定我们安装`miri`的工具链。如果您不想每次都指定它，您可以使用[rustup覆盖设置](https://rust-lang.github.io/rustup/overrides.html)。

```txt
> cargo +nightly-2023-02-14 miri test

I will run `"cargo.exe" "install" "xargo"` to install
a recent enough xargo. Proceed? [Y/n]
```

呃，XARGO到底是什么？

“没事的，别担心。”

```txt
> y

    Updating crates.io index
  Installing xargo v0.3.24
...
    Finished release [optimized] target(s) in 10.65s
  Installing C:\Users\ninte\.cargo\bin\xargo-check.exe
  Installing C:\Users\ninte\.cargo\bin\xargo.exe
   Installed package `xargo v0.3.24` (executables `xargo-check.exe`, `xargo.exe`)

I will run `"rustup" "component" "add" "rust-src"` to install 
the `rust-src` component for the selected toolchain. Proceed? [Y/n]
```

呃？？？

“谁不喜欢拥有一份Rust的源代码呢？”

```txt
> y

info: downloading component 'rust-src'
info: installing component 'rust-src'
```

“哦，是的，准备好了，这是精彩的部分。”（`miri`报错了！）

```txt
   Compiling lists v0.1.0 (C:\Users\ninte\dev\tmp\lists)
    Finished test [unoptimized + debuginfo] target(s) in 0.25s
     Running unittests (lists-5cc11d9ee5c3e924.exe)

error: Undefined Behavior: trying to reborrow for Unique at alloc84055, 
       but parent tag <209678> does not have an appropriate item in 
       the borrow stack

   --> \lib\rustlib\src\rust\library\core\src\option.rs:846:18
    |
846 |             Some(x) => Some(f(x)),
    |                  ^ trying to reborrow for Unique at alloc84055, 
    |                    but parent tag <209678> does not have an 
    |                    appropriate item in the borrow stack
    |
    = help: this indicates a potential bug in the program: 
      it performed an invalid operation, but the rules it 
      violated are still experimental
    = help: see https://github.com/rust-lang/unsafe-code-guidelines/blob/master/wip/stacked-borrows.md 
      for further information

    = note: inside `std::option::Option::<std::boxed::Box<fifth::Node<i32>>>::map::<i32, [closure@src\fifth.rs:31:30: 40:10]>` at \lib\rustlib\src\rust\library\core\src\option.rs:846:18

note: inside `fifth::List::<i32>::pop` at src\fifth.rs:31:9
   --> src\fifth.rs:31:9
    |
31  | /         self.head.take().map(|head| {
32  | |             let head = *head;
33  | |             self.head = head.next;
34  | |
...   |
39  | |             head.elem
40  | |         })
    | |__________^
note: inside `fifth::test::basics` at src\fifth.rs:74:20
   --> src\fifth.rs:74:20
    |
74  |         assert_eq!(list.pop(), Some(1));
    |                    ^^^^^^^^^^
note: inside closure at src\fifth.rs:62:5
   --> src\fifth.rs:62:5
    |
61  |       #[test]
    |       ------- in this procedural macro expansion
62  | /     fn basics() {
63  | |         let mut list = List::new();
64  | |
65  | |         // Check empty list behaves right
...   |
96  | |         assert_eq!(list.pop(), None);
97  | |     }
    | |_____^
 ...
error: aborting due to previous error
```

哇哦。这真是个大错误。

“是的，看看那个东西。你喜欢看它。”

谢谢？

“把那瓶雌二醇也拿来，你待会儿会用到的。”

等一下，为什么？

“你马上就要考虑记忆模型了，相信我。”

> **旁白**：神秘人随后变成了一只狐狸，从墙上的一个洞里跑了出来。作者盯着中间的距离看了几分钟，试图理解刚刚发生的一切。

小巷里那只神秘的狐狸不仅对我的性别说对了：`miri`真的是个好东西。

好吧，那么[`miri`](https://github.com/rust-lang/miri)是什么？

（原文）
> An experimental interpreter for Rust's mid-level intermediate representation (MIR). It can run binaries and test suites of cargo projects and detect certain classes of undefined behavior, for example:
>
> - Out-of-bounds memory accesses and use-after-free
> - Invalid use of uninitialized data
> - Violation of intrinsic preconditions (an unreachable_unchecked being reached, calling copy_nonoverlapping with overlapping ranges, ...)
> - Not sufficiently aligned memory accesses and references
> - Violation of some basic type invariants (a bool that is not 0 or 1, for example, or an invalid enum discriminant)
> - Experimental: Violations of the Stacked Borrows rules governing aliasing for reference types
> - Experimental: Data races (but no weak memory effects)
>
> On top of that, Miri will also tell you about memory leaks: when there is memory still allocated at the end of the execution, and that memory is not reachable from a global static, Miri will raise an error.
>
> ...
>
> However, be aware that Miri will not catch all cases of undefined behavior in your program, and cannot run all programs

（翻译）
> Rust中级中间表示 (`MIR`) 的实验性解释器。它可以运行Cargo项目的二进制文件和测试套件，并检测某些类别的未定义行为，例如：
>
> - 越界内存访问和释放后使用
> - 未初始化数据的无效使用
> - 违反内在前提条件（达到一个`unreachable_unchecked`、调用具有重叠范围的`copy_nonoverlapping`等）
> - 内存访问和引用未充分对齐
> - 违反某些基本类型不变量（例如，非0或1的布尔值或无效的枚举判别式）
> - 实验性：违反管理引用类型别名的栈借用规则
> - 实验性：数据竞争（但没有弱内存效应）
>
> 除此之外，Miri还会告诉您有关内存泄漏的信息：当执行结束时仍有内存分配，并且无法从全局静态访问该内存时，Miri将引发错误。
>
> ...
>
> 但是，请注意，`miri`不会捕获程序中所有未定义行为的情况，也无法运行所有程序

太长了，别看了！

它会解释您的程序，并注意到您是否在运行时违反了规则并执行了未定义的行为。这是必要的，因为未定义的行为通常是在运行时发生的事情。如果可以在编译时发现问题，编译器就会将其设为错误！

如果您熟悉`ubsan`和`tsan`等工具：它基本上就是这样，但总体来说更为极端。

----
`miri`现在拿着一把刀挂在教室窗外。一把学习刀。如果我们想让`miri`检查我们的工作，我们可以让他们用以下方式解释我们的测试套件：

```txt
> cargo +nightly-2023-02-14 miri test
```

现在让我们仔细看看他们在我们桌子上刻了什么：

```txt
error: Undefined Behavior: trying to reborrow for Unique at alloc84055, but parent tag <209678> does not have an appropriate item in the borrow stack

   --> \lib\rustlib\src\rust\library\core\src\option.rs:846:18
    |
846 |             Some(x) => Some(f(x)),
    |                  ^ trying to reborrow for Unique at alloc84055, 
    |                    but parent tag <209678> does not have an 
    |                    appropriate item in the borrow stack
    |

    = help: this indicates a potential bug in the program: it 
      performed an invalid operation, but the rules it 
      violated are still experimental
    
    = help: see 
      https://github.com/rust-lang/unsafe-code-guidelines/blob/master/wip/stacked-borrows.md 
      for further information
```

好吧，我知道我们犯了一个错误，但这是一个令人困惑的错误消息。什么是“借用栈”？

我们将在下一节中尝试弄清楚这一点。

## 6.5 栈借用

### 尝试理解栈借用

在上一节中，我们尝试在`miri`下运行不安全的单链队列，它说我们违反了栈借用的规则，并给我们提供了一些文档链接。

通常我会引导大家浏览文档，但我们并不是该文档的目标受众。它更适合编译器开发人员和研究Rust语义的学者。

所以我将只向您介绍“栈借用”的高级概念，然后为您提供遵循规则的简单策略。

> **旁白**：栈借用作为Rust的语义模型仍处于“实验阶段”，因此违反这些规则可能并不意味着您的程序“出错了”。但除非您真的在编译器上工作，否则您应该在`miri`抱怨时修复您的程序。当涉及到未定义行为时，谨慎总比后悔好。

### 动机：指针别名

在讨论我们违反了哪些规则之前，首先了解规则存在的原因会有所帮助。有几个不同的动机问题，但我认为最重要的是指针别名。

当两个指针指向的内存块重叠时，我们称它们为别名。就像“使用别名”的人可以用两个不同的名字来指代一样，重叠的内存块也可以由两个不同的指针来指代。这可能会导致问题。

编译器使用有关指针别名的信息来优化对内存的访问，因此如果它拥有的信息是错误的，那么程序将被错误编译并产生随机垃圾。

> **旁白**：从实际角度来说，别名与内存访问的关系比指针本身更密切，并且只有在其中一个访问发生变化时才真正重要。之所以强调指针，是因为它们便于附加规则。

为了理解为什么指针别名信息很重要，让我们来思考一下《愤怒的小矮人寓言》。

----
一天，**米歇尔**在翻阅书架时，看到了一本不记得的书。他们从书架上拿出这本书，看了看封面。

“哦，是的，我以前读过的《战争与和平》这本书。我很喜欢和平的部分。”

突然，门外传来一阵敲门声。米歇尔把书放回书架，打开了门——是他们的死对头**哈姆斯劳**。正当哈姆斯劳准备对米歇尔明显低劣的代码高尔夫技术进行猛烈抨击时，他们感觉到了一个机会：

“嘿，哈姆斯劳，你读过《战争与和平》吗？”

“嗯，我看过了，你看它就在我的书架上，这显然意味着我读过它。”

哈姆斯劳简直不敢相信。她的脸上从平时自鸣得意的样子变成了一张充满愤怒和决心的铁面具。哈姆斯劳把米歇尔推到一边，快步走到书架前，以一千名女武神的愤怒把书从原处劈开。她把这本古老的书翻过来，一看到封面，她就开始发抖。

米歇尔正准备炫耀他们显然无与伦比的才华，却被哈姆斯劳突然的笑声打断了。

“这不是战争与和平，这是战争与脚！”

哈姆斯劳的脸上流下了泪水。这显然是她一生中最伟大的时刻。

“不！我只是看了看！”

他们从哈姆斯劳手里抢过书，检查了封面。果然，“和平”一词被划掉了，取而代之的是“脚”。

米歇尔惊恐万分。这显然是他们一生中最糟糕的时刻。

他们跪倒在地，茫然地盯着书架。怎么会发生这种事？他们刚才才检查过封面呢！

然后他们看到书架上出现了一点动静。那是一个矮小的男人。一个带着米歇尔所见过的最愤怒的怒容的矮小男人。那个矮小的男人对米歇尔竖了中指，嘴里念叨着“没人会相信你”，然后消失在书堆里。

米歇尔的计划很完美，但他们没有考虑到一个拿着记号笔的愤怒的小矮人和毁灭欲望的可能性。他们以为自己知道书的封面上写的是什么，他们认为没有人能改变它。但可惜的是，他们错了。

哈姆斯劳已经在制作一本纪念她令人难以置信的胜利的杂志——米歇尔在当地网吧的声誉将永远无法恢复。

----
没人想成为米歇尔，但也没人想一直生活在对这个愤怒的小矮人的恐惧中。我们想知道这个愤怒的小矮人什么时候会捉弄我们。当他捉弄我们时，我们会非常小心谨慎地检查所有东西，然后再使用。但是当这个愤怒的小矮人消失时，我们希望能够记住事情。

这就是指针别名的（非常简单的）关键：编译器何时可以假设“记住”（缓存）值而不是一遍又一遍地加载它们是安全的？要知道这一点，编译器需要知道什么时候可能会有愤怒的小矮人背着你改变内存。

> **旁白**：编译器还会使用这些信息来缓存存储，这意味着如果它认为没有人会注意到，它就可以避免将内容提交到内存中。在这种情况下，问题仍然是愤怒的小矮人，但他们只需要读取内存就会成为问题。

### 安全的栈借用

好的，所以我们希望编译器具有良好的指针别名信息，我们可以这样做吗？好吧，Rust似乎就是为此而设计的。可变引用在定义上不是别名，虽然共享引用可以互为别名，但它们不能被改变。完美！发货！

但它比这更复杂。我们可以像这样“重新借用”可变指针：

编辑`src/main.rs`

```rust
#![allow(unused)]
fn main() {
    let mut data = 10;
    let ref1 = &mut data;
    let ref2 = &mut *ref1;

    *ref2 += 2;
    *ref1 += 1;

    println!("{}", data);
}
```

```txt
> cargo run

13
```

编译并运行良好。这是怎么回事？

好吧，我们可以通过交换两个用途来查看发生了什么：

```rust
#![allow(unused)]
fn main() {
    let mut data = 10;
    let ref1 = &mut data;
    let ref2 = &mut *ref1;
    // ORDER SWAPPED!
    *ref1 += 1;
    *ref2 += 2;

    println!("{}", data);
}
```

```txt
> cargo run

error[E0503]: cannot use `*ref1` because it was mutably borrowed
 --> src/main.rs:8:5
  |
5 |     let ref2 = &mut *ref1;
  |                ---------- borrow of `*ref1` occurs here
6 |     
7 |     *ref1 += 1;
  |     ^^^^^^^^^^ use of borrowed `*ref1`
8 |     *ref2 += 2;
  |     ---------- borrow later used here

For more information about this error, try `rustc --explain E0503`.
error: could not compile `playground` due to previous error
```

突然出现编译器错误！

当我们重新借用（`reborrow`）可变指针时，原先的指针不能再使用，直到借用者用完它（不再使用）。

在前面可以有效执行的代码中，有一个不错的使用嵌套。我们重新借用指针，使用新指针一段时间，然后在再次使用旧指针之前停止使用它。在后面无法编译的代码中，不会发生这种情况。因为我们任意地交错使用两个指针。

这就是我们可以重新借用并仍然具有别名信息的方式：我们所有的重新借用都明确嵌套，因此我们可以认为在任何给定时间只有一个“活着”。

嘿，你知道什么是表示干净嵌套事物的好方法吗？栈。借用的栈。 哦，嘿，这就是栈借用！

借用栈顶部的任何东西都是“活动的”，并且知道它实际上是无别名的。当您重新借用一个指针时，新指针将被押入到栈上，成为活动指针。当您使用旧指针时，它会通过弹出借用栈上方的所有内容来恢复活力。此时，指针“知道”它已被重新借用，并且内存可能已被修改，但它再次拥有独占访问权限——无需担心愤怒的小矮人。

因此，访问重新借用的指针实际上总是可以的，因为我们总是可以弹出它上面的所有内容。真正的麻烦是访问已经从借用栈中弹出的指针——那么你就搞砸了。而类似上面一样交错使用就会导致这样的问题。

值得庆幸的是，借用检查器的设计确保了安全的Rust程序遵循这些规则，正如我们在上面的例子中看到的那样，但编译器通常从栈借用的角度“反向”看待这个问题。它不是说使用`ref1`会使`ref2`无效，而是坚持认为`ref2`必须对所有用途都有效，并且`ref1`是因不按顺序使用而搞砸事情的人。

因此“不能使用`*ref1`，因为它是可变借用的”。这是相同的结果（尤其是对于非词汇生命周期），但以一种可能更直观的方式构建。

但是当我们开始使用不安全的指针时，借用检查器无法帮助我们！

### 不安全的栈借用

因此，我们希望以某种方式让不安全指针参与这个堆栈借用系统，即使编译器无法正确跟踪它们。我们还希望系统相当宽松，这样就不会太容易搞砸并导致未定义的行为（`Undefined Behavioury`缩写为`UB`）。

这是一个难题，我不知道如何解决它，但研究栈借用的人想出了一个可行的办法，`miri`试图实现它。

非常高级的概念是，当您将引用（或任何其他安全指针）转换为原始指针时，这基本上就像重新借用（`reborrow`）一样。因此，现在允许原始指针对该内存执行任何操作，并且当重新借用到期时，它就像正常重新借用时一样。

但问题是，重新借用何时到期？好吧，可能一个合适的到期时间是您再次开始使用原始引用时。否则事情就不是一个很好的嵌套的栈。

但是等等，你可以将原始指针转换为引用！而且你可以复制原始指针！如果你转到`&mut -> *mut -> &mut -> *mut`然后访问第一个`*mut`会怎么样？那么栈借用到底是如何工作的？

我真的不知道！这就是事情复杂的原因。事实上，它们更加复杂，因为堆借用试图更加宽容，让更多不安全的代码按照你期望的方式工作。这就是为什么我在`miri`下运行程序以帮助我捕捉错误。

事实上，这种混乱就是为什么`miri`有一个额外的实验性超严格模式：`-Zmiri-tag-raw-pointers`。

为了启用它，我们需要通过`MIRIFLAGS`环境变量传递它，如下所示：

```txt
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri test

# 注意，实际我用的是这个
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2023-02-14 miri test
```

或者像在Windows上这样，你只需要全局设置变量：

```txt
$env:MIRIFLAGS="-Zmiri-tag-raw-pointers"
cargo +nightly-2022-01-21 miri test

# 注意，实际我用的是这个
cargo +nightly-2022-02-14 miri test
```

我们通常会尝试遵循这种非常严格的模式，以便对我们的工作更加自信。从某种意义上说，它也更“简单”，因此实际上更适合乱搞和获得堆栈借用的直觉。

### 管理栈借用

因此，当使用原始指针时，我们将尝试坚持一种简单而直接的启发式方法，并且希望具有较大的误差幅度：

**一旦开始使用原始指针，请尝试仅使用原始指针。**

这使得意外丢失原始指针访问内存的“权限”的可能性尽可能小。

> **旁白**：这在两个方面过于简单：
>
> 1. 安全指针通常断言的属性不仅仅是别名：内存已分配、已对齐、足够大以适合指针指向的类型、指针指向已正确初始化等。因此，当事情处于可疑状态时，疯狂地抛出它们会更加危险。
> 2. 即使您停留在原始指针领域，也不能随意地为任何内存设置别名。指针在概念上与特定的“分配”相关联（可以像堆栈上的局部变量一样细化），并且您不应该从一个分配中获取指针，对其进行偏移，然后访问不同分配中的内存。如果允许这样做，那么到处都会有愤怒的小矮人的威胁。这也是“指针只是整数”是一个有问题的观点的部分原因。

现在，我们仍然希望在界面中实现安全引用，因为我们希望构建一个良好的安全抽象，以便列表的用户不必知道或担心。

所以我们要做的是：

1. 在方法开始时，使用输入引用来获取我们的原始指针
2. 从现在开始尽量只使用不安全指针
3. 如果需要，最后将我们的原始指针转换回安全指针

但是我们类型的字段是私有的，所以我们将把它们完全保留为原始指针。

事实上，我们犯下的一个大错误就是继续使用`Box`！`Box`中有一个特殊的注释，告诉编译器“嘿，这很像`&mut`，因为它唯一地拥有该指针”。这是真的！

但是我们保留到列表末尾的原始指针指向一个`Box`，所以每当我们正常访问`Box`时，我们可能都会使该原始指针的“重新借用”无效！☠

在下一节中，我们将回到我们的真实形式，并用一堆例子撞击我们的脑袋。

## 6.6 测试栈借用

> 上一节中Rust的（简化）内存模型的小结：
>
> - Rust在概念上通过维护“借用栈”来处理重新借用
> - 只有栈顶部的直指针才是“活动的”（具有独占访问权限）
> - 当您访问较低的栈时，它将变为“活动的”，并且其上方的栈内容将被弹出
> - 您不得使用已从借用栈弹出的指针
> - 借用检查器确保安全代码遵循此规则
> - `miri`理论上会在运行时检查原始指针是否遵循此规则

以上就是很多理论和想法——让我们继续讨论这本书的真正核心和灵魂：编写一些糟糕的代码，并让工具向我们发出尖叫声。我们将通过大量示例来尝试了解我们的思维模型是否合理，并尝试直观地了解栈借用。

> **旁白**：在实践中捕捉未定义行为是一件棘手的事情。毕竟，你正在处理编译器实际上认为不会发生的情况。
>
> 如果你很幸运，事情今天“似乎可以正常工作”，但对于更智能的编译器或代码的轻微更改来说，它们将是一颗定时炸弹。如果你真的很幸运，事情就会可靠地崩溃，这样你就可以捕捉到错误并修复它。但如果你运气不好，那么事情就会以奇怪和令人困惑的方式被破坏（即莫名其妙地崩溃）。
>
> `miri`试图通过获取`rustc`对程序最幼稚和最不优化的视图并在解释时跟踪额外状态来解决这个问题。就“消毒剂”而言，这是一种相当确定和强大的方法，但它永远不会完美。你需要你的测试程序真正执行该未定义的行为（`UB`），对于足够大的程序，很容易引入各种非确定性（例如`HashMap`默认使用`RNG`！）。
>
> 我们永远不能将`miri`赞同我们程序的执行视为绝对肯定不存在未定义的行为（`UB`）的说法。`miri`也有可能认为某事是未定义的行为（`UB`），但实际上并非如此。但如果我们对事物的运作方式有一个心理模型，并且`miri`似乎同意我们的观点，那么这是一个很好的迹象，表明我们走在正确的轨道上。

### 基本借用

在前面的部分中，我们看到`rustc`的借用检查器不喜欢这样的代码：

```rust
#![allow(unused)]
fn main() {
    let mut data = 10;
    let ref1 = &mut data;
    let ref2 = &mut *ref1;
    // ORDER SWAPPED!
    *ref1 += 1;
    *ref2 += 2;

    println!("{}", data);
}
```

让我们看看当我们用`*mut`替换`ref2`时会发生什么：

```rust
#![allow(unused)]
fn main() {
    unsafe {
        let mut data = 10;
        let ref1 = &mut data;
        let ptr2 = ref1 as *mut _;
        // ORDER SWAPPED!
        *ref1 += 1;
        *ptr2 += 2;
    
        println!("{}", data);
    }
}
```

```txt
> cargo run
   Compiling miri-sandbox v0.1.0
    Finished dev [unoptimized + debuginfo] target(s) in 0.71s
     Running `target\debug\miri-sandbox.exe`
13
```

`rustc`似乎对此非常满意：没有警告，程序产生了我们预期的结果！现在让我们看看`miri`（在严格模式下）对此的看法：

```txt
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2023-02-14 miri run

    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
     Running cargo-miri.exe target\miri

error: Undefined Behavior: no item granting read access 
to tag <untagged> at alloc748 found in borrow stack.

 --> src\main.rs:9:9
  |
9 |         *ptr2 += 2;
  |         ^^^^^^^^^^ no item granting read access to tag <untagged> 
  |                    at alloc748 found in borrow stack.
  |
  = help: this indicates a potential bug in the program: 
    it performed an invalid operation, but the rules it 
    violated are still experimental
```

太棒了！我们对事物运作方式的直观模型是正确的：虽然编译器无法帮我们捕获问题，但`miri`可以。

让我们尝试一些更复杂的东西，即我们之前提到的`&mut -> *mut -> &mut -> *mut`的情况：

```rust
#![allow(unused)]
fn main() {
    unsafe {
        let mut data = 10;
        let ref1 = &mut data;
        let ptr2 = ref1 as *mut _;
        let ref3 = &mut *ptr2;
        let ptr4 = ref3 as *mut _;

        // Access the first raw pointer first
        *ptr2 += 2;

        // Then access things in "borrow stack" order
        *ptr4 += 4;
        *ref3 += 3;
        *ptr2 += 2;
        *ref1 += 1;

        println!("{}", data);
    }
}
```

```txt
> cargo run
22

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2023-02-14 miri run

error: Undefined Behavior: no item granting read access 
to tag <1621> at alloc748 found in borrow stack.

  --> src\main.rs:13:5
   |
13 |     *ptr4 += 4;
   |     ^^^^^^^^^^ no item granting read access to tag <1621> 
   |                at alloc748 found in borrow stack.
   |
```

哇，是的！在严格模式下，`miri`可以“区分”两个原始指针，并使用第二个指针使第一个指针无效。让我们看看当我们删除第一个弄乱一切的使用时，一切是否正常：

```rust
#![allow(unused)]
fn main() {
    unsafe {
        let mut data = 10;
        let ref1 = &mut data;
        let ptr2 = ref1 as *mut _;
        let ref3 = &mut *ptr2;
        let ptr4 = ref3 as *mut _;

        // Access things in "borrow stack" order
        *ptr4 += 4;
        *ref3 += 3;
        *ptr2 += 2;
        *ref1 += 1;

        println!("{}", data);
    }
}
```

```txt
> cargo run
20

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2023-02-14 miri run
20
```

很好。

是的，我很确定现在我们都可以获得编程语言内存模型设计和实现方面的博士学位。谁还需要编译器，这些东西很简单。

> **旁白**：事实并非如此，（编译器当然很复杂，）但我仍然为你感到骄傲。（要完成我们的工作，这些就差不多了。）

### 测试数组

让我们来处理一些数组和指针偏移（添加和减法）。这应该可以，对吧？

```rust
#![allow(unused)]
fn main() {
    unsafe {
        let mut data = [0; 10];
        let ref1_at_0 = &mut data[0];           // Reference to 0th element
        let ptr2_at_0 = ref1_at_0 as *mut i32;  // Ptr to 0th element
        let ptr3_at_1 = ptr2_at_0.add(1);       // Ptr to 1st element
    
        *ptr3_at_1 += 3;
        *ptr2_at_0 += 2;
        *ref1_at_0 += 1;
    
        // Should be [3, 3, 0, ...]
        println!("{:?}", &data[..]);
    }
}
```

```txt
> cargo run
[3, 3, 0, 0, 0, 0, 0, 0, 0, 0]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2023-02-14 miri run

error: Undefined Behavior: no item granting read access 
to tag <1619> at alloc748+0x4 found in borrow stack.
 --> src\main.rs:8:5
  |
8 |     *ptr3_at_1 += 3;
  |     ^^^^^^^^^^^^^^^ no item granting read access to tag <1619>
  |                     at alloc748+0x4 found in borrow stack.
```

撕毁研究生申请

发生了什么？我们完全正常地使用了借用栈！当我们转到`ptr -> ptr`时是否发生了一些奇怪的事情？如果我们只是复制指针，让它们都转到同一个位置会怎么样：

```rust
#![allow(unused)]
fn main() {
    unsafe {
        let mut data = [0; 10];
        let ref1_at_0 = &mut data[0];           // Reference to 0th element
        let ptr2_at_0 = ref1_at_0 as *mut i32;  // Ptr to 0th element
        let ptr3_at_0 = ptr2_at_0;              // Ptr to 0th element

        *ptr3_at_0 += 3;
        *ptr2_at_0 += 2;
        *ref1_at_0 += 1;

        // Should be [6, 0, 0, ...]
        println!("{:?}", &data[..]);
    }
}
```

```txt
> cargo run
[6, 0, 0, 0, 0, 0, 0, 0, 0, 0]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2023-02-14 miri run
[6, 0, 0, 0, 0, 0, 0, 0, 0, 0]
```

不，这样很好。也许我们运气好，让我们把指针弄得一团糟：

```rust
#![allow(unused)]
fn main() {
    unsafe {
        let mut data = [0; 10];
        let ref1_at_0 = &mut data[0];            // Reference to 0th element
        let ptr2_at_0 = ref1_at_0 as *mut i32;   // Ptr to 0th element
        let ptr3_at_0 = ptr2_at_0;               // Ptr to 0th element
        let ptr4_at_0 = ptr2_at_0.add(0);        // Ptr to 0th element
        let ptr5_at_0 = ptr3_at_0.add(1).sub(1); // Ptr to 0th element

        // An absolute jumbled hash of ptr usages
        *ptr3_at_0 += 3;
        *ptr2_at_0 += 2;
        *ptr4_at_0 += 4;
        *ptr5_at_0 += 5;
        *ptr3_at_0 += 3;
        *ptr2_at_0 += 2;
        *ref1_at_0 += 1;

        // Should be [20, 0, 0, ...]
        println!("{:?}", &data[..]);
    }
}
```

```txt
> cargo run
[20, 0, 0, 0, 0, 0, 0, 0, 0, 0]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2023-02-14 miri run
[20, 0, 0, 0, 0, 0, 0, 0, 0, 0]
```

不！对于从其他原始指针派生的原始指针，`miri`实际上更加宽容。它们都共享相同的“借用”（或者`miri`称之为标签）。

一旦您开始使用原始指针，它们就可以自由地分裂成自己的愤怒的小矮人并自相残杀。这是可以的，因为编译器理解这一点，并且不会像优化引用那样优化读取和写入。

> **旁白**：如果代码足够简单，编译器可以跟踪所有派生的指针，并尽可能地进行优化，但它会比用于引用的推理要脆弱得多。

那么真正的问题是什么？

尽管数据是一个“分配”（局部变量），但`ref1_at_0`仅借用了第一个元素。Rust允许将借用分解，以便它们仅适用于分配的特定部分！让我们尝试一下：

```rust
#![allow(unused)]
fn main() {
    unsafe {
        let mut data = [0; 10];
        let ref1_at_0 = &mut data[0];           // Reference to 0th element
        let ref2_at_1 = &mut data[1];           // Reference to 1th element
        let ptr3_at_0 = ref1_at_0 as *mut i32;  // Ptr to 0th element
        let ptr4_at_1 = ref2_at_1 as *mut i32;   // Ptr to 1th element

        *ptr4_at_1 += 4;
        *ptr3_at_0 += 3;
        *ref2_at_1 += 2;
        *ref1_at_0 += 1;

        // Should be [4, 6, 0, ...]
        println!("{:?}", &data[..]);
    }
}
```

```txt
> cargo run

error[E0499]: cannot borrow `data[_]` as mutable more than once at a time
 --> src\main.rs:5:21
  |
4 |     let ref1_at_0 = &mut data[0];           // Reference to 0th element
  |                     ------------ first mutable borrow occurs here
5 |     let ref2_at_1 = &mut data[1];           // Reference to 1th element
  |                     ^^^^^^^^^^^^ second mutable borrow occurs here
6 |     let ptr3_at_0 = ref1_at_0 as *mut i32;  // Ptr to 0th element
  |                     --------- first borrow later used here
  |
  = help: consider using `.split_at_mut(position)` or similar method 
    to obtain two mutable non-overlapping sub-slices
```

哎呀！不能这么做，Rust不会跟踪数组索引来证明这些借用是不相交的，但它为我们提供了切片（`slice`）上的方法`split_at_mut`，以一种可以安全假设的方式将完整的切片分成多个部分：

```rust
#![allow(unused)]
fn main() {
    unsafe {
        let mut data = [0; 10];

        let slice1 = &mut data[..];
        let (slice2_at_0, slice3_at_1) = slice1.split_at_mut(1); 

        let ref4_at_0 = &mut slice2_at_0[0];    // Reference to 0th element
        let ref5_at_1 = &mut slice3_at_1[0];    // Reference to 1th element
        let ptr6_at_0 = ref4_at_0 as *mut i32;  // Ptr to 0th element
        let ptr7_at_1 = ref5_at_1 as *mut i32;  // Ptr to 1th element

        *ptr7_at_1 += 7;
        *ptr6_at_0 += 6;
        *ref5_at_1 += 5;
        *ref4_at_0 += 4;

        // Should be [10, 12, 0, ...]
        println!("{:?}", &data[..]);
    }
}
```

```txt
> cargo run
[10, 12, 0, 0, 0, 0, 0, 0, 0, 0]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2023-02-14 miri run
[10, 12, 0, 0, 0, 0, 0, 0, 0, 0]
```

嘿，这有效！切片正确地告诉编译器和`miri`，“嘿，我正在借用我范围内的所有内存”，所以他们知道所有元素都可以变异。

还请注意，允许使用象`split_at_mut`的操作告诉我们借用可以不那么像栈，而更像树，因为我们可以将一个大借用分解为一堆不相交的小借用，一切仍然有效。

（我认为在实际的栈借用模型中，一切仍然是栈，因为栈在概念上跟踪程序每个字节的权限......？）

如果我们直接将切片变成指针会怎样？该指针是否可以访问整个切片？

```rust
#![allow(unused)]
fn main() {
    unsafe {
        let mut data = [0; 10];

        let slice1_all = &mut data[..];         // Slice for the entire array
        let ptr2_all = slice1_all.as_mut_ptr(); // Pointer for the entire array

        let ptr3_at_0 = ptr2_all;               // Pointer to 0th elem (the same)
        let ptr4_at_1 = ptr2_all.add(1);        // Pointer to 1th elem
        let ref5_at_0 = &mut *ptr3_at_0;        // Reference to 0th elem
        let ref6_at_1 = &mut *ptr4_at_1;        // Reference to 1th elem

        *ref6_at_1 += 6;
        *ref5_at_0 += 5;
        *ptr4_at_1 += 4;
        *ptr3_at_0 += 3;

        // Just for fun, modify all the elements in a loop
        // (Could use any of the raw pointers for this, they share a borrow!)
        for idx in 0..10 {
            *ptr2_all.add(idx) += idx;
        }

        // Safe version of this same code for fun
        for (idx, elem_ref) in slice1_all.iter_mut().enumerate() {
            *elem_ref += idx; 
        }

        // Should be [8, 12, 4, 6, 8, 10, 12, 14, 16, 18]
        println!("{:?}", &data[..]);
    }
}
```

```txt
> cargo run
[8, 12, 4, 6, 8, 10, 12, 14, 16, 18]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2023-02-14 miri run
[8, 12, 4, 6, 8, 10, 12, 14, 16, 18]
```

太棒了！指针不仅仅是整数：它们具有与之关联的内存范围，而使用Rust我们可以缩小该范围！就是说我们可以利用它访问数组中的一个或多个元素。

### 测试共享引用

在所有这些示例中，我都非常小心地只使用可变引用并执行读取-修改-写入操作（`+=`），以使事情尽可能简单。

但Rust具有只读且可以自由复制的共享引用，它们应该如何工作？好吧，我们已经看到原始指针可以自由复制，我们可以通过说它们“共享”单个借用来处理这个问题。也许我们以同样的方式考虑共享引用？

让我们用一个读取值的函数来测试一下（`println!`）对于自动引用/解引用来说可能有点神奇，所以我将它包装在一个函数中以确保我们测试的是我们想要的）：

```rust
#![allow(unused)]
fn main() {
    fn opaque_read(val: &i32) {
        println!("{}", val);
    }

    unsafe {
        let mut data = 10;
        let mref1 = &mut data;
        let sref2 = &mref1;
        let sref3 = sref2;
        let sref4 = &*sref2;

        // Random hash of shared reference reads
        opaque_read(sref3);
        opaque_read(sref2);
        opaque_read(sref4);
        opaque_read(sref2);
        opaque_read(sref3);

        *mref1 += 1;

        opaque_read(&data);
    }
}
```

```txt
> cargo run

warning: unnecessary `unsafe` block
 --> src\main.rs:6:1
  |
6 | unsafe {
  | ^^^^^^ unnecessary `unsafe` block
  |
  = note: `#[warn(unused_unsafe)]` on by default

warning: `miri-sandbox` (bin "miri-sandbox") generated 1 warning

10
10
10
10
10
11
```

哦，是的，我们忘了对原始指针做任何事情，但至少我们可以看到，所有共享引用都可以互换使用。现在让我们混合一些原始指针：

```rust
#![allow(unused)]
fn main() {
    fn opaque_read(val: &i32) {
        println!("{}", val);
    }

    unsafe {
        let mut data = 10;
        let mref1 = &mut data;
        let ptr2 = mref1 as *mut i32;
        let sref3 = &*mref1;
        let ptr4 = sref3 as *mut i32;

        *ptr4 += 4;
        opaque_read(sref3);
        *ptr2 += 2;
        *mref1 += 1;

            opaque_read(&data);
    }
    }
}
```

```txt
> cargo run

error[E0606]: casting `&&mut i32` as `*mut i32` is invalid
  --> src\main.rs:11:16
   |
11 |     let ptr4 = sref3 as *mut i32;
   |                ^^^^^^^^^^^^^^^^^
```

哦，糟糕，我们实际上是在用`& &mut`而不是`&`乱搞！Rust 非常擅长在无关紧要的时候掩盖这一点。让我们把代码修改为```let sref3 = &*mref1;```正确地重新借用它：

```txt
> cargo run

error[E0606]: casting `&i32` as `*mut i32` is invalid
  --> src\main.rs:11:16
   |
11 |     let ptr4 = sref3 as *mut i32;
   |                ^^^^^^^^^^^^^^^^^
```

不，Rust仍然不喜欢这样！您只能将共享引用转换为只能读取的`*const`。但是如果我们只是……这样做……会怎么样？

```rust
    let ptr4 = sref3 as *const i32 as *mut i32;
```

```txt
> cargo run

14
17
```

> 上面是作者使用的老版本编译器（v1.60？）的结果。
>
> 注意，我的新版本`rustc 1.77.1 (7cf61ebde 2024-03-27)`，还是不能通过编译，错误如下，和作者的版本使用`miri`报错相同
>
> ```txt
> error: assigning to `&T` is undefined behavior, consider using an `UnsafeCell`
>  --> src/main.rs:14:9
>    |
> 12 |         let ptr4 = sref3 as *const i32 as *mut i32;
>    |                    ------------------------------- casting happend here
> 13 |
> 14 |         *ptr4 += 4;
>    |         ^^^^^^^^^^
>    |
>    = note: for more information, visit <https://doc.rust-lang.org/book/ch15-05-interior-mutability.html>
>    = note: `#[deny(invalid_reference_casting)]` on by default
> ```

什么。好的，当然可以吗？Rust的转换系统很棒。`*const`几乎是一种非常无用的类型，它只存在于描述C API并模糊地建议正确的用法（确实如此）。`miri`怎么想？

```txt
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2023-02-14 miri run

error: Undefined Behavior: no item granting write access to 
tag <1621> at alloc742 found in borrow stack.
  --> src\main.rs:13:5
   |
13 |     *ptr4 += 4;
   |     ^^^^^^^^^^ no item granting write access to tag <1621>
   |                at alloc742 found in borrow stack.
```

唉，虽然我们可以通过双重转换来避免编译器的抱怨，但实际上并没有允许此操作。当我们获取共享引用时，我们承诺不修改该值。（当然，新版本编译器可以识别这类问题了，它建议我们使用`UnsafeCell`处理。）

这很重要，因为这意味着当共享借用从借用栈中弹出时，其下方的可变指针可以假设内存没有改变。可能有一些愤怒的小矮人读取内存（因此必须提交写入），但他们无法修改它，并且可变指针可以假设他们写入的最后一个值仍然在那里！

**一旦共享引用在借用栈上，推送到它上面的所有内容都只有读取权限。**

但是我们可以这样做：

```rust
#![allow(unused)]
fn main() {
    fn opaque_read(val: &i32) {
        println!("{}", val);
    }

    unsafe {
        let mut data = 10;
        let mref1 = &mut data;
        let ptr2 = mref1 as *mut i32;
        let sref3 = &*mref1;
        let ptr4 = sref3 as *const i32 as *mut i32;

        opaque_read(&*ptr4);
        opaque_read(sref3);
        *ptr2 += 2;
        *mref1 += 1;

        opaque_read(&data);
    }
}
```

请注意，只要我们实际上只读取它（指针4），创建可变的原始指针（指针4）仍然是“可以的”！（虽然它只能用于读取。）

```txt
> cargo run
10
10
13

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2023-02-14 miri run
10
10
13
```

为了确保万无一失，让我们检查一下共享引用是否像正常一样从借用栈中弹出：

```rust
#![allow(unused)]
fn main() {
    fn opaque_read(val: &i32) {
        println!("{}", val);
    }

    unsafe {
        let mut data = 10;
        let mref1 = &mut data;
        let ptr2 = mref1 as *mut i32;
        let sref3 = &*mref1;

        *ptr2 += 2;
        opaque_read(sref3); // Read in the wrong order?
        *mref1 += 1;

        opaque_read(&data);
    }
}
```

```txt
> cargo run
12
13

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2023-02-14 miri run

error: Undefined Behavior: trying to reborrow for SharedReadOnly 
at alloc742, but parent tag <1620> does not have an appropriate 
item in the borrow stack

  --> src\main.rs:13:17
   |
13 |     opaque_read(sref3); // Read in the wrong order?
   |                 ^^^^^ trying to reborrow for SharedReadOnly 
   |                       at alloc742, but parent tag <1620> 
   |                       does not have an appropriate item 
   |                       in the borrow stack
   |
```

嘿，我们甚至收到了一条关于`SharedReadOnly`的略微不同的错误消息，而不是某个特定标签。这很有意义：一旦有任何共享引用，基本上其他一切都只是一大堆`SharedReadOnly`杂烩，因此无需区分它们！

### 测试内部可变性

还记得书中那个非常糟糕的章节吗？对，就是第五章！我们尝试使用`RefCell`和`Rc`创建一个链表，当尝试编写这个该死的链表时，一切都比平时更糟糕。

我们一直坚持共享引用不能用于改变，但那一章是关于如何通过具有内部可变性的共享引用执行改变。让我们尝试简单易懂的[`std::cell::Cell`](https://doc.rust-lang.org/std/cell/struct.Cell.html)类型，它只对可以`Copy`的类型有效：

```rust
#![allow(unused)]
fn main() {
    use std::cell::Cell;

    unsafe {
        let mut data = Cell::new(10);
        let mref1 = &mut data;
        let ptr2 = mref1 as *mut Cell<i32>;
        let sref3 = &*mref1;

        sref3.set(sref3.get() + 3);
        (*ptr2).set((*ptr2).get() + 2);
        mref1.set(mref1.get() + 1);

        println!("{}", data.get());
    }
}
```

啊，真是一团糟。看到`miri`吐在上面报一大堆错误一定很可爱。

```txt
> cargo run
16

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2023-02-14 miri run
16
```

等一下，真的吗？没报错？为什么？怎么做到的？`Cell`到底是什么？看看它的定义

砸碎标准库上的挂锁

```rust
pub struct Cell<T: ?Sized> {
    value: UnsafeCell<T>,
}
```

`UnsafeCell`到底是什么？

砸碎另一把挂锁只是为了向标准库表明我们是认真的

```rust
#[lang = "unsafe_cell"]
#[repr(transparent)]
#[repr(no_niche)]
pub struct UnsafeCell<T: ?Sized> {
    value: T,
}
```

哦，这是巫师的魔法。好吧。我猜。`#[lang = "unsafe_cell"]`实际上是在说`UnsafeCell`就是`UnsafeCell`。让我们停止破坏锁，并检查[`std::cell::UnsafeCell`](https://doc.rust-lang.org/std/cell/struct.UnsafeCell.html)的实际文档。

（原文）
> The core primitive for interior mutability in Rust.
>
> If you have a reference `&T`, then normally in Rust the compiler performs optimizations based on the knowledge that &T points to immutable data. Mutating that data, for example through an alias or by transmuting an `&T` into an `&mut T`, is considered undefined behavior. `UnsafeCell<T>` opts-out of the immutability guarantee for `&T`: a shared reference `&UnsafeCell<T>` may point to data that is being mutated. This is called “interior mutability”.

（翻译）
> Rust中内部可变性的核心原语。
>
> 如果您有一个引用`&T`，那么在Rust中，编译器通常会根据`&T`指向不可变数据的知识执行优化。改变该数据（例如通过别名或将`&T`转换为`&mut T`）被视为未定义的行为。`UnsafeCell<T>`选择退出`&T`的不变性保证：共享引用`&UnsafeCell<T>`可能指向正在改变的数据。这称为“内部可变性”。

哦，这真的只是巫师的魔法。

`UnsafeCell`基本上告诉编译器“嘿，听着，我们要对这个内存进行一些愚蠢的操作，不要对它做出任何常见的别名假设”。就像挂上一个巨大的“小心：愤怒的小矮人正在过马路”标志。

让我们看看添加`UnsafeCell`如何让`miri`感到高兴：

```rust
#![allow(unused)]
fn main() {
    use std::cell::UnsafeCell;

    fn opaque_read(val: &i32) {
        println!("{}", val);
    }

    unsafe {
        let mut data = UnsafeCell::new(10);
        let mref1 = data.get_mut();      // Get a mutable ref to the contents
        let ptr2 = mref1 as *mut i32;
        let sref3 = &*ptr2;

        *ptr2 += 2;
        opaque_read(sref3);
        *mref1 += 1;

        println!("{}", *data.get());
    }
}
```

```txt
> cargo run
12
13

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2023-02-14 miri run

error: Undefined Behavior: trying to reborrow for SharedReadOnly
at alloc748, but parent tag <1629> does not have an appropriate
item in the borrow stack

  --> src\main.rs:15:17
   |
15 |     opaque_read(sref3);
   |                 ^^^^^ trying to reborrow for SharedReadOnly 
   |                       at alloc748, but parent tag <1629> does
   |                       not have an appropriate item in the
   |                       borrow stack
   |
```

等等，什么？我们说出了神奇的话语！我要用所有这些联邦政府批准的仪式增强山羊血做什么？

好吧，我们确实这么做了，但是我们完全放弃了这个咒语，使用`get_mut`窥视`UnsafeCell`内部并对其进行适当的`&mut i32`！

想想看：如果编译器必须假设`&mut i32`可以查看`UnsafeCell`内部，那么它就永远无法对别名做出任何假设！一切都可能充满愤怒的小矮人。

所以我们需要做的是将`UnsafeCell`保留在我们的指针类型中，以便编译器了解我们在做什么。

```rust

#![allow(unused)]
fn main() {
    use std::cell::UnsafeCell;

    fn opaque_read(val: &i32) {
        println!("{}", val);
    }

    unsafe {
        let mut data = UnsafeCell::new(10);
        let mref1 = &mut data;              // Mutable ref to the *outside*
        let ptr2 = mref1.get();             // Get a raw pointer to the insides
        let sref3 = &*mref1;                // Get a shared ref to the *outside*

        *ptr2 += 2;                         // Mutate with the raw pointer
        opaque_read(&*sref3.get());         // Read from the shared ref
        *sref3.get() += 3;                  // Write through the shared ref
        *mref1.get() += 1;                  // Mutate with the mutable ref

        println!("{}", *data.get());
    }
}
```

```txt
> cargo run
12
16

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2023-02-14 miri run
12
16
```

它有效！我终于不用流这么多血了。

实际上，嘿，等等。我们对这里的顺序还是有点搞错了。

我们先创建`ptr2`，然后从可变指针创建`sref3`。然后我们在共享指针之前使用了原始指针。这一切似乎……都是错的。（可为什么`miri`没有报错呢？）

实际上，等等，我们也对`Cell`示例这样做了。嗯。

原因可能是以下两个结论之一：

- `miri`并不完美，这实际上仍然是未定义的行为（`UB`）。
- 我们的简化模型实际上过于简单。

我更愿意相信第二个，但为了安全起见，让我们在我们的简化栈借用模型中制作一个绝对严谨的版本：

```rust
#![allow(unused)]
fn main() {
    use std::cell::UnsafeCell;

    fn opaque_read(val: &i32) {
        println!("{}", val);
    }

    unsafe {
        let mut data = UnsafeCell::new(10);
        let mref1 = &mut data;
        // These two are swapped so the borrows are *definitely* totally stacked
        let sref2 = &*mref1;
        // Derive the ptr from the shared ref to be super safe!
        let ptr3 = sref2.get();             

        *ptr3 += 3;
        opaque_read(&*sref2.get());
        *sref2.get() += 2;
        *mref1.get() += 1;

        println!("{}", *data.get());
    }
}
```

```txt
> cargo run
13
16

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2023-02-14 miri run
13
16
```

（然后，我们任意调整后面那四行的顺序，程序都是正常的，`miri`都不会报错，只是控制台的输出会有所不同。）

现在，我们的第一个实现可能是正确的原因之一是，如果你认真思考，**`&UnsafeCell<T>`就别名而言与`*mut T`真的没什么不同。你可以无限复制它并通过它进行修改！**

所以在某种意义上，我们只是创建了两个原始指针并像平常一样交替使用它们。两者都是从可变引用派生出来的，这有点可疑，所以也许第二个的创建仍然应该将第一个从借用堆栈中弹出，但这并不是真正必要的，因为我们实际上并没有访问可变引用的内容，只是复制了它的地址。

像`let sref2 = &*mref1`这样的代码行很棘手。从语法上看，我们好像是在解引用它，但解引用本身实际上并不是这样？考虑`&my_tuple.0`：您实际上并没有对`my_tuple`或`.0`做任何事情，您只是使用它们来引用内存中的位置，并在其前面放置`&`，表示“不要加载它，只需写下它的地址”。

`&*`是同一件事：`*`只是说“嘿，让我们讨论一下这个指针指向的位置”，而`&`只是说“现在写下那个地址”。这当然与原始指针的值相同。但指针的类型已经改变，因为，呃，类型！

也就是说，如果您执行`&**`，那么您实际上是在用第一个`*`加载一个值！（即等同于`*`，但是在上面的代码中如果你省略`&*`，会导致所有权的转移问题。）`*`很奇怪！

> **旁白**：没人关心你是否知道“左值”这个词，乔纳森。在Rust中我们称它们为位置，这完全不同，而且更酷吗？

### 测试`Box`

嘿，还记得我们为什么开始写这么长的篇幅吗？你不记得了吗？很奇怪。

那是因为我们混合了`Box`和原始指针。`Box`有点像`&mut`，因为它声称对其指向的内存拥有唯一所有权。让我们测试一下这个说法：

```rust
#![allow(unused)]
fn main() {
    unsafe {
        let mut data = Box::new(10);
        let ptr1 = (&mut *data) as *mut i32;

        *data += 10;
        *ptr1 += 1;

        // Should be 21
        println!("{}", data);
    }
}
```

```txt
> cargo run
21

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2023-02-14 miri run

error: Undefined Behavior: no item granting read access 
       to tag <1707> at alloc763 found in borrow stack.

 --> src\main.rs:7:5
  |
7 |     *ptr1 += 1;
  |     ^^^^^^^^^^ no item granting read access to tag <1707> 
  |                at alloc763 found in borrow stack.
  |
```

是的，`miri`讨厌这样。让我们检查一下是否按照正确的顺序执行操作：

```rust
#![allow(unused)]
fn main() {
    unsafe {
        let mut data = Box::new(10);
        let ptr1 = (&mut *data) as *mut i32;

        *ptr1 += 1;
        *data += 10;

        // Should be 21
        println!("{}", data);
    }
}
```

```txt
> cargo run
21

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2023-02-14 miri run
21
```

是的！（上面关于`Box`的说法是正确的。）

好了，就这些了，我们终于结束了关于栈借用的讨论和思考！

...等等，我们如何用`Box`解决这个问题？比如，当然我们可以编写这样的玩具程序，但我们需要将`Box`存储在某个地方，并可能长时间保留我们的原始指针。肯定会混淆和失效吧？

好问题！要回答这个问题，我们最终将回到我们真正的使命：编写一些该死的链表。

等等，我需要再次编写链表吗？让我们不要急躁。要理智。稍等一下，我相信还有其他一些有趣的问题需要我讨论——

## 6.7 布局和基本操作2：获取原始指针

> 前三个部分的小结：随机混合安全指针（如`&`、`&mut`和`Box`）与不安全指针（如`*mut`和`*const`）会导致未定义行为，因为安全指针引入了我们在使用原始指针时不会遵守的额外约束。

天哪，我需要再次编写链表。很好。很好。很好。我们很好。

我们将很快完成此部分的大部分内容，因为我们已经在第一次尝试中讨论了设计，并且我们所做的一切基本上都是正确的，除了我们如何混合使用安全和不安全指针。

### 布局

因此，在新的布局中，我们只会使用原始指针，一切都会很完美，我们永远不会再犯错误。

这是我们旧的不好的布局，它使用了`Box`：

```rust
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>, // INNOCENT AND KIND
}

type Link<T> = Option<Box<Node<T>>>; // THE REAL EVIL

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

这是我们的新布局，去掉了`Box`，只使用原始指针：

```rust
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>,
}

type Link<T> = *mut Node<T>; // MUCH BETTER

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

请记住：当我们使用原始指针时，`Option`并不那么好用，所以我们不再使用它了。在后面的部分中，我们将介绍`NonNull`类型，但现在不用担心。

### 基本操作

`List::new`基本相同。

```rust
use ptr;

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: ptr::null_mut(), tail: ptr::null_mut() }
    }
}
```

`push`方法基本上就是

```rust
    pub fn push(&mut self, elem: T) {
        let mut new_tail = Box::new(
```

等一下，我们不再使用`Box`。没有`Box`我们该如何分配内存？

好吧，我们可以使用`std::alloc::alloc`，但这属于杀鸡用牛刀。它可以完成工作，但有点过度和笨重。

我们想要有个`Box`，但没有。一个完全疯狂但可能可行的选择是做这样的事情：

```rust
struct Node<T> {
    elem: T,
    real_next: Option<Box<Node<T>>>,
    next: *mut Node<T>,
}
```

这个想法是，我们创建`Box`并将它们存储在我们的节点中，然后我们将原始指针放入其中，并且只使用该原始指针，直到我们完成`Node`并想要销毁它为止。然后我们可以将`Box`从`real_next`中取出并丢弃。我认为这符合我们非常简化的栈借用模型？

如果你想尝试这样做，那就“玩得开心”，但看起来很糟糕，对吧？这不是关于`Rc`和`RefCell`的章节，我们不会再玩这个游戏了。我们只会制作简单干净的东西。

因此，我们将使用非常好的[`Box::into_raw`](https://doc.rust-lang.org/std/boxed/struct.Box.html#method.into_raw)函数：

（原文）
>
> ```rust
>     pub fn into_raw(b: Box<T>) -> *mut T
> ```
>
> Consumes the Box, returning a wrapped raw pointer.
>
> The pointer will be properly aligned and non-null.
>
> After calling this function, the caller is responsible for the memory previously managed by the Box. In particular, the caller should properly destroy T and release the memory, taking into account the memory layout used by Box. The easiest way to do this is to convert the raw pointer back into a Box with the Box::from_raw function, allowing the Box destructor to perform the cleanup.
>
> Note: this is an associated function, which means that you have to call it as Box::into_raw(b) instead of b.into_raw(). This is so that there is no conflict with a method on the inner type.
>
> **Examples**
>
> Converting the raw pointer back into a Box with Box::from_raw for automatic cleanup:
>
> ```rust
> let x = Box::new(String::from("Hello"));
> let ptr = Box::into_raw(x);
> let x = unsafe { Box::from_raw(ptr) };
> ```

（翻译）
>
> ```rust
>     pub fn into_raw(b: Box<T>) -> *mut T
> ```
>
> 消费`Box`，返回包装的原始指针。
>
> 指针将正确对齐且非空。
>
> 调用此函数后，调用者负责先前由`Box`管理的内存。特别是，调用者应正确销毁`T`并释放内存，同时考虑`Box`使用的内存布局。最简单的方法是使用`Box::from_raw`函数将原始指针转换回`Box`，允许`Box`析构函数执行清理。
>
> 注意：这是一个关联函数，这意味着您必须将其调用为`Box::into_raw(b)`而不是`b.into_raw()`。这样就不会与内部类型的方法发生冲突。
>
> **示例**
>
> 使用`Box::from_raw`将原始指针转换回`Box`以进行自动清理：
>
> ```rust
> let x = Box::new(String::from("Hello"));
> let ptr = Box::into_raw(x);
> let x = unsafe { Box::from_raw(ptr) };
> ```

很好，这看起来确实是为我们的用例设计的。它还符合我们试图遵循的规则：从安全的东西开始，变成原始指针，然后只在最后转换回安全的东西（当我们想要丢弃它时）。

这基本上就像做奇怪的`real_next`事情一样，但不必费力地存储`Box`，因为它与原始指针完全相同。

此外，既然现在我们是到处都在使用原始指针，我们就不必担心保持这些不安全的（`unsafe`）程序块足够短：现在一切都不安全了。（它一直都是，但有时欺骗自己也不错。）

让我们试试用它实现`push`

```rust
    pub fn push(&mut self, elem: T) {
        unsafe {
            // Immediately convert the Box into a raw pointer
            let new_tail = Box::into_raw(Box::new(Node {
                elem: elem,
                next: ptr::null_mut(),
            }));

            if !self.tail.is_null() {
                (*self.tail).next = new_tail;
            } else {
                self.head = new_tail;
            }

            self.tail = new_tail;
        }
    }
```

嘿，既然我们坚持使用原始指针，代码现在看起来确实干净多了！

接下来是`pop`，这也与我们之前的做法非常相似，尽管我们必须记住使用`Box::from_raw`来清理分配：

```rust
    pub fn pop(&mut self) -> Option<T> {
        unsafe {
            if self.head.is_null() {
                None
            } else {
                // RISE FROM THE GRAVE
                let head = Box::from_raw(self.head);
                self.head = head.next;

                if self.head.is_null() {
                    self.tail = ptr::null_mut();
                }

                Some(head.elem)
            }
        }
    }
```

我们漂亮的小`take`和`map`方法已经死了，因为不使用`Option`，也就不能用了，现在只需手动检查并设置为`null`。

既然我们在这里，让我们加入析构函数。这次我们将它实现为重复弹出，因为它很可爱也很简单：

```rust
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        while let Some(_) = self.pop() { }
    }
}
```

现在到了关键时刻，测试：

```rust
#[cfg(test)]
mod test {
    use super::List;
    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop(), None);

        // Populate list
        list.push(1);
        list.push(2);
        list.push(3);

        // Check normal removal
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push(4);
        list.push(5);

        // Check normal removal
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), None);

        // Check the exhaustion case fixed the pointer right
        list.push(6);
        list.push(7);

        // Check normal removal
        assert_eq!(list.pop(), Some(6));
        assert_eq!(list.pop(), Some(7));
        assert_eq!(list.pop(), None);
    }
}
```

```txt
> cargo test

running 12 tests
test fifth::test::basics ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test fourth::test::into_iter ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 12 passed; 0 failed; 0 ignored; 0 measured
```

很好，但是`miri`同意吗？

```txt
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2023-02-14 miri test

running 12 tests
test fifth::test::basics ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test fourth::test::into_iter ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 12 passed; 0 failed; 0 ignored; 0 measured
```

耶耶耶！！！！！

它真管用！

可能吧！

找不到未定义的行为并不能证明它不存在，等着引起问题，但我对一本关于链表的笑话书的严谨程度是有限的，所以我们将称其为100%机器验证证明，任何说不是的人都可以吃我的COQ（一个流行的形式化验证工具）！

 ∴ QED（证明完毕） □

## 6.8 额外的垃圾代码

现在已经编写了`push`和`pop`，奇怪的是，其他所有内容实际上与之前实现的栈（`stack`）情况完全相同。只有更改列表长度的操作才需要接触尾指针。

但当然，既然一切都是不安全指针，我们需要重写代码以使用它们！如果我们要接触所有代码，我们不妨抓住机会确保我们没有遗漏任何东西。

但无论如何，让我们开始从栈的实现中复制粘贴代码，这是之前的代码：

```rust
// ...

pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}
```

`IntoIter`看起来不错，但`Iter`和`IterMut`违反了我们不再在类型中使用安全指针的简单规则。让我们安全地将它们更改为使用原始指针：

```rust
pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: *mut Node<T>,
}

pub struct IterMut<'a, T> {
    next: *mut Node<T>,
}

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }

    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head }
    }

    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        IterMut { next: self.head }
    }
}
```

看起来不错！

```txt
error[E0392]: parameter `'a` is never used
  --> src\fifth.rs:17:17
   |
17 | pub struct Iter<'a, T> {
   |                 ^^ unused parameter
   |
   = help: consider removing `'a`, referring to it in a field, 
     or using a marker such as `PhantomData`

error[E0392]: parameter `'a` is never used
  --> src\fifth.rs:21:20
   |
21 | pub struct IterMut<'a, T> {
   |                    ^^ unused parameter
   |
   = help: consider removing `'a`, referring to it in a field, 
     or using a marker such as `PhantomData`
```

看起来不太好！他们说的[`PhantomData`](https://doc.rust-lang.org/std/marker/struct.PhantomData.html)是什么？

（原文）
> Zero-sized type used to mark things that “act like” they own a `T`.
>
> Adding a `PhantomData<T>` field to your type tells the compiler that your type acts as though it stores a value of type `T`, even though it doesn’t really. This information is used when computing certain safety properties.
>
> For a more in-depth explanation of how to use `PhantomData<T>`, please see [the Nomicon](https://doc.rust-lang.org/nightly/nomicon/).

（翻译）
> 零大小类型用于标记“表现得像”拥有`T`的事物。
>
> 向类型添加`PhantomData<T>`字段会告诉编译器，您的类型表现得好像存储了类型`T`的值，即使事实并非如此。此信息用于计算某些安全属性。
>
> 有关如何使用`PhantomData<T>`的更深入说明，请参阅[Nomicon](https://doc.rust-lang.org/nightly/nomicon/)。

嘿，别急，我们正在读我写的书。不是某个大书呆子可能写的其他书！我敢打赌，如果他们在那里写一个数据结构，那一定是数组堆栈之类的蹩脚东西，而不是链表。

（原文）
> Unused lifetime parameters
>
> Perhaps the most common use case for PhantomData is a struct that has an unused lifetime parameter, typically as part of some unsafe code.

（翻译）
> 未使用的生命周期参数
>
> `PhantomData`最常见的用例可能是具有未使用的生命周期参数的结构，通常作为某些不安全代码的一部分。

啊，所以我们在类型中命名了一个生命周期，但实际上并没有使用它。我们可以沿着`PhantomData`路径走下去，但我想把它留到下一章中真正需要它的双向链表。

我们处于一个有趣的情况，我们实际上并不需要`PhantomData`。我想。我只是要声明这一点并相信这是真的，如果`miri`最后对我们大喊大叫，我会承认这一点，我们会做`PhantomData`的事情。

我们实际上要做的是将引用放回这些迭代器类型中，并很高兴我们仍然可以在某些地方使用引用。我认为这是合理的，因为使用迭代器时仍然有一种适当的嵌套：创建迭代器，使用安全引用一段时间，然后丢弃迭代器。

只有迭代器消失后，您才能访问列表并调用`push`和`pop`之类的操作，这些操作需要与尾指针和`Box`混淆。现在，在迭代过程中，我们将解引用一堆原始指针，因此存在某种混合，但我们应该能够将这些引用视为对不安全指针的重新借用。

我甚至不是100%确信，但我只是想尝试一下看看！

```rust
pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }

    pub fn iter(&self) -> Iter<'_, T> {
        unsafe {
            Iter { next: self.head.as_ref() }
        }
    }

    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        unsafe {
            IterMut { next: self.head.as_mut() }
        }
    }
}
```

如果我们要存储引用，我们需要将原始指针升级为引用选项。我们可以检查指针是否为空，但这是我认为可以使用讨厌的`ptr::as_ref`和`ptr::as_mut`方法的极其狭窄的情况之一。

我通常建议像躲避瘟疫一样避免使用这些方法，因为它们会做一些令人惊讶和讨厌的事情，而且它们本质上会重新引入引用，而我的整个“简单规则”就是避免这样做！

这些方法有很多警告，但最有趣的是：

（原文）
> You must enforce Rust’s aliasing rules, since the returned lifetime 'a is arbitrarily chosen and does not necessarily reflect the actual lifetime of the data. In particular, for the duration of this lifetime, the memory the pointer points to must not get accessed (read or written) through any other pointer.

（翻译）
> 您必须强制执行Rust的别名规则，因为返回的生命周期`'a`是任意选择的，不一定反映数据的实际生命周期。特别是，在此生命周期内，指针指向的内存不得通过任何其他指针访问（读取或写入）。

嘿，看，这就是我们花了25页讨论的内容！我已经断言我们在这里使用引用绝对没问题，所以别名问题解决了！另一个邪恶的部分是签名：

```rust
    pub unsafe fn as_mut<'a>(self) -> Option<&'a mut T>
```

您是否看到了，由于`self`是按值传递的，因此生命周期与输入完全无关？是的，这就是我们所说的“无界生命周期”，而且它很讨厌。它愿意假装像我们要求的那样大，甚至是“静态”！处理这个问题的方法是将它放在有界的地方，这通常意味着“尽快从函数中返回它，以便函数签名限制它”。

天哪，我对此很紧张，但我们会继续努力！让我们从堆栈中窃取一些迭代器实现：

```rust
impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.pop()
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        unsafe {
            self.next.map(|node| {
                self.next = node.next.as_ref();
                &node.elem
            })
        }
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        unsafe {
            self.next.take().map(|node| {
                self.next = node.next.as_mut();
                &mut node.elem
            })
        }
    }
}
```

关键时刻……测试

让我们先从`second.rs`复制足够的测试用例`into_iter`、`iter`、`iter_mut`，然后调整断言，因为`second.rs`是堆栈，而现在我们正在做的是队列。

```txt
> cargo test

running 15 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::iter ... ok
test third::test::basics ... ok

test result: ok. 15 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out;
```

```txt
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2023-02-14 miri test

running 15 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 15 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

是的！！！请听我说！有时我不会犯错！

> **旁白**：但错误存在的意义不就是为了教育读者吗？

是的，有时候教训就是我是对的，当我谈论不安全代码时每个人都应该听我说，因为我花了太多时间思考迭代器实现的合理性？！好吗？！好。

让我们继续做`peek`和`peek_mut`。

```rust
    pub fn peek(&self) -> Option<&T> {
        unsafe {
            self.head.as_ref()
        }
    }

    pub fn peek_mut(&mut self) -> Option<&mut T> {
        unsafe {
            self.head.as_mut()
        }
    }
```

我甚至不会测试它们，因为我不再犯错。

> **旁白**：`cargo build`

```txt
error[E0308]: mismatched types
  --> src\fifth.rs:66:13
   |
25 | impl<T> List<T> {
   |      - this type parameter
...
64 |     pub fn peek(&self) -> Option<&T> {
   |                           ---------- expected `Option<&T>` 
   |                                      because of return type
65 |         unsafe {
66 |             self.head.as_ref()
   |             ^^^^^^^^^^^^^^^^^^ expected type parameter `T`, 
   |                                found struct `fifth::Node`
   |
   = note: expected enum `Option<&T>`
              found enum `Option<&fifth::Node<T>>`
```

非常好！出错了！类型不匹配，需要用`map`方法处理一下。

```rust
    pub fn peek(&self) -> Option<&T> {
        unsafe {
            self.head.as_ref().map(|node| &node.elem)
        }
    }

    pub fn peek_mut(&mut self) -> Option<&mut T> {
        unsafe {
            self.head.as_mut().map(|node| &mut node.elem)
        }
    }
```

我想我还会继续犯错误，所以我们要格外小心，并添加一个我称之为“miri food”的新测试：它会弄乱并混合我们的API，以帮助`miri`发现我们的错误。

```rust
    #[test]
    fn miri_food() {
        let mut list = List::new();

        list.push(1);
        list.push(2);
        list.push(3);

        assert!(list.pop() == Some(1));
        list.push(4);
        assert!(list.pop() == Some(2));
        list.push(5);

        assert!(list.peek() == Some(&3));
        list.push(6);
        list.peek_mut().map(|x| *x *= 10);
        assert!(list.peek() == Some(&30));
        assert!(list.pop() == Some(30));

        for elem in list.iter_mut() {
            *elem *= 100;
        }

        let mut iter = list.iter();
        assert_eq!(iter.next(), Some(&400));
        assert_eq!(iter.next(), Some(&500));
        assert_eq!(iter.next(), Some(&600));
        assert_eq!(iter.next(), None);
        assert_eq!(iter.next(), None);

        assert!(list.pop() == Some(400));
        list.peek_mut().map(|x| *x *= 10);
        assert!(list.peek() == Some(&5000));
        list.push(7);

        // Drop it on the ground and let the dtor exercise itself
    }
```

```txt
> cargo test

running 16 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test fifth::test::miri_food ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::iter ... ok
test second::test::iter ... ok
test third::test::basics ... ok

test result: ok. 16 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out



MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2023-02-14 miri test

running 16 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test fifth::test::miri_food ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::iter ... ok
test second::test::iter ... ok
test third::test::basics ... ok

test result: ok. 16 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

完美！

## 6.9 最终代码

好吧，通过一点点不安全，我们设法在简单的安全队列上实现了线性时间改进，并且我们设法重用了之前实现的安全的栈中的几乎所有逻辑！

你知道，除了`miri`完全向我们灌输的那一部分，我们必须就Rust的内存模型写一篇简短的硕士论文。你知道，正如你所知道的。

但好的一面是我们不必编写任何使用垃圾的`Rc`或`RefCell`的东西。

```rust
use std::ptr;

pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>,
}

type Link<T> = *mut Node<T>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: ptr::null_mut(), tail: ptr::null_mut() }
    }
    pub fn push(&mut self, elem: T) {
        unsafe {
            let new_tail = Box::into_raw(Box::new(Node {
                elem: elem,
                next: ptr::null_mut(),
            }));

            if !self.tail.is_null() {
                (*self.tail).next = new_tail;
            } else {
                self.head = new_tail;
            }

            self.tail = new_tail;
        }
    }
    pub fn pop(&mut self) -> Option<T> {
        unsafe {
            if self.head.is_null() {
                None
            } else {
                let head = Box::from_raw(self.head);
                self.head = head.next;

                if self.head.is_null() {
                    self.tail = ptr::null_mut();
                }

                Some(head.elem)
            }
        }
    }

    pub fn peek(&self) -> Option<&T> {
        unsafe {
            self.head.as_ref().map(|node| &node.elem)
        }
    }

    pub fn peek_mut(&mut self) -> Option<&mut T> {
        unsafe {
            self.head.as_mut().map(|node| &mut node.elem)
        }
    }

    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }

    pub fn iter(&self) -> Iter<'_, T> {
        unsafe {
            Iter { next: self.head.as_ref() }
        }
    }

    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        unsafe {
            IterMut { next: self.head.as_mut() }
        }
    }
}

impl<T> Drop for List<T> {
    fn drop(&mut self) {
        while let Some(_) = self.pop() { }
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.pop()
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        unsafe {
            self.next.map(|node| {
                self.next = node.next.as_ref();
                &node.elem
            })
        }
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        unsafe {
            self.next.take().map(|node| {
                self.next = node.next.as_mut();
                &mut node.elem
            })
        }
    }
}

#[cfg(test)]
mod test {
    use super::List;
    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop(), None);

        // Populate list
        list.push(1);
        list.push(2);
        list.push(3);

        // Check normal removal
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push(4);
        list.push(5);

        // Check normal removal
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), None);

        // Check the exhaustion case fixed the pointer right
        list.push(6);
        list.push(7);

        // Check normal removal
        assert_eq!(list.pop(), Some(6));
        assert_eq!(list.pop(), Some(7));
        assert_eq!(list.pop(), None);
    }

    #[test]
    fn into_iter() {
        let mut list = List::new();
        list.push(1); list.push(2); list.push(3);

        let mut iter = list.into_iter();
        assert_eq!(iter.next(), Some(1));
        assert_eq!(iter.next(), Some(2));
        assert_eq!(iter.next(), Some(3));
        assert_eq!(iter.next(), None);
    }

    #[test]
    fn iter() {
        let mut list = List::new();
        list.push(1); list.push(2); list.push(3);

        let mut iter = list.iter();
        assert_eq!(iter.next(), Some(&1));
        assert_eq!(iter.next(), Some(&2));
        assert_eq!(iter.next(), Some(&3));
        assert_eq!(iter.next(), None);
    }

    #[test]
    fn iter_mut() {
        let mut list = List::new();
        list.push(1); list.push(2); list.push(3);

        let mut iter = list.iter_mut();
        assert_eq!(iter.next(), Some(&mut 1));
        assert_eq!(iter.next(), Some(&mut 2));
        assert_eq!(iter.next(), Some(&mut 3));
        assert_eq!(iter.next(), None);
    }

    #[test]
    fn miri_food() {
        let mut list = List::new();

        list.push(1);
        list.push(2);
        list.push(3);

        assert!(list.pop() == Some(1));
        list.push(4);
        assert!(list.pop() == Some(2));
        list.push(5);

        assert!(list.peek() == Some(&3));
        list.push(6);
        list.peek_mut().map(|x| *x *= 10);
        assert!(list.peek() == Some(&30));
        assert!(list.pop() == Some(30));

        for elem in list.iter_mut() {
            *elem *= 100;
        }

        let mut iter = list.iter();
        assert_eq!(iter.next(), Some(&400));
        assert_eq!(iter.next(), Some(&500));
        assert_eq!(iter.next(), Some(&600));
        assert_eq!(iter.next(), None);
        assert_eq!(iter.next(), None);

        assert!(list.pop() == Some(400));
        list.peek_mut().map(|x| *x *= 10);
        assert!(list.peek() == Some(&5000));
        list.push(7);

        // Drop it on the ground and let the dtor exercise itself
    }
}
```
