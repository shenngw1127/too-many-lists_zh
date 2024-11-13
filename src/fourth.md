# 5. 糟糕但安全的双链双端队列

现在我们已经了解了`Rc`并了解了内部可变性，这给了我们一个有趣的想法……也许我们可以通过`Rc`执行改变操作。如果是这样的话，也许我们可以完全安全地实现双向链表！

在此过程中，我们将熟悉内部可变性，并且可能通过艰难的方式了解到安全并不意味着正确。双向链表很难，我总是在某个地方犯错。

让我们添加一个名为`fourth.rs`的新文件：

```rust
// in lib.rs

pub mod first;
pub mod second;
pub mod third;
pub mod fourth;
```

这一次也是全新的操作，尽管像往常一样，我们可能会发现一些之前可以复用的逻辑。

免责声明：本章基本上证明了这是一个**非常糟糕**的想法。

## 5.1 布局

我们设计的关键是使用`RefCell`类型。`RefCell`的核心是一对方法：

```rust
fn borrow(&self) -> Ref<'_, T>;
fn borrow_mut(&self) -> RefMut<'_, T>;
```

`borrow`和`borrow_mut`的规则与`&`和`&mut`完全相同：你可以根据需要多次调用`borrow`，但`borrow_mut`需要排他性。

`RefCell`不是静态地强制执行这些规则，而是在运行时强制执行这些规则。如果你违反规则，`RefCell`就会崩溃并使程序崩溃。为什么它会返回这些`Ref`和`RefMut`东西？好吧，它们基本上像`Rc`一样，但是用于借用。它们还会保持`RefCell`的借用状态，直到它们超出范围。我们稍后会讲到这一点。

现在有了`Rc`和`RefCell`，我们可以成为……一种极其冗长、普遍可变的垃圾收集语言，但还是无法收集循环引用！是的……

好吧，我们想要实现双重链接的列表。这意味着每个节点都有一个指向前一个和下一个节点的指针。此外，列表本身有一个指向第一个和最后一个节点的指针。这使我们能够在列表的两端快速插入和删除。

所以我们可能想要类似的东西：

```rust
use std::rc::Rc;
use std::cell::RefCell;

pub struct List<T> {
    head: Link<T>,
    tail: Link<T>,
}

type Link<T> = Option<Rc<RefCell<Node<T>>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
    prev: Link<T>,
}
```

```txt
> cargo build

warning: field is never used: `head`
 --> src/fourth.rs:5:5
  |
5 |     head: Link<T>,
  |     ^^^^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: field is never used: `tail`
 --> src/fourth.rs:6:5
  |
6 |     tail: Link<T>,
  |     ^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/fourth.rs:12:5
   |
12 |     elem: T,
   |     ^^^^^^^

warning: field is never used: `next`
  --> src/fourth.rs:13:5
   |
13 |     next: Link<T>,
   |     ^^^^^^^^^^^^^

warning: field is never used: `prev`
  --> src/fourth.rs:14:5
   |
14 |     prev: Link<T>,
   |     ^^^^^^^^^^^^^
```

嘿，它构建成功了！有很多死代码警告，但它构建成功了！让我们尝试使用它。

## 5.2 增加元素

好吧，我们先从构建`List`开始。使用这个新系统，这非常简单。`new`仍然很简单，只需将所有字段都设置为`None`即可。此外，由于它变得有点笨重，我们也来分解一个`Node`构造函数：

```rust
impl<T> Node<T> {
    fn new(elem: T) -> Rc<RefCell<Self>> {
        Rc::new(RefCell::new(Node {
            elem: elem,
            prev: None,
            next: None,
        }))
    }
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }
}
```

```txt
> cargo build

**A BUNCH OF DEAD CODE WARNINGS BUT IT BUILT**
```

耶！

现在让我们尝试将押入写入列表的前面。由于双向链表要复杂得多，因此我们需要做更多的工作。单链表操作可以简化为简单的一行代码，而双向链表操作则相当复杂。

具体来说，我们现在需要特别处理空列表周围的一些边界情况。大多数操作只会触及头指针或尾指针。然而，当转换到空列表或从空列表转换时，我们需要同时编辑两者。

验证我们的方法是否合理的一种简单方法是，我们是否保持以下不变：**每个节点应该恰好有两个指向它的指针**。**列表中间的每个节点都由其前任和后继指向，而两端的节点则由列表本身指向**。

让我们来尝试一下：

```rust
    pub fn push_front(&mut self, elem: T) {
        // new node needs +2 links, everything else should be +0
        let new_head = Node::new(elem);
        match self.head.take() {
            Some(old_head) => {
                // non-empty list, need to connect the old_head
                old_head.prev = Some(new_head.clone()); // +1 new_head
                new_head.next = Some(old_head);         // +1 old_head
                self.head = Some(new_head);             // +1 new_head, -1 old_head
                // total: +2 new_head, +0 old_head -- OK!
            }
            None => {
                // empty list, need to set the tail
                self.tail = Some(new_head.clone());     // +1 new_head
                self.head = Some(new_head);             // +1 new_head
                // total: +2 new_head -- OK!
            }
        }
    }
```

```txt
> cargo build

error[E0609]: no field `prev` on type `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>`
  --> src/fourth.rs:39:26
   |
39 |                 old_head.prev = Some(new_head.clone()); // +1 new_head
   |                          ^^^^ unknown field

error[E0609]: no field `next` on type `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>`
  --> src/fourth.rs:40:26
   |
40 |                 new_head.next = Some(old_head);         // +1 old_head
   |                          ^^^^ unknown field
```

好吧。编译器错误。好的开始。好的开始。

为什么我们不能访问节点上的`prev`和`next`字段？以前我们只有`Rc<Node>`时，它还能正常工作。似乎`RefCell`妨碍了我们。

我们应该检查一下文档。

Google搜索一下“rust refcell”

[点击第一个链接](https://doc.rust-lang.org/std/cell/struct.RefCell.html)

（原文）
> A mutable memory location with dynamically checked borrow rules
>
> See the [module-level documentation](https://doc.rust-lang.org/std/cell/index.html) for more.

（翻译）
> 具有动态检查借用规则的可变内存位置
>
> 查看[模块级文档](https://doc.rust-lang.org/std/cell/index.html)获取更多信息。

点击链接

（原文）
> Shareable mutable containers.
>
> Values of the `Cell<T>` and `RefCell<T>` types may be mutated through shared references (i.e. the common &T type), whereas most Rust types can only be mutated through unique (&mut T) references. We say that `Cell<T>` and `RefCell<T>` provide 'interior mutability', in contrast with typical Rust types that exhibit 'inherited mutability'.
>
> Cell types come in two flavors: `Cell<T>` and `RefCell<T>`. `Cell<T>` provides get and set methods that change the interior value with a single method call. `Cell<T>` though is only compatible with types that implement Copy. For other types, one must use the `RefCell<T>` type, acquiring a write lock before mutating.
>
> `RefCell<T>` uses Rust's lifetimes to implement 'dynamic borrowing', a process whereby one can claim temporary, exclusive, mutable access to the inner value. Borrows for `RefCell<T>`s are tracked 'at runtime', unlike Rust's native reference types which are entirely tracked statically, at compile time. Because `RefCell<T>` borrows are dynamic it is possible to attempt to borrow a value that is already mutably borrowed; when this happens it results in thread panic.
>
> ## When to choose interior mutability
>
> The more common inherited mutability, where one must have unique access to mutate a value, is one of the key language elements that enables Rust to reason strongly about pointer aliasing, statically preventing crash bugs. Because of that, inherited mutability is preferred, and interior mutability is something of a last resort. Since cell types enable mutation where it would otherwise be disallowed though, there are occasions when interior mutability might be appropriate, or even must be used, e.g.
>
> - Introducing inherited mutability roots to shared types.
> - Implementation details of logically-immutable methods.
> - Mutating implementations of Clone.
>
> ### Introducing inherited mutability roots to shared types
>
> Shared smart pointer types, including `Rc<T>` and `Arc<T>`, provide containers that can be cloned and shared between multiple parties. Because the contained values may be multiply-aliased, they can only be borrowed as shared references, not mutable references. Without cells it would be impossible to mutate data inside of shared boxes at all!
>
> It's very common then to put a `RefCell<T>` inside shared pointer types to reintroduce mutability:
>
> ```rust
> use std::collections::HashMap;
> use std::cell::RefCell;
> use std::rc::Rc;
>
> fn main() {
>     let shared_map: Rc<RefCell<_>> = Rc::new(RefCell::new(HashMap::new()));
>     shared_map.borrow_mut().insert("africa", 92388);
>     shared_map.borrow_mut().insert("kyoto", 11837);
>     shared_map.borrow_mut().insert("piccadilly", 11826);
>     shared_map.borrow_mut().insert("marbles", 38);
> }
> ```
>
> Note that this example uses `Rc<T>` and not `Arc<T>`. `RefCell<T>`s are for single-threaded scenarios. Consider using `Mutex<T>` if you need shared mutability in a multi-threaded situation.

（翻译）
> 可共享的可变容器。
>
> `Cell<T>`和`RefCell<T>`类型的值可以通过共享引用（即通用`&T`类型）进行改变，而大多数Rust类型只能通过唯一 (`&mut T`) 引用进行改变。我们说`Cell<T>`和`RefCell<T>`提供“内部可变性”，而典型的Rust类型则表现出“继承可变性”。
>
> `Cell`类型有两种类型：`Cell<T>`和`RefCell<T>`。`Cell<T>`提供`get`和`set`方法，可通过单个方法调用更改内部值。但是`Cell<T>`仅与实现`Copy`的类型兼容。对于其他类型，必须使用`RefCell<T>`类型，在修改之前还要获取写锁。
>
> `RefCell<T>`使用Rust的生命周期来实现“动态借用”，这是一个可以声明对内部值的临时、独占、可变访问的过程。`RefCell<T>`的借用在“运行时”进行跟踪，而Rust的原生引用类型则在编译时完全静态跟踪。由于`RefCell<T>`的借用是动态的，因此可以尝试借用已经可变借用的值；当发生这种情况时，会导致线程恐慌。
>
> ## 何时选择内部可变性
>
> 更常见的继承可变性（即必须具有唯一访问权限才能改变值）是Rust能够强有力地推理指针别名的关键语言元素之一，可静态防止崩溃错误。因此，继承可变性是首选，而内部可变性则是最后的手段。由于单元格类型允许在原本不允许的地方进行改变，因此在某些情况下内部可变性可能是合适的，甚至必须使用，例如
>
> 将继承的可变性根引入共享类型。
> 逻辑上不可变的方法的实现细节。
> `clone`的可修改实现。
>
> ### 将继承的可变性根引入共享类型
>
> 共享智能指针类型（包括`Rc<T>`和`Arc<T>`）提供可在多方之间克隆和共享的容器。由于所包含的值可能被多次别名，因此它们只能作为共享引用借用，而不能作为可变引用借用。如果没有单元格，就根本不可能在共享框内改变数据！
>
> 因此，将`RefCell<T>`放在共享指针类型中以重新引入可变性是非常常见的：
>
> ```rust
> use std::collections::HashMap;
> use std::cell::RefCell;
> use std::rc::Rc;
>
> fn main() {
>     let shared_map: Rc<RefCell<_>> = Rc::new(RefCell::new(HashMap::new()));
>     shared_map.borrow_mut().insert("africa", 92388);
>     shared_map.borrow_mut().insert("kyoto", 11837);
>     shared_map.borrow_mut().insert("piccadilly", 11826);
>     shared_map.borrow_mut().insert("marbles", 38);
> }
> ```
>
> 请注意，此示例使用的是`Rc<T>`而不是`Arc<T>`。`RefCell<T>`适用于单线程场景。如果您在多线程情况下需要共享可变性，请考虑使用`Mutex<T>`。

嘿，Rust的文档仍然非常棒。

我们关心的重点是这一行：

```rust
    shared_map.borrow_mut().insert("africa", 92388);
```

特别是`borrow_mut`相关的事情。看来我们需要明确借用一个`RefCell`。`.`操作符不会为我们做这件事。很奇怪。让我们试试：

```rust
    pub fn push_front(&mut self, elem: T) {
        let new_head = Node::new(elem);
        match self.head.take() {
            Some(old_head) => {
                old_head.borrow_mut().prev = Some(new_head.clone());
                new_head.borrow_mut().next = Some(old_head);
                self.head = Some(new_head);
            }
            None => {
                self.tail = Some(new_head.clone());
                self.head = Some(new_head);
            }
        }
    }
```

```txt
> cargo build

warning: field is never used: `elem`
  --> src/fourth.rs:12:5
   |
12 |     elem: T,
   |     ^^^^^^^
   |
   = note: #[warn(dead_code)] on by default
```

嘿，建好了！文档又赢了。

## 5.3 移除元素

`pop_front`应该与`push_front`的基本逻辑相同，但是是反向的。让我们尝试一下：

```rust
    pub fn pop_front(&mut self) -> Option<T> {
        // need to take the old head, ensuring it's -2
        self.head.take().map(|old_head| {                         // -1 old
            match old_head.borrow_mut().next.take() {
                Some(new_head) => {                               // -1 new
                    // not emptying list
                    new_head.borrow_mut().prev.take();            // -1 old
                    self.head = Some(new_head);                   // +1 new
                    // total: -2 old, +0 new
                }
                None => {
                    // emptying list
                    self.tail.take();                             // -1 old
                    // total: -2 old, (no new)
                }
            }
            old_head.elem
        })
    }
```

```txt
> cargo build

error[E0609]: no field `elem` on type `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>`
  --> src/fourth.rs:64:22
   |
64 |             old_head.elem
   |                      ^^^^ unknown field
```

收到。`RefCells`。我猜得再次使用`borrow_mut`...

```rust
    pub fn pop_front(&mut self) -> Option<T> {
        self.head.take().map(|old_head| {
            match old_head.borrow_mut().next.take() {
                Some(new_head) => {
                    new_head.borrow_mut().prev.take();
                    self.head = Some(new_head);
                }
                None => {
                    self.tail.take();
                }
            }
            old_head.borrow_mut().elem
        })
    }
```

```txt
> cargo build

error[E0507]: cannot move out of borrowed content
  --> src/fourth.rs:64:13
   |
64 |             old_head.borrow_mut().elem
   |             ^^^^^^^^^^^^^^^^^^^^^^^^^^ cannot move out of borrowed content
```

（原文）
> cannot move out of borrowed content

（翻译）
> 不能移出借用的内容

嗯……看来`Box`真的把我们宠坏了。`borrow_mut`只能给我们一个`&mut Node<T>`，但我们无法移出它！

我们需要一个可以接受`RefCell<T>`并返回`T`的东西。让我们查看[文档](https://doc.rust-lang.org/std/cell/struct.RefCell.html)以查找类似的东西：

（原文）
> ``fn into_inner(self) -> T``
>
> Consumes the RefCell, returning the wrapped value.

（翻译）
> ``fn into_inner(self) -> T``
>
> 消费`RefCell`，返回包装的值。

看起来很有希望！

```rust
    old_head.into_inner().elem
```

```txt
> cargo build

error[E0507]: cannot move out of an `Rc`
  --> src/fourth.rs:64:13
   |
64 |             old_head.into_inner().elem
   |             ^^^^^^^^ cannot move out of an `Rc`
```

啊，该死。`into_inner`想要移出`RefCell`，但我们不能，因为它在`Rc`中。正如我们在上一章中看到的，`Rc<T>`只允许我们将共享引用放入其内部。这是有道理的，因为这就是引用计数指针的全部意义所在：它们是共享的！

当我们想要为引用计数列表实现`Drop`时，这对我们来说是一个问题，解决方案是相同的：`Rc::try_unwrap`，如果`Rc`的引用计数为`1`，它会移出`Rc`的内容。

```rust
    Rc::try_unwrap(old_head).unwrap().into_inner().elem
```

`Rc::try_unwrap`返回一个`Result<T, Rc<T>>`。结果基本上是一个通用选项，其中`None`情况有与之关联的数据。在本例中，就是您尝试解包的`Rc`。由于我们不关心失败的情况（如果我们正确编写了程序，它就必须成功），我们只需对其调用`unwrap`即可。 无论如何，让我们看看接下来会得到什么编译器错误（让我们面对现实，肯定会有一个）。

```txt
> cargo build

error[E0599]: no method named `unwrap` found for type `std::result::Result<std::cell::RefCell<fourth::Node<T>>, std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>>` in the current scope
  --> src/fourth.rs:64:38
   |
64 |             Rc::try_unwrap(old_head).unwrap().into_inner().elem
   |                                      ^^^^^^
   |
   = note: the method `unwrap` exists but the following trait bounds were not satisfied:
           `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>> : std::fmt::Debug`
```

呃。`Result`上的`unwrap`要求您可以调试打印错误情况。`RefCell<T>`仅在`T`实现`Debug`时才会实现`Debug`特征。而`Node`未实现`Debug`特征。

与其这样做，不如通过将`Result`利用`ok`方法转换为`Option`来解决这个问题：

```rust
    Rc::try_unwrap(old_head).ok().unwrap().into_inner().elem
```

请。

```txt
> cargo build
```

是的。

呼

我们做到了。

我们实现了押入和弹出。

让我们通过复制旧的堆栈的基本测试来进行测试（因为这是我们迄今为止实现的全部）：

```rust
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop_front(), None);

        // Populate list
        list.push_front(1);
        list.push_front(2);
        list.push_front(3);

        // Check normal removal
        assert_eq!(list.pop_front(), Some(3));
        assert_eq!(list.pop_front(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push_front(4);
        list.push_front(5);

        // Check normal removal
        assert_eq!(list.pop_front(), Some(5));
        assert_eq!(list.pop_front(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop_front(), Some(1));
        assert_eq!(list.pop_front(), None);
    }
}
```

```txt
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 9 tests
test first::test::basics ... ok
test fourth::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::basics ... ok
test fifth::test::iter_mut ... ok
test third::test::basics ... ok
test second::test::iter ... ok
test third::test::iter ... ok
test second::test::into_iter ... ok

test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured
```

搞定了。

现在我们可以正确地从列表中删除内容，我们可以实现`Drop`。这次`Drop`在概念上更有趣一些。以前我们费尽心思为堆栈实现`Drop`只是为了避免无限制递归，现在我们需要实现`Drop`才能让任何事情发生。

`Rc`无法处理循环。如果存在循环，那么其他一切都会继续存在。事实证明，双向链表只是由许多微小循环组成的大链！因此，当我们删除列表时，两个末端节点的引用计数将减少到`1`，...然后就不会发生其他任何事情了。好吧，如果我们的列表只包含一个节点，那么我们就可以开始了。但理想情况下，如果列表包含多个元素，它应该可以正常工作。也许这只是我的想法。

正如我们所见，删除元素有点痛苦。因此，对我们来说，最简单的事情就是弹出，直到得到`None`：

```rust
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        while self.pop_front().is_some() {}
    }
}
```

```txt
> cargo build
```

（我们实际上可以用可变堆栈来实现这一点，但捷径是为那些理解事物的人准备的！）

我们可以考虑实现`push`和`pop`的`_back`版本，但它们只是复制粘贴的工作，我们将在本章后面讨论。现在让我们看看更有趣的事情！

## 5.4 `peek`方法

好吧，我们完成了押入和弹出。我不撒谎，当时有点激动。编译时正确性是一剂强心剂。

让我们通过做一些简单的事情来放松一下：让我们实现`peek_front`。以前这总是很容易的。现在应该还是很容易的，对吧？

对吧？

事实上，我想我可以直接复制粘贴它！

```rust
    pub fn peek_front(&self) -> Option<&T> {
        self.head.as_ref().map(|node| {
            &node.elem
        })
    }
```

等等。这次不行。需要加上`borrow`

```rust
    pub fn peek_front(&self) -> Option<&T> {
        self.head.as_ref().map(|node| {
            // BORROW!!!!
            &node.borrow().elem
        })
    }
```

哈哈。

```txt
> cargo build

error[E0515]: cannot return value referencing temporary value
  --> src/fourth.rs:66:13
   |
66 |             &node.borrow().elem
   |             ^   ----------^^^^^
   |             |   |
   |             |   temporary value created here
   |             |
   |             returns a value referencing data owned by the current function
```

好吧，我只是在烧毁我的电脑。

这与我们的单链堆栈的逻辑完全相同。为什么事情会有所不同。为什么。

答案实际上是本章的全部寓意：`RefCells`让一切变得悲伤。到目前为止，`RefCells`只是一种滋扰。现在它们将成为一场噩梦。

那么到底发生了什么？为了理解这一点，我们需要回到`borrow`方法的定义：

```rust
    fn borrow<'a>(&'a self) -> Ref<'a, T>
    fn borrow_mut<'a>(&'a self) -> RefMut<'a, T>
```

在布局部分我们说过：

> `RefCell`不是静态地强制执行这些规则，而是在运行时强制执行这些规则。如果你违反规则，`RefCell`就会崩溃并使程序崩溃。为什么它会返回这些`Ref`和`RefMut`东西？好吧，它们基本上像`Rc`一样，但是用于借用。此外，它们会保持`RefCell`的借用状态，直到它们超出范围。我们稍后会讲到这一点。

现在就是这个稍后。

`Ref`和`RefMut`分别实现`Deref`和`DerefMut`。因此，在大多数情况下，它们的行为与`&T`和`&mut T`完全相同。但是，由于这些特征的工作方式，返回的引用与`Ref`的生命周期有关，而不是实际的`RefCell`。这意味着只要我们保留引用，`Ref`就必须存在。

事实上，这对于正确性来说是必要的。当`Ref`被删除时，它会告诉`RefCell`它不再被借用。因此，如果我们确实设法将引用保留的时间比`Ref`存在的时间更长，那么当引用四处游走时，我们可能会得到一个`RefMut`，从而完全破坏Rust的类型系统。

那么这给我们留下了什么？我们只想返回一个引用，但我们需要保留这个`Ref`东西。但是，一旦我们从`peek`返回引用，函数就结束了，`Ref`超出了范围。

😖

据我所知，我们实际上已经完全陷入困境。你不能完全封装`RefCell`的使用。

但是……如果我们放弃完全隐藏我们的实现细节会怎样？如果我们返回`Ref`会怎样？

```rust
    pub fn peek_front(&self) -> Option<Ref<T>> {
        self.head.as_ref().map(|node| {
            node.borrow()
        })
    }
```

```txt
> cargo build

error[E0412]: cannot find type `Ref` in this scope
  --> src/fourth.rs:63:40
   |
63 |     pub fn peek_front(&self) -> Option<Ref<T>> {
   |                                        ^^^ not found in this scope
help: possible candidates are found in other modules, you can import them into scope
   |
1  | use core::cell::Ref;
   |
1  | use std::cell::Ref;
   |
```

哦。得导入一些东西。

```rust
use std::cell::{Ref, RefCell};
```

```txt
> cargo build

error[E0308]: mismatched types
  --> src/fourth.rs:64:9
   |
64 | /         self.head.as_ref().map(|node| {
65 | |             node.borrow()
66 | |         })
   | |__________^ expected type parameter, found struct `fourth::Node`
   |
   = note: expected type `std::option::Option<std::cell::Ref<'_, T>>`
              found type `std::option::Option<std::cell::Ref<'_, fourth::Node<T>>>`
```

嗯...没错。我们有一个`Ref<Node<T>>`，但我们想要一个`Ref<T>`。我们可以放弃所有封装的希望，只返回它。我们还可以让事情变得更加复杂，将`Ref<Node<T>>`包装成一种新类型，只公开对`&T`的访问。

这两个选项都有点蹩脚。

相反，我们要更深入地研究。让我们找点乐子。我们的乐趣来源是这个野兽：

```rust
map<U, F>(orig: Ref<'b, T>, f: F) -> Ref<'b, U>
    where F: FnOnce(&T) -> &U,
          U: ?Sized
```

（原文）
> Make a new Ref for a component of the borrowed data.

（翻译）
> 为借用数据的组件创建一个新的`Ref`。

是的：就像您可以在`Option`上执行`map`操作一样，您也可以在`Ref`上执行`map`。

我确信某个地方的某个人会因为`monad`或其他东西而感到非常兴奋，但我对这些都不关心。另外，我不认为这是一个合适的`monad`，因为没有`None`之类的情况，但我离题了。

这很酷，这对我来说很重要。我需要这个。

```rust
    pub fn peek_front(&self) -> Option<Ref<T>> {
        self.head.as_ref().map(|node| {
            Ref::map(node.borrow(), |node| &node.elem)
        })
    }
```

```txt
> cargo build
```

哦，太好了

让我们通过从堆栈中复制过来的测试来确保它正常工作。我们需要做一些修改来处理`Ref`没有实现比较方法的问题。

```rust
    #[test]
    fn peek() {
        let mut list = List::new();
        assert!(list.peek_front().is_none());
        list.push_front(1); list.push_front(2); list.push_front(3);

        assert_eq!(&*list.peek_front().unwrap(), &3);
    }
```

```txt
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 10 tests
test first::test::basics ... ok
test fourth::test::basics ... ok
test second::test::basics ... ok
test fourth::test::peek ... ok
test second::test::iter_mut ... ok
test second::test::into_iter ... ok
test third::test::basics ... ok
test second::test::peek ... ok
test second::test::iter ... ok
test third::test::iter ... ok

test result: ok. 10 passed; 0 failed; 0 ignored; 0 measured
```

太棒了！

## 5.5 对称的垃圾代码

好吧，让我们把所有操作的组合对称性都搞定。

我们要做的就是进行一些基本的文本替换：

```txt
tail <-> head
next <-> prev
front -> back
```

哦，我们还需要为`peek`方法添加`_mut`变体。

```rust
use std::cell::{Ref, RefCell, RefMut};

//..

    pub fn push_back(&mut self, elem: T) {
        let new_tail = Node::new(elem);
        match self.tail.take() {
            Some(old_tail) => {
                old_tail.borrow_mut().next = Some(new_tail.clone());
                new_tail.borrow_mut().prev = Some(old_tail);
                self.tail = Some(new_tail);
            }
            None => {
                self.head = Some(new_tail.clone());
                self.tail = Some(new_tail);
            }
        }
    }

    pub fn pop_back(&mut self) -> Option<T> {
        self.tail.take().map(|old_tail| {
            match old_tail.borrow_mut().prev.take() {
                Some(new_tail) => {
                    new_tail.borrow_mut().next.take();
                    self.tail = Some(new_tail);
                }
                None => {
                    self.head.take();
                }
            }
            Rc::try_unwrap(old_tail).ok().unwrap().into_inner().elem
        })
    }

    pub fn peek_back(&self) -> Option<Ref<T>> {
        self.tail.as_ref().map(|node| {
            Ref::map(node.borrow(), |node| &node.elem)
        })
    }

    pub fn peek_back_mut(&mut self) -> Option<RefMut<T>> {
        self.tail.as_ref().map(|node| {
            RefMut::map(node.borrow_mut(), |node| &mut node.elem)
        })
    }

    pub fn peek_front_mut(&mut self) -> Option<RefMut<T>> {
        self.head.as_ref().map(|node| {
            RefMut::map(node.borrow_mut(), |node| &mut node.elem)
        })
    }
```

并大量充实我们的测试：

```rust
    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop_front(), None);

        // Populate list
        list.push_front(1);
        list.push_front(2);
        list.push_front(3);

        // Check normal removal
        assert_eq!(list.pop_front(), Some(3));
        assert_eq!(list.pop_front(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push_front(4);
        list.push_front(5);

        // Check normal removal
        assert_eq!(list.pop_front(), Some(5));
        assert_eq!(list.pop_front(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop_front(), Some(1));
        assert_eq!(list.pop_front(), None);

        // ---- back -----

        // Check empty list behaves right
        assert_eq!(list.pop_back(), None);

        // Populate list
        list.push_back(1);
        list.push_back(2);
        list.push_back(3);

        // Check normal removal
        assert_eq!(list.pop_back(), Some(3));
        assert_eq!(list.pop_back(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push_back(4);
        list.push_back(5);

        // Check normal removal
        assert_eq!(list.pop_back(), Some(5));
        assert_eq!(list.pop_back(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop_back(), Some(1));
        assert_eq!(list.pop_back(), None);
    }

    #[test]
    fn peek() {
        let mut list = List::new();
        assert!(list.peek_front().is_none());
        assert!(list.peek_back().is_none());
        assert!(list.peek_front_mut().is_none());
        assert!(list.peek_back_mut().is_none());

        list.push_front(1); list.push_front(2); list.push_front(3);

        assert_eq!(&*list.peek_front().unwrap(), &3);
        assert_eq!(&mut *list.peek_front_mut().unwrap(), &mut 3);
        assert_eq!(&*list.peek_back().unwrap(), &1);
        assert_eq!(&mut *list.peek_back_mut().unwrap(), &mut 1);
    }
```

是否有一些情况我们没有测试？可能。组合空间在这里真的爆炸了。我们的代码至少不是明显的错误。

```txt
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 10 tests
test first::test::basics ... ok
test fourth::test::basics ... ok
test second::test::basics ... ok
test fourth::test::peek ... ok
test second::test::iter ... ok
test third::test::iter ... ok
test second::test::into_iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok

test result: ok. 10 passed; 0 failed; 0 ignored; 0 measured
```

很好。复制粘贴是最好的编程方式。

## 5.6 迭代

让我们尝试处理迭代这个坏男孩。

### `IntoIter`

与之前一样，`IntoIter`是最简单的。只需包装堆栈并调用`pop`：

```rust
pub struct IntoIter<T>(List<T>);

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.pop_front()
    }
}
```

但是我们有一个有趣的新进展。以前我们的列表只有一种“自然”迭代顺序，而`Deque`本质上是双向的。从前到后有什么特别之处？如果有人想在另一个方向迭代怎么办？

Rust实际上对此有一个答案：`DoubleEndedIterator`。`DoubleEndedIterator`继承自`Iterator`（这实际意味着所有`DoubleEndedIterator`都是`Iterator`），并且需要一个新方法：`next_back`。它具有与`next`完全相同的签名，但它应该从另一端产生元素。`DoubleEndedIterator`的语义对我们来说非常方便：迭代器变成了双端队列。您可以从前端和后端使用元素，直到两端汇合，此时迭代器为空。

与`Iterator`和`next`非常相似，事实证明，`next_back`并不是`DoubleEndedIterator`的使用者真正关心的东西。相反，这个接口最好的部分是它公开了`rev`方法，该方法包装了迭代器以生成一个以相反顺序产生元素的新迭代器。这个语义相当直接：在反向迭代器上调用`next`只是调用`next_back`。

无论如何，因为我们已经是一个双端队列，所以提供这个API非常容易：

```rust
impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<T> {
        self.0.pop_back()
    }
}
```

让我们测试一下：

```rust
    #[test]
    fn into_iter() {
        let mut list = List::new();
        list.push_front(1); list.push_front(2); list.push_front(3);

        let mut iter = list.into_iter();
        assert_eq!(iter.next(), Some(3));
        assert_eq!(iter.next_back(), Some(1));
        assert_eq!(iter.next(), Some(2));
        assert_eq!(iter.next_back(), None);
        assert_eq!(iter.next(), None);
    }
```

```txt
#[test]
fn into_iter() {
    let mut list = List::new();
    list.push_front(1); list.push_front(2); list.push_front(3);

    let mut iter = list.into_iter();
    assert_eq!(iter.next(), Some(3));
    assert_eq!(iter.next_back(), Some(1));
    assert_eq!(iter.next(), Some(2));
    assert_eq!(iter.next_back(), None);
    assert_eq!(iter.next(), None);
}
```

很好。

### `Iter`

`Iter`的宽容度会降低一些。我们将不得不再次处理那些可怕的`Ref`问题！由于`Ref`，我们无法像以前那样返回`&Node`。相反，让我们尝试使用`Ref<Node>`：

```rust
pub struct Iter<'a, T>(Option<Ref<'a, Node<T>>>);

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter(self.head.as_ref().map(|head| head.borrow()))
    }
}
```

```txt
> cargo build
```

到目前为止一切顺利。接下来的实现会有点棘手，但我认为它的基本逻辑与旧堆栈`IterMut`相同，但带有额外的`RefCell`疯狂：

```rust
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = Ref<'a, T>;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.take().map(|node_ref| {
            self.0 = node_ref.next.as_ref().map(|head| head.borrow());
            Ref::map(node_ref, |node| &node.elem)
        })
    }
}
```

```txt
> cargo build

error[E0521]: borrowed data escapes outside of closure
   --> src/fourth.rs:155:13
    |
153 |     fn next(&mut self) -> Option<Self::Item> {
    |             --------- `self` is declared here, outside of the closure body
154 |         self.0.take().map(|node_ref| {
155 |             self.0 = node_ref.next.as_ref().map(|head| head.borrow());
    |             ^^^^^^   -------- borrow is only valid in the closure body
    |             |
    |             reference to `node_ref` escapes the closure body here

error[E0505]: cannot move out of `node_ref` because it is borrowed
   --> src/fourth.rs:156:22
    |
153 |     fn next(&mut self) -> Option<Self::Item> {
    |             --------- lifetime `'1` appears in the type of `self`
154 |         self.0.take().map(|node_ref| {
155 |             self.0 = node_ref.next.as_ref().map(|head| head.borrow());
    |             ------   -------- borrow of `node_ref` occurs here
    |             |
    |             assignment requires that `node_ref` is borrowed for `'1`
156 |             Ref::map(node_ref, |node| &node.elem)
    |                      ^^^^^^^^ move out of `node_ref` occurs here
```

糟糕。

`node_ref`的存活时间不够长。与普通引用不同，Rust不允许我们像这样拆分`Ref`。我们从`head.borrow()`中获取的`Ref`只能存活与`node_ref`一样长的时间，但我们最终在`Ref::map`调用中将其丢弃。

我们想要的函数存在，它被称为`map_split()`：

```rust
pub fn map_split<U, V, F>(orig: Ref<'b, T>, f: F) -> (Ref<'b, U>, Ref<'b, V>) where
    F: FnOnce(&T) -> (&U, &V),
    U: ?Sized,
    V: ?Sized,
```

喔，我们来试试吧……

```rust
    fn next(&mut self) -> Option<Self::Item> {
        self.0.take().map(|node_ref| {
            let (next, elem) = Ref::map_split(node_ref, |node| {
                (&node.next, &node.elem)
            });

            self.0 = next.as_ref().map(|head| head.borrow());

            elem
        })
    }
```

```txt
> cargo build
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
error[E0521]: borrowed data escapes outside of closure
   --> src/fourth.rs:159:13
    |
153 |     fn next(&mut self) -> Option<Self::Item> {
    |             --------- `self` is declared here, outside of the closure body
...
159 |             self.0 = next.as_ref().map(|head| head.borrow());
    |             ^^^^^^   ---- borrow is only valid in the closure body
    |             |
    |             reference to `next` escapes the closure body here
```

呃。我们需要再次使用`Ref::Map`来正确获取我们的生命周期。但是`Ref::Map`返回一个`Ref`，而我们需要一个`Option<Ref>`，但我们需要通过`Ref`来在我们的`Option`上执行`map`操作...

长时间凝视远方

？？？？？？

```rust
    fn next(&mut self) -> Option<Self::Item> {
        self.0.take().map(|node_ref| {
            let (next, elem) = Ref::map_split(node_ref, |node| {
                (&node.next, &node.elem)
            });

            self.0 = if next.is_some() {
                Some(Ref::map(next, |next| &**next.as_ref().unwrap()))
            } else {
                None
            };

            elem
        })
    }
```

```txt
error[E0308]: mismatched types
   --> src/fourth.rs:162:22
    |
162 |                 Some(Ref::map(next, |next| &**next.as_ref().unwrap()))
    |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `fourth::Node`, found struct `std::cell::RefCell`
    |
    = note: expected type `std::cell::Ref<'_, fourth::Node<_>>`
               found type `std::cell::Ref<'_, std::cell::RefCell<fourth::Node<_>>>`
```

哦。对了。有多个`RefCell`。我们越深入列表，每个`RefCell`下的嵌套就越多。我们需要维护一个`Ref`堆栈来表示我们持有的所有未偿还贷款，因为如果我们停止查看某个元素，我们需要减少它之前的每个`RefCell`的借入计数。................

我认为我们在这里无能为力。这是死路一条。让我们尝试摆脱多重`RefCell`。

我们的`Rc`怎么样。谁说我们甚至需要存储引用？为什么我们不能克隆整个`Rc`以获得一个不错的拥有句柄到列表中间？

```rust
pub struct Iter<T>(Option<Rc<Node<T>>>);

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter(self.head.as_ref().map(|head| head.clone()))
    }
}

impl<T> Iterator for Iter<T> {
    type Item = ???
    ...
}
```

呃...等等我们现在应该返回什么？`&T`？`Ref<T>`？

不，这些都行不通……我们的`Iter`不再有生命周期了！`&T`和`Ref<T>`都要求我们在进入下一个之前预先声明一些生命周期。但是我们设法从`Rc`中得到的任何东西都会借用`Iterator`……脑子……疼……啊啊啊啊啊啊

也许我们可以……使用`map`……`Rc`……来得到一个`Rc<T>`？有这个东西吗？`Rc`的文档似乎没有这样的东西。实际上有人制作了一个可以让你这样做的[crate](https://crates.io/crates/owning_ref)。

但是等等，即使我们这样做，我们也会遇到更大的问题：可怕的迭代器失效幽灵。以前我们完全不受迭代器失效的影响，因为`Iter`借用了列表，使其完全不可变。但是，如果我们的`Iter`产生`Rc`，它们根本不会借用列表！这意味着人们可以在持有指向列表的指针时开始调用列表上的`push`和`pop`！

天啊，那会怎么样？！

嗯，押入操作其实没问题。我们已经看到了列表的一些子范围，列表会超出我们的视线。没什么大不了的。

但是弹出则是另一回事。如果他们弹出我们范围之外的元素，应该还是没问题的。我们看不到那些节点，所以什么也不会发生。但是，如果他们试图弹出我们指向的节点……一切都会崩溃！特别是当他们去解开`try_unwrap`的结果时，它实际上会失败，整个程序都会崩溃。

这实际上很酷。我们可以将大量内部拥有的指针放入列表中并同时对其进行改变，它会一直工作，直到它们试图删除我们指向的节点。即使那时我们没有得到悬垂指针或任何东西，程序也会确定性地恐慌！

但在映射`Rc`之上还要处理迭代器失效似乎……很糟糕。`Rc<RefCell>`真的最终让我们失望了。有趣的是，我们经历了持久堆栈情况的反转。持久堆栈很难收回数据的所有权，但时时刻刻都能获得引用，我们的列表在获得所有权方面没有问题，但在借出我们的引用方面却非常困难。

虽然公平地说，我们的大部分困难都围绕着想要隐藏实现细节并拥有一个不错的API。如果我们可以将`Node`设为`pub`并传递到各处，我们可以做所有事情。

哎呀，我们可以创建多个并发的`IterMut`，这些`IterMut`经过运行时检查，不可更改，以访问同一元素！

实际上，这种设计更适合永远不会向API消费者公开的内部数据结构。**内部可变性非常适合编写安全的应用程序，但不太适合安全的库。**

无论如何，这就是我**放弃**`Iter`和`IterMut`的原因。我们可以做它们，但是呃。

## 5.7 最终代码

好吧，这就是在Rust中实现100%安全的双向链表。实现起来非常困难，会泄露实现细节，并且不支持几个基本操作，所以几乎没法用于实际开发。

但是，它确实存在。

哦，我想它还充斥着大量“不必要的”运行时检查，以确保`Rc`和`RefCell`之间的正确性。我将不必要的放在引号中，因为它们实际上是保证整个过程真正安全所必需的。我们遇到了一些地方，这些检查实际上是必要的。双向链表有一个非常复杂的别名和所有权故事！

不过，这是我们可以做的事情。特别是如果我们不关心向消费者公开内部数据结构的话。

从现在开始，我们将专注于这枚硬币的另一面：通过使我们的实现不安全来重新获得所有控制权。

```rust
use std::rc::Rc;
use std::cell::{Ref, RefMut, RefCell};

pub struct List<T> {
    head: Link<T>,
    tail: Link<T>,
}

type Link<T> = Option<Rc<RefCell<Node<T>>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
    prev: Link<T>,
}


impl<T> Node<T> {
    fn new(elem: T) -> Rc<RefCell<Self>> {
        Rc::new(RefCell::new(Node {
            elem: elem,
            prev: None,
            next: None,
        }))
    }
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push_front(&mut self, elem: T) {
        let new_head = Node::new(elem);
        match self.head.take() {
            Some(old_head) => {
                old_head.borrow_mut().prev = Some(new_head.clone());
                new_head.borrow_mut().next = Some(old_head);
                self.head = Some(new_head);
            }
            None => {
                self.tail = Some(new_head.clone());
                self.head = Some(new_head);
            }
        }
    }

    pub fn push_back(&mut self, elem: T) {
        let new_tail = Node::new(elem);
        match self.tail.take() {
            Some(old_tail) => {
                old_tail.borrow_mut().next = Some(new_tail.clone());
                new_tail.borrow_mut().prev = Some(old_tail);
                self.tail = Some(new_tail);
            }
            None => {
                self.head = Some(new_tail.clone());
                self.tail = Some(new_tail);
            }
        }
    }

    pub fn pop_back(&mut self) -> Option<T> {
        self.tail.take().map(|old_tail| {
            match old_tail.borrow_mut().prev.take() {
                Some(new_tail) => {
                    new_tail.borrow_mut().next.take();
                    self.tail = Some(new_tail);
                }
                None => {
                    self.head.take();
                }
            }
            Rc::try_unwrap(old_tail).ok().unwrap().into_inner().elem
        })
    }

    pub fn pop_front(&mut self) -> Option<T> {
        self.head.take().map(|old_head| {
            match old_head.borrow_mut().next.take() {
                Some(new_head) => {
                    new_head.borrow_mut().prev.take();
                    self.head = Some(new_head);
                }
                None => {
                    self.tail.take();
                }
            }
            Rc::try_unwrap(old_head).ok().unwrap().into_inner().elem
        })
    }

    pub fn peek_front(&self) -> Option<Ref<T>> {
        self.head.as_ref().map(|node| {
            Ref::map(node.borrow(), |node| &node.elem)
        })
    }

    pub fn peek_back(&self) -> Option<Ref<T>> {
        self.tail.as_ref().map(|node| {
            Ref::map(node.borrow(), |node| &node.elem)
        })
    }

    pub fn peek_back_mut(&mut self) -> Option<RefMut<T>> {
        self.tail.as_ref().map(|node| {
            RefMut::map(node.borrow_mut(), |node| &mut node.elem)
        })
    }

    pub fn peek_front_mut(&mut self) -> Option<RefMut<T>> {
        self.head.as_ref().map(|node| {
            RefMut::map(node.borrow_mut(), |node| &mut node.elem)
        })
    }

    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }
}

impl<T> Drop for List<T> {
    fn drop(&mut self) {
        while self.pop_front().is_some() {}
    }
}

pub struct IntoIter<T>(List<T>);

impl<T> Iterator for IntoIter<T> {
    type Item = T;

    fn next(&mut self) -> Option<T> {
        self.0.pop_front()
    }
}

impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<T> {
        self.0.pop_back()
    }
}

#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop_front(), None);

        // Populate list
        list.push_front(1);
        list.push_front(2);
        list.push_front(3);

        // Check normal removal
        assert_eq!(list.pop_front(), Some(3));
        assert_eq!(list.pop_front(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push_front(4);
        list.push_front(5);

        // Check normal removal
        assert_eq!(list.pop_front(), Some(5));
        assert_eq!(list.pop_front(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop_front(), Some(1));
        assert_eq!(list.pop_front(), None);

        // ---- back -----

        // Check empty list behaves right
        assert_eq!(list.pop_back(), None);

        // Populate list
        list.push_back(1);
        list.push_back(2);
        list.push_back(3);

        // Check normal removal
        assert_eq!(list.pop_back(), Some(3));
        assert_eq!(list.pop_back(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push_back(4);
        list.push_back(5);

        // Check normal removal
        assert_eq!(list.pop_back(), Some(5));
        assert_eq!(list.pop_back(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop_back(), Some(1));
        assert_eq!(list.pop_back(), None);
    }

    #[test]
    fn peek() {
        let mut list = List::new();
        assert!(list.peek_front().is_none());
        assert!(list.peek_back().is_none());
        assert!(list.peek_front_mut().is_none());
        assert!(list.peek_back_mut().is_none());

        list.push_front(1); list.push_front(2); list.push_front(3);

        assert_eq!(&*list.peek_front().unwrap(), &3);
        assert_eq!(&mut *list.peek_front_mut().unwrap(), &mut 3);
        assert_eq!(&*list.peek_back().unwrap(), &1);
        assert_eq!(&mut *list.peek_back_mut().unwrap(), &mut 1);
    }

    #[test]
    fn into_iter() {
        let mut list = List::new();
        list.push_front(1); list.push_front(2); list.push_front(3);

        let mut iter = list.into_iter();
        assert_eq!(iter.next(), Some(3));
        assert_eq!(iter.next_back(), Some(1));
        assert_eq!(iter.next(), Some(2));
        assert_eq!(iter.next_back(), None);
        assert_eq!(iter.next(), None);
    }
}
```
