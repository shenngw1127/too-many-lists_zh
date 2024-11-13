# 2. 一个糟糕的单链栈

这将是迄今为止最长的一篇，因为我们需要介绍Rust的所有内容，并且会“以困难的方式”建立一些东西以更好地理解该语言。

我们会将第一个列表放在`src/first.rs`文件中。我们需要告诉Rust，`first.rs`是我们的库使用的东西。我们只需要将其放在`src/lib.rs`（Cargo为我们创建的文件）的顶部：

```rust
// in lib.rs
pub mod first;
```

## 2.1 基本数据布局

好吧，那么什么是链表呢？基本上，它是堆上的一堆数据（内核人员请保持安静！），它们按顺序指向彼此。链表是一些过程式程序员不应该触碰的东西，也是函数式程序员用于一切的东西。因此，我们应该向函数式程序员询问链表的定义，这似乎是公平的。他们可能会给你类似以下定义的东西：

```clojure
List a = Empty | Elem a (List a)
```

大致意思为“列表要么是空的，要么是元素后紧跟一个列表”。这是一个递归定义，表示为和类型（`sum type`），它是“可以具有不同值的类型，这些值可能是不同类型的类型”的别称。Rust将和类型称为枚举（`enum`）！如果您来自类似C的语言，那么这正是您熟悉和喜爱的枚举（其实，和类型更象C中的联合（`union`），不过这只是Rust枚举的另一个方面），但速度更快。所以让我们将这个函数定义转译到Rust中！

现在我们将避免使用泛型以保持简单。我们只支持存储有符号的32位整数：

```rust
// in first.rs

// pub says we want people outside this module to be able to use List
pub enum List {
    Empty,
    Elem(i32, List),
}
```

呼，我忙不过来了。我们直接编译一下吧，你会看到如下错误：

```txt
> cargo build

error[E0072]: recursive type `first::List` has infinite size
 --> src/first.rs:4:1
  |
4 | pub enum List {
  | ^^^^^^^^^^^^^ recursive type has infinite size
5 |     Empty,
6 |     Elem(i32, List),
  |               ---- recursive without indirection
  |
  = help: insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to make `first::List` representable
```

好吧。我不知道你是怎么想的，但我确实觉得函数式编程社区背叛了我。

如果我们真的检查一下错误消息（在我们克服了整个背叛事件之后），我们可以看到`rustc`实际上告诉我们如何解决这个问题：

> insert indirection (e.g., a Box, Rc, or &) at some point to make first::List representable

好吧，`Box`。那是什么？让我们谷歌一下`rust box`……

> [std::boxed::Box - Rust](https://doc.rust-lang.org/std/boxed/struct.Box.html)

在这里...

> `pub struct Box<T>(_);`
>
> A pointer type for heap allocation. See the [module-level documentation](https://doc.rust-lang.org/std/boxed/) for more.

点击链接

（原文）
> `Box<T>`, casually referred to as a 'box', provides the simplest form of heap allocation in Rust. Boxes provide ownership for this allocation, and drop their contents when they go out of scope.
>
> Examples
>
> Creating a box:
>
> `let x = Box::new(5);`
>
> Creating a recursive data structure:

（翻译）
> `Box<T>`，也被称为“盒子”，提供了Rust中最简单的堆分配形式。盒子提供此分配的所有权，并在超出范围时删除其内容。
>
> 示例
>
> 创建一个盒子：
>
> `let x = Box::new(5);`
>
> 创建一个递归的结构：

```rust
#[derive(Debug)]
enum List<T> {
    Cons(T, Box<List<T>>),
    Nil,
}
```

```rust
fn main() {
    let list: List<i32> = List::Cons(1, Box::new(List::Cons(2, Box::new(List::Nil))));
    println!("{:?}", list);
}
```

（原文）
> This will print `Cons(1, Box(Cons(2, Box(Nil))))`.
>
> Recursive structures must be boxed, because if the definition of `Cons` looked like this:
>
> `Cons(T, List<T>)`,
>
> It wouldn't work. This is because the size of a List depends on how many elements are in the list, and so we don't know how much memory to allocate for a Cons. By introducing a Box, which has a defined size, we know how big Cons needs to be.

（翻译）
> 这将打印出`Cons(1, Box(Cons(2, Box(Nil))))`.
>
> 递归的结构必须用盒子封装，因为如果`Cons`的定义如下：
>
> `Cons(T, List<T>)`,
>
> 这行不通，无法通过编译。这是因为`List`的大小取决于列表中有多少个元素，所以我们不知道要为`Cons`分配多少内存。通过引入具有定义大小的`Box`，我们知道`Cons`需要多大。

哇，呃。这可能是我见过的最相关、最有帮助的文档。文档中的第一件事就是我们要写的内容、为什么它不起作用以及如何修复它。

哎呀，文档很重要。

好的，让我们这样修改代码：

```rust
pub enum List {
    Empty,
    Elem(i32, Box<List>),
}
```

```txt
> cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

嘿，构建成功了！

...但这实际上是`List`的一个**非常愚蠢**的定义，它不应该这样被定义，原因有几个。

考虑一个包含两个元素（三个节点）的列表：

```txt
[] = 在栈（`Stack`）上分配内存
() = 在堆（`Heap`）上分配内存

[Elem A, ptr] -> (Elem B, ptr) -> (Empty, *junk*)
```

有两个关键问题：

- 分配了一个节点（最右侧的节点3），它只是说“我实际上不是一个节点”
- 其中一个节点（最左侧的节点1）根本不是在堆上分配的（因为`i32`可以在栈上分配）。

从表面上看，这两者似乎相互抵消。我们在堆上分配了一个额外的节点，但我们的一个节点根本不需要堆分配。但是，请考虑我们的列表的以下潜在布局：

```txt
[ptr] -> (Elem A, ptr) -> (Elem B, *null*)
```

在此布局中，我们现在无条件地在堆上分配节点。关键区别在于，第一个布局中存在垃圾（junk）而这个没有。这个垃圾是什么？要理解这一点，我们需要看看枚举在内存中的布局方式。

一般来说，如果我们有一个像这样的枚举：

```rust
enum Foo {
    D1(T1),
    D2(T2),
    ...
    Dn(Tn),
}
```

`Foo`需要存储一些整数来指示它代表枚举的哪个变体（`D1`、`D2`、.. `Dn`）。这是枚举的标签。它还需要足够的空间来存储`T1`、`T2`、.. `Tn`中最大的一个（加上一些额外的空间来满足对齐要求）。

这样最大的问题是，尽管`Empty`是单个信息，但它必然会占用足够的空间来容纳指针和元素，因为它必须随时准备成为`Elem`。因此，第一个布局堆分配了一个充满垃圾的额外元素，比第二个布局占用的空间要多一点。

令人惊讶的是，我们的一个节点根本没有分配，这比总是分配它更糟糕。这是因为它给我们带来了不均匀的节点布局。这对押入和弹出节点没有太大的影响，但对拆分和合并列表有影响。

如果我们要在两种布局中拆分列表：

```txt
布局1：

[Elem A, ptr] -> (Elem B, ptr) -> (Elem C, ptr) -> (Empty *junk*)

从元素C处拆分：

[Elem A, ptr] -> (Elem B, ptr) -> (Empty *junk*)
[Elem C, ptr] -> (Empty *junk*)
```

```txt
布局2：

[ptr] -> (Elem A, ptr) -> (Elem B, ptr) -> (Elem C, *null*)

从元素C处拆分：

[ptr] -> (Elem A, ptr) -> (Elem B, *null*)
[ptr] -> (Elem C, *null*)
```

布局2的拆分仅涉及将元素B的指针复制到堆栈并将旧值清零。布局1最终会执行相同的操作，但还必须将元素C从堆复制到栈。合并是相反的相同过程。

链表的几个优点之一是，您可以在节点本身中构造元素，然后自由地在列表中移动它，而无需移动它。您只需摆弄指针，东西就会被“移动”。布局1会破坏此属性。

好吧，我有理由相信布局1很糟糕。我们如何重写我们的列表？好吧，我们可以这样做：

```rust
pub enum List {
    Empty,
    ElemThenEmpty(i32),
    ElemThenNotEmpty(i32, Box<List>),
}
```

这对你来说似乎是一个更糟糕的想法。不要这样写！最值得注意的是，这确实使我们的逻辑变得复杂，因为现在有一个完全无效的状态：`ElemThenNotEmpty(0, Box(Empty))`。并且它仍然受到非统一分配元素的影响（有可能部分元素的内存在堆上，部分元素的内存在栈上）。

然而，它确实有一个有趣的属性：它完全避免为`Empty`分配内存的情况，将堆分配的总数减少了1。不幸的是，这样做会浪费更多的空间！这是因为之前的布局利用了**空指针优化**。

我们之前看到，每个枚举都必须存储一个标签，以指定其位代表枚举的哪个变体。但是，如果我们有一种特殊的枚举：

```rust
enum Foo {
    A,
    B(ContainsANonNullPtr),
}
```

空指针优化开始生效，从而消除了标记所需的空间。编译时的空指针优化会这样做：如果变体是A，则整个枚举将设置为全`0`。否则，变体是B。这是可行的，因为B永远不会全为`0`，因为它包含一个非空指针。太棒了！

您能想到其他可以进行这种优化的枚举和类型吗？实际上有很多！这就是为什么Rust完全不指定枚举布局的原因。Rust会为我们做一些更复杂的枚举布局优化，但空指针优化绝对是最重要的！这意味着`&`、`&mut`、`Box`、`Rc`、`Arc`、`Vec`和Rust中的其他几种重要类型在被放入`Option`时没有任何开销！（我们会在适当的时候介绍其中的大部分内容。）

那么，我们如何避免多余的垃圾（junk），统一分配，并获得理想的空指针优化呢？我们需要更好地将拥有一个元素的想法与分配另一个列表的想法区分开来。要做到这一点，我们必须更多地象C语言一样思考考虑： 对，就是用结构体（`struct`）！

枚举允许我们声明一个可以包含多个值之一的类型，而结构体允许我们声明一个可以同时包含多个值的类型。让我们将列表分为两种类型：列表和节点。

与之前一样，`List`要么为空，要么有一个元素后跟另一个`List`。通过用一个完全独立的类型表示“有一个元素后跟另一个`List`”的情况，我们可以将`Box`提升到更理想的位置：

```rust
struct Node {
    elem: i32,
    next: List,
}

pub enum List {
    Empty,
    More(Box<Node>),
}
```

让我们检查一下我们的优先级：

- 列表尾部永远不会分配额外的垃圾：检查ok！
- 枚举是完美的可以被空指针优化形式：检查ok！
- 所有元素的内存都是统一一种形式分配的（使用`Box`封装，都在堆上）：检查ok！

好吧！我们实际上只是构建了我们用来演示我们的第一个布局（如官方Rust文档所建议的），但它还是个有问题的布局，错误如下。

```txt
> cargo build

warning: private type `first::Node` in public interface (error E0446)
 --> src/first.rs:8:10
  |
8 |     More(Box<Node>),
  |          ^^^^^^^^^
  |
  = note: #[warn(private_in_public)] on by default
  = warning: this was previously accepted by the compiler but
    is being phased out; it will become a hard error in a future release!
```

嗯，Rust又对我们生气了。我们将`List`标记为`pub`（因为我们希望人们能够使用它），但没有将`Node`标记为`pub`。问题是枚举的内部是完全公开的，不应该包含私有类型。我们可以将`Node`的所有内容都设为完全公开即`pub`，但通常在Rust中我们倾向于将实现细节保持私有。更好的修改方式是将`List`设为结构体，外包一层，这样我们就可以隐藏实现细节了：

```rust
pub struct List {
    head: Link,
}

enum Link {
    Empty,
    More(Box<Node>),
}

struct Node {
    elem: i32,
    next: Link,
}
```

因为`List`是具有单个字段的结构，所以其大小与该字段相同。零成本抽象！

```txt
> cargo build

warning: field is never used: `head`
 --> src/first.rs:2:5
  |
2 |     head: Link,
  |     ^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: variant is never constructed: `Empty`
 --> src/first.rs:6:5
  |
6 |     Empty,
  |     ^^^^^

warning: variant is never constructed: `More`
 --> src/first.rs:7:5
  |
7 |     More(Box<Node>),
  |     ^^^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/first.rs:11:5
   |
11 |     elem: i32,
   |     ^^^^^^^^^

warning: field is never used: `next`
  --> src/first.rs:12:5
   |
12 |     next: Link,
   |     ^^^^^^^^^^
```

好吧，编译通过了！但Rust非常生气，因为据它所知，我们编写的所有内容都是完全无用的：我们从没使用过`head`，使用我们库的人也不能使用它，因为它是私有的。从传递性上讲，这意味着`Link`和`Node`也是无用的。所以让我们解决这个问题！让我们为我们的`List`实现一些代码！

## 2.2 `new`方法

为了将实际代码与类型关联起来，我们使用`impl`块：

```rust
impl List {
    // TODO, make code happen
}
```

现在我们只需要弄清楚如何实际编写代码。在Rust中，我们声明一个函数，如下所示：

```rust
fn foo(arg1: Type1, arg2: Type2) -> ReturnType {
    // body
}
```

我们首先需要一种构造列表的方法。由于我们隐藏了实现细节，因此我们需要将其作为函数提供。在Rust中，通常的做法是提供一个静态方法，它只是`impl`内部的一个普通函数：

```rust
impl List {
    pub fn new() -> Self {
        List { head: Link::Empty }
    }
}
```

关于这里的几点说明：

- `Self`是“我在`impl`旁边顶部写的那个类型”的别名。非常好，不重复自己！
- 我们创建结构体实例的方式与声明它的方式大致相同，只是我们不提供字段的类型，而是用值初始化它们。
- 我们使用`::`（命名空间运算符）来引用枚举的变体。
- 函数的最后一个表达式是隐式返回的。这使简单的函数更简洁一些。您仍然可以像其他类似C的语言一样使用`return`提前返回。

## 2.3 所有权101

现在我们可以构造一个`List`了，如果能用它做一些事情就好了。我们用“普通”（非静态）方法来实现这一点。在Rust中，方法是函数的一个特例，因为它的`self`参数没有声明类型：

```rust
fn foo(self, arg2: Type2) -> ReturnType {
    // body
}
```

`self`有3种主要形式：`self`、`&mut self`和`&self`。这3种形式代表了Rust中所有权的3种主要形式：

- `self` - 值
- `&mut self` - 可变引用
- `&self` - 共享引用

值代表真正的所有权。您可以对值做任何您想做的事情：移动它、销毁它、改变它或通过引用借出它。当您通过值传递某个值时，它会被移动到新位置。新位置现在拥有该值，而旧位置不再能访问它。出于这个原因，大多数方法都不需要`self`——如果尝试使用`List`，然后使其消失，那就太糟糕了！

可变引用表示对您不拥有的值的临时独占访问。您可以对具有可变引用的值执行任何操作，只要您在完成后将其保持在有效状态即可（否则对所有者来说很不礼貌！）。这意味着您实际上可以完全覆盖该值。一个非常有用的特殊情况是将一个值换成另一个值，我们将经常使用它。您**不能**用`&mut`做的唯一事情是将值移出而不进行替换。`&mut self`非常适合想要改变自身的方法。

共享引用表示对您不拥有的值的临时共享访问。由于您拥有共享访问权限，因此通常不允许您改变任何内容。可以将`&`视为将值展示在博物馆中。`&`非常适合只想观察自身的方法。

稍后我们将看到，在某些情况下可以绕过有关改变的规则。这就是为什么共享引用不称为不可变引用的原因。实际上，可变引用可以称为唯一引用，但我们发现将所有权与可变性联系起来在`99%`的情况下都能给出正确的直觉。

## 2.4 `push`方法

因此，让我们编写将值押入（`push`）到列表的过程。押入会改变列表，因此我们需要采用`&mut self`。我们还需要一个`i32`类型的参数作为要押入的值：

```rust
impl List {
    pub fn push(&mut self, elem: i32) {
        // TODO
    }
}
```

首先，我们需要创建一个节点来存储我们的元素：

```rust
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: ?????
        };
    }
```

`next`应该被赋予一个什么值呢？好吧，整个旧列表！我们可以...就象下面这样直接写吗？

```rust
impl List {
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: self.head,
        };
    }
}
```

```txt
> cargo build
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:19:19
   |
19 |             next: self.head,
   |                   ^^^^^^^^^ cannot move out of borrowed content
```

不。Rust告诉我们正确的事情，但它到底意味着什么，或者该怎么做，显然并不明显：

（原文）
> cannot move out of borrowed content

（翻译）
> 不能移出借用的内容

我们试图将`self.head`字段移出到`next`，但Rust不希望我们这样做。当我们结束借用并“归还”给其合法所有者时，这将使`self`仅部分初始化。正如我们之前所说，这是您不能用`&mut`做的事情：这会非常粗鲁，而Rust非常有礼貌（这也会非常危险，但这肯定不是它关心的原因）。

如果我们放回一些东西会怎么样？即我们正在创建的节点：

```rust
    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head,
        });

        self.head = Link::More(new_node);
    }
```

```txt
> cargo build
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:19:19
   |
19 |             next: self.head,
   |                   ^^^^^^^^^ cannot move out of borrowed content
```

不行。原则上，Rust可以接受这一点，但它不会接受（出于各种原因——最严重的是[异常安全](https://doc.rust-lang.org/nightly/nomicon/exception-safety.html)）。我们需要某种方法来获取`head`，而Rust不会注意到它已经消失了。为了寻求建议，我们向臭名昭著的Rust黑客印第安的琼斯（著名电影人物）求助：

啊，是的，印第建议使用[`mem::replace`](https://doc.rust-lang.org/std/mem/fn.replace.html)操作。这个非常有用的函数让我们可以通过用另一个值替换借用的值来窃取借用的值，然后返回原来的值。

（原文）
> `pub fn replace<T>(dest: &mut T, src: T) -> T`
>
> Moves src into the referenced dest, returning the previous dest value.
>
> Neither value is dropped.
>
> Examples
>
> ```rust
> use std::mem;
> 
> let mut v: Vec<i32> = vec![1, 2];
> 
> let old_v = mem::replace(&mut v, vec![3, 4, 5]);
> assert_eq!(vec![1, 2], old_v);
> assert_eq!(vec![3, 4, 5], v);
> ```

（翻译）
> `pub fn replace<T>(dest: &mut T, src: T) -> T`
>
> 将`src`移至引用的`dest`，返回先前的`dest`值。
>
> 这两个值都不会被丢弃。
>
> 示例
>
> ```rust
> use std::mem;
> 
> let mut v: Vec<i32> = vec![1, 2];
> 
> let old_v = mem::replace(&mut v, vec![3, 4, 5]);
> assert_eq!(vec![1, 2], old_v);
> assert_eq!(vec![3, 4, 5], v);
> ```

我们只需在文件顶部导入 `std::mem`，这样`mem`就处于本地作用域中：

```rust
use std::mem;
```

并适当地使用它：

```rust
    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: mem::replace(&mut self.head, Link::Empty),
        });

        self.head = Link::More(new_node);
    }
```

在这里，我们暂时用`Link::Empty`替换`self.head`，然后再用列表的新`head`替换它。我不会撒谎：这是一件非常不幸的事情。遗憾的是，我们必须这样做（目前）。

但是嘿，押入就完成了！可能吧。老实说，我们应该测试一下。现在最简单的方法可能是编写弹出（`pop`），并确保它产生正确的结果。

## 2.5 `pop`方法

与押入（`push`）操作类似，弹出（`pop`）操作也会改变列表。与`push`不同，我们实际上希望返回一些东西。但`pop`还必须处理一个棘手的极端情况：如果列表为空怎么办？为了表示这种情况，我们使用了可信赖的`Option`类型：

```rust
    pub fn pop(&mut self) -> Option<i32> {
        // TODO
    }
```

`Option<T>`是一个枚举，表示可能存在的值。它可以是`Some(T)`或`None`。我们可以像为`Link`所做的那样为其创建自己的枚举，但我们希望用户能够理解我们的返回类型到底是什么，而`Option`无处不在，每个人都知道它。事实上，它是如此基础，以至于它被隐式导入到每个文件的作用域中，以及它的变体`Some`和`None`（所以我们不必写`Option::None`）。

`Option<T>`上的尖头括号表示`Option`实际上是`T`上的泛型。这意味着您可以为任何类型创建`Option`！ 那么呃，我们有这个`Link`东西，我们如何确定它是空的还是有更多？使用`match`进行模式匹配是广泛使用方式！

```rust
    pub fn pop(&mut self) -> Option<i32> {
        match self.head {
            Link::Empty => {
                // TODO
            }
            Link::More(node) => {
                // TODO
            }
        };
    }
```

```txt
> cargo build

error[E0308]: mismatched types
  --> src/first.rs:27:30
   |
27 |     pub fn pop(&mut self) -> Option<i32> {
   |            ---               ^^^^^^^^^^^ expected enum `std::option::Option`, found ()
   |            |
   |            this function's body doesn't return
   |
   = note: expected type `std::option::Option<i32>`
              found type `()`
```

哎呀，编译报错了，`pop`方法必须返回一个值，而我们还没有这样做。我们可以返回`None`，但在这种情况下，返回`unimplemented!()`可能是一个更好的主意，以表明我们尚未完成该函数的实现。`unimplemented!()`是一个宏（`!`表示一个宏），当我们到达它时，它会使程序崩溃（～以受控的方式使其崩溃）。

```rust
    pub fn pop(&mut self) -> Option<i32> {
        match self.head {
            Link::Empty => {
                // TODO
            }
            Link::More(node) => {
                // TODO
            }
        };
        unimplemented!()
    }
```

无条件恐慌是[发散函数](https://doc.rust-lang.org/nightly/book/ch19-04-advanced-types.html#the-never-type-that-never-returns)的一个例子。发散函数永远不会返回给调用者，因此它们可用于需要任何类型的值的地方。这里，`unimplemented!()`产生了无条件恐慌（`unconditional panics`），被用来代替类型为`Option<T>`的值。

还要注意，我们不需要在程序中写`return`。函数中的最后一个表达式（基本上是一行）隐式地是它的返回值。这让我们可以更简洁地表达非常简单的事情。当然，您也可以像任何其他类似C的语言一样，始终使用`return`显式地提前返回。

```txt
> cargo build

error[E0507]: cannot move out of borrowed content
  --> src/first.rs:28:15
   |
28 |         match self.head {
   |               ^^^^^^^^^
   |               |
   |               cannot move out of borrowed content
   |               help: consider borrowing here: `&self.head`
...
32 |             Link::More(node) => {
   |                        ---- data moved here
   |
note: move occurs because `node` has type `std::boxed::Box<first::Node>`, which does not implement the `Copy` trait
  --> src/first.rs:32:24
   |
32 |             Link::More(node) => {
   |                        ^^^^
```

来吧，Rust，别再纠缠我们了！和往常一样，Rust对我们非常生气。值得庆幸的是，这次它还给了我们全部信息！默认情况下，模式匹配会尝试将其内容移动到新分支中，但我们不能这样做，因为这里我们没有按值拥有自身。

（原文）
> help: consider borrowing here: `&self.head`

（翻译）
> 帮助：考虑借用这里：`&self.head`

Rust说我们应该添加对匹配的引用（即增加`&`操作符）来解决这个问题。🤷‍♀️让我们尝试一下：

```rust
    pub fn pop(&mut self) -> Option<i32> {
        match &self.head {
            Link::Empty => {
                // TODO
            }
            Link::More(node) => {
                // TODO
            }
        };
        unimplemented!()
    }
```

```txt
> cargo build

warning: unused variable: `node`
  --> src/first.rs:32:24
   |
32 |             Link::More(node) => {
   |                        ^^^^ help: consider prefixing with an underscore: `_node`
   |
   = note: #[warn(unused_variables)] on by default

warning: field is never used: `elem`
  --> src/first.rs:13:5
   |
13 |     elem: i32,
   |     ^^^^^^^^^
   |
   = note: #[warn(dead_code)] on by default

warning: field is never used: `next`
  --> src/first.rs:14:5
   |
14 |     next: Link,
   |     ^^^^^^^^^^
```

万岁，又编译成功了！现在让我们弄清楚这个逻辑。我们需要返回一个`Option`，所以让我们为它创建一个变量。在`Empty`的情况下，我们需要返回`None`。在`More`的情况下，我们需要返回`Some(i32)`，并更改列表的头部。那么，让我们试着这样做吧？

```rust
    pub fn pop(&mut self) -> Option<i32> {
        let result;
        match &self.head {
            Link::Empty => {
                result = None;
            }
            Link::More(node) => {
                result = Some(node.elem);
                self.head = node.next;
            }
        };
        result
    }
```

```txt
> cargo build
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:35:29
   |
35 |                 self.head = node.next;
   |                             ^^^^^^^^^ cannot move out of borrowed content
```

head

node.next

当我们拥有的只是对节点的共享引用时，我们正试图移出节点。

我们应该退后一步，想想我们试图做什么。我们想要：

- 检查列表是否为空。
- 如果为空，则返回`None`
- 如果不为空
  - 删除列表的`head`
  - 删除其`elem`
  - 用节点的`next`替换列表的`head`
  - 返回`Some(elem)`

关键的问题是我们想要删除一些东西，这意味着我们想要通过值获取列表的头部。我们当然不能通过通过`&self.head`获得的共享引用来做到这一点。我们还“只”拥有对`self`的可变引用，因此我们移动东西的唯一方法是替换它。看起来我们又要跳`Empty`舞了！（还是调用`mem::replace`，并传递`Link::Empty`）

让我们试试：

```rust
    pub fn pop(&mut self) -> Option<i32> {
        let result;
        match mem::replace(&mut self.head, Link::Empty) {
            Link::Empty => {
                result = None;
            }
            Link::More(node) => {
                result = Some(node.elem);
                self.head = node.next;
            }
        };
        result
    }
```

```txt
cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

哦，上帝！

编译通过没有任何警告！！！！！

实际上，我要在这里应用我自己的个人`lint`程序检测规则：我们让这个结果值返回，但实际上我们根本不需要这样做！就像函数求值为它的最后一个表达式一样，每个块也求值为它的最后一个表达式。通常我们用分号来抑制这种行为，这会使块求值结果为空元组`()`。这实际上是没有声明返回值的函数（就象`push`方法）返回的值。（实际上，好的IDE也会推荐你这么做！）

因此，我们可以将`pop`写为：

```rust
    pub fn pop(&mut self) -> Option<i32> {
        match mem::replace(&mut self.head, Link::Empty) {
            Link::Empty => None,
            Link::More(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
```

这更简洁、更符合语法。请注意，`Link::Empty`匹配臂完全没有使用花括号，因为我们只需要计算一个表达式。这只是简单情况下的简写。

```txt
cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

很好，仍然有效！

## 2.6 测试

好了，我们已经编写了`push`和`pop`方法，现在我们可以实际测试我们实现的栈了！Rust和Cargo将测试作为一流功能来支持，因此这将非常简单。我们所要做的就是编写一个函数，并用`#[test]`对其进行标注。

通常，我们会尝试将测试放在Rust社区中正在测试的代码旁边。但是，我们通常会为测试创建一个新的命名空间，以避免与“真实”代码冲突。就像我们使用`mod`来指定`first.rs`应该包含在`lib.rs`中一样，我们可以使用`mod`基本上以内联方式创建一个新文件（内联是指只把文件内容嵌入到当前文件，实际并没有创建新的文件）：

```rust
// in first.rs

mod test {
    #[test]
    fn basics() {
        // TODO
    }
}
```

然后我们用`cargo test`来调用它。

```txt
> cargo test
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
    Finished dev [unoptimized + debuginfo] target(s) in 1.00s
     Running /Users/ADesires/dev/lists/target/debug/deps/lists-86544f1d97438f1f

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
; 0 filtered out
```

耶，我们写的啥也没做的测试通过了！让我们改写它让它做点事。我们将使用`assert_eq!`宏来实现这一点。这不是什么特殊的测试魔法。它所做的就是比较您给它的两个参数，如果它们不匹配，程序就会崩溃。是的，通过崩溃来向测试工具表明测试失败！

```rust
mod test {
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
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push(4);
        list.push(5);

        // Check normal removal
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), None);
    }
}
```

```txt
> cargo test

error[E0433]: failed to resolve: use of undeclared type or module `List`
  --> src/first.rs:43:24
   |
43 |         let mut list = List::new();
   |                        ^^^^ use of undeclared type or module `List`
```

哎呀！因为我们创建了一个新模块，所以我们需要明确导入`List`才能使用它。

```rust
mod test {
    use super::List;
    // everything else the same
    ...
}
```

```txt
> cargo test

warning: unused import: `super::List`
  --> src/first.rs:45:9
   |
45 |     use super::List;
   |         ^^^^^^^^^^^
   |
   = note: #[warn(unused_imports)] on by default

    Finished dev [unoptimized + debuginfo] target(s) in 0.43s
     Running /Users/ADesires/dev/lists/target/debug/deps/lists-86544f1d97438f1f

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
; 0 filtered out
```

耶！

但是这个警告是怎么回事……？我们在测试中显然使用了`List`！

...但仅限于测试时！为了安抚编译器（并对我们的消费者友好），我们应该指出，只有在运行测试时才应该编译整个测试模块。

```rust
#[cfg(test)]
mod test {
    use super::List;
    // everything else the same
    ...
}
```

这就是测试的全部内容！

## 2.7 实现`Drop`特征

我们可以创建一个堆栈，将其押入，弹出，我们甚至测试过它是否正常工作！

我们需要担心清理我们的列表吗？从技术上讲，完全不需要！与C++语言一样，Rust使用析构函数在资源使用完毕后自动清理它们。如果类型实现了名为`Drop`的特征，则它有一个析构函数。特征是Rust对接口的别称。`Drop`特征具有以下接口：

```rust
pub trait Drop {
    fn drop(&mut self);
}
```

基本上，“当你超出范围时，我会给你一秒钟来清理你的事务”。

如果你包含实现`Drop`的类型，你实际上不需要实现`Drop`，你只需要调用它们的析构函数。在`List`的情况下，它只需要删除它的头部，而头部又可能会尝试删除`Box<Node>`。所有这些都会自动为我们处理……只有一个问题。

自动处理会很糟糕。

让我们考虑一个简单的列表：

```txt
list -> A -> B -> C
```

当`list`被丢弃时，它会尝试丢弃`A`，`A`会尝试丢弃`B`，`B`会尝试丢弃`C`。有些人可能会感到紧张。这是递归代码，而递归代码可能会破坏堆栈！

你们中的一些人可能会想“这显然是尾递归，任何像样的语言都会确保这样的代码不会破坏堆栈”。事实上，这是不正确的！为了了解原因，让我们尝试编写编译器必须执行的操作，通过手动实现`List`的`Drop`特征，就像编译器一样：

```rust
impl Drop for List {
    fn drop(&mut self) {
        // NOTE: you can't actually explicitly call `drop` in real Rust code;
        // we're pretending to be the compiler!
        self.head.drop(); // tail recursive - good!
    }
}

impl Drop for Link {
    fn drop(&mut self) {
        match *self {
            Link::Empty => {} // Done!
            Link::More(ref mut boxed_node) => {
                boxed_node.drop(); // tail recursive - good!
            }
        }
    }
}

impl Drop for Box<Node> {
    fn drop(&mut self) {
        self.ptr.drop(); // uh oh, not tail recursive!
        deallocate(self.ptr);
    }
}

impl Drop for Node {
    fn drop(&mut self) {
        self.next.drop();
    }
}
```

上面的代码显示，我们不能在释放后删除`Box`的内容，所以没有办法以尾部递归的方式删除！（不能做成尾递归，所以，为了避免执行自动`drop`时递归导致的栈溢出，）我们必须手动为`List`编写一个迭代删除程序，将`Node`从`Box`中提升出来。

（秘诀还是`mem::replace`，使用`Link::Empty`作为参数）

```rust
impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, Link::Empty);
        // `while let` == "do this thing until this pattern doesn't match"
        while let Link::More(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, Link::Empty);
            // boxed_node goes out of scope and gets dropped here;
            // but its Node's `next` field has been set to Link::Empty
            // so no unbounded recursion occurs.
        }
    }
}
```

```txt
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured
```

非常好！

----

### 过早优化的奖励部分

我们对`drop`的实现实际上与`while let Some(_) = self.pop() { }`非常相似，后者当然更简单。它有何不同？一旦我们开始将列表泛型化为存储整数以外的东西，它会导致哪些性能问题？

答案：

`pop`方法返回`Option<i32>`，而我们的实现仅操作`Link`（即`Box<Node>`）。因此，我们的实现仅围绕指向节点的指针移动，而基于`pop`的实现将围绕我们存储在节点中的值移动。如果我们概括我们的列表并且有人使用它来存储非常大的东西并带有析构方法（`VeryBigThingWithADropImpl`缩写为`VBTWADI`的实例，这可能会非常昂贵，也就是更耗费内存而且更慢。`Box`能够就地运行其内容的`drop`实现，因此它不会受到此问题的影响。由于`VBTWADI`正是那种实际上使用链表时比数组更好用，因此在这种情况下性能表现不佳会有点令人失望。所以还是用我们代码中的版本为好。

如果您希望同时拥有两种实现中的最佳功能，以简化代码消除重复结构，您可以添加一种新方法，`fn pop_node(&mut self) -> Link`，`pop`和`drop`都可以从中干净地派生出来。

```rust
impl List {
    fn pop_node(&mut self) -> Link {
        mem::replace(&mut self.head, Link::Empty)
    }
}
```

如你所见，`pop_node`方法只是对函数`mem::replace`做了一个简单封装，后面的使用泛型的版本中，我们实际用其他函数替换了它。（所以这里称之为“过早优化”）

## 2.8 最终代码

好了，写了6000个字之后，下面就是我们实际编写的所有代码。

```rust
use std::mem;

pub struct List {
    head: Link,
}

enum Link {
    Empty,
    More(Box<Node>),
}

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: Link::Empty }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: mem::replace(&mut self.head, Link::Empty),
        });

        self.head = Link::More(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match mem::replace(&mut self.head, Link::Empty) {
            Link::Empty => None,
            Link::More(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, Link::Empty);

        while let Link::More(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, Link::Empty);
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
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push(4);
        list.push(5);

        // Check normal removal
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), None);
    }
}
```

天哪。80 行代码，一半都是测试！好吧，我确实说过第一个测试会花点时间！
