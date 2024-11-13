# 3. 一个好的单链栈

在上一章中，我们编写了一个最小可行单链栈。但是，有一些设计决策让它有点糟糕。下面我们来修改它，让它不那么糟糕。在此过程中，我们有以下目标：

- 不再发明轮子
- 让我们的列表能够处理任何元素类型
- 添加`peek`方法
- 让我们的列表可迭代

在此过程中，我们将了解

- 高级`Option`使用
- 泛型
- 生命周期
- 迭代器

以上每项对应一个目标。

让我们添加一个名为`second.rs`的新文件：

```rust
// in lib.rs

pub mod first;
pub mod second;
```

并将`first.rs`中的所有内容复制到其中。

## 3.1 使用`Option`

特别细心的读者可能已经注意到，我们实际上重新发明了一个非常糟糕的`Option`版本：

```rust
enum Link {
    Empty,
    More(Box<Node>),
}
```

`Link`实际就是`Option<Box<Node>>`。现在，不必到处都写`Option<Box<Node>>`，这很好，而且与`pop`不同，我们不会将其暴露给外界，所以也许这样也行。但是`Option`有一些非常好的方法，我们一直在手动实现。我们不要这样做，而是用`Options`替换所有内容。首先，我们将简单地重命名所有内容以使用`Some`和`None`：

```rust
use std::mem;

pub struct List {
    head: Link,
}

// yay type aliases!
type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: mem::replace(&mut self.head, None),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match mem::replace(&mut self.head, None) {
            None => None,
            Some(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, None);
        while let Some(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, None);
        }
    }
}
```

简单来说，就是把`Link::Empty`替换为`None`，把`Link::More`替换为`Some`。

这稍微好一些，但最大的胜利将来自于`Option`提供的方法。

首先，`mem::replace(&mut option, None)`是一种非常常见的用法，`Option`实际上只是封装了它，将其变成一个方法：`take`。

（原文）
> `pub fn take(&mut self) -> Option<T>`
>
> Takes the value out of the option, leaving a None in its place.
>
> Examples
>
> ```rust
> let mut x = Some(2);
> let y = x.take();
> assert_eq!(x, None);
> assert_eq!(y, Some(2));
> 
> let mut x: Option<u32> = None;
> let y = x.take();
> assert_eq!(x, None);
> assert_eq!(y, None);
> ```

（翻译）
> `pub fn take(&mut self) -> Option<T>`
>
> 从选项中取出值，将`None`留在其位置。
>
> 示例
>
> ```rust
> let mut x = Some(2);
> let y = x.take();
> assert_eq!(x, None);
> assert_eq!(y, Some(2));
> 
> let mut x: Option<u32> = None;
> let y = x.take();
> assert_eq!(x, None);
> assert_eq!(y, None);
> ```

所以我们可以用`take`方法替换它。

```rust
pub struct List {
    head: Link,
}

type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match self.head.take() {
            None => None,
            Some(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

其次，匹配臂`{ None => None, Some(x) => Some(y) }`是一种非常常见的习惯用法，因此它被称为`map`。`map`接受一个函数，在`Some(x)`中的`x`上执行该函数，以生成`Some(y)`中的`y`。我们可以编写一个适当的函数，并将其传递给`map`，但我们更愿意将要执行的操作写在内联的匿名函数中。

```rust
    pub fn pop(&mut self) -> Option<i32> {
        self.head.take().map(|node| {
            self.head = node.next;
            node.elem
        })
    }
```

啊，好多了。让我们确保没有破坏任何东西：

```txt
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured
```

太棒了！让我们继续实际改进代码的行为。

## 3.2 使用泛型

我们已经通过`Option`和`Box`稍微接触过泛型。但是到目前为止，我们已经设法避免声明任何实际上对任意元素都是泛型的新类型。

事实证明这实际上非常简单。现在让我们将所有类型都设为泛型：

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

你只要让所有东西都加上尖括号，你的代码就会突然变得支持泛型。当然，我们不能只做这个，否则编译器会非常生气。

```txt
> cargo test

error[E0107]: wrong number of type arguments: expected 1, found 0
  --> src/second.rs:14:6
   |
14 | impl List {
   |      ^^^^ expected 1 type argument

error[E0107]: wrong number of type arguments: expected 1, found 0
  --> src/second.rs:36:15
   |
36 | impl Drop for List {
   |               ^^^^ expected 1 type argument
```

问题非常明显：我们正在谈论这个`List`东西，但它不再是真实的。像`Option`和`Box`一样，我们现在总是必须谈论`List<Something>`。

但我们在所有这些实现中使用的`Something`是什么？就像`List`一样，我们希望我们的实现能够与所有`T`一起工作。所以，就像`List`一样，让我们​​让我们的实现也加上尖括号，方法的参数和返回值类型也由`i32`改为`T`：

```rust
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<T> {
        self.head.take().map(|node| {
            self.head = node.next;
            node.elem
        })
    }
}

impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

...就是这样！

```txt
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured
```

现在，我们所有的代码都完全适用于任意的`T`值。天哪，Rust太简单了。我想特别指出的是，`new`方法甚至没有改变：

```rust
    pub fn new() -> Self {
        List { head: None }
    }
```

沐浴在`Self`的荣耀中，它是重构和复制粘贴编码的守护者。同样有趣的是，我们在构造列表实例时不会写`List<T>`。这部分是基于我们从一个需要`List<T>`的函数返回它这一事实推断出来的。

好吧，让我们继续讨论全新的行为！

## 3.3 `peek`方法

上一章我们甚至没有费心实现的一件事就是窥视（`peek`）方法。让我们继续做吧。我们需要做的就是返回对列表头部元素的引用（如果存在）。听起来很简单，让我们试试：

```rust
    pub fn peek(&self) -> Option<&T> {
        self.head.map(|node| {
            &node.elem
        })
    }
```

```txt
> cargo build

error[E0515]: cannot return reference to local data `node.elem`
  --> src/second.rs:37:13
   |
37 |             &node.elem
   |             ^^^^^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:36:9
   |
36 |         self.head.map(|node| {
   |         ^^^^^^^^^ cannot move out of borrowed content
```

唉。Rust又不高兴了，现在怎么办？

`Map`按值获取自身，这会将`Option`移出其所在位置。以前这没问题，因为我们刚刚将其取出，但现在我们实际上想将其留在原处。处理此问题的正确方法是使用`Option`上的`as_ref`方法，该方法具有以下定义：

```rust
impl<T> Option<T> {
    pub fn as_ref(&self) -> Option<&T>;
}
```

（原文）
> `pub const fn as_ref(&self) -> Option<&T>`
>
> Converts from `&Option<T>` to `Option<&T>`.
>
> Examples
>
> Calculates the length of an `Option<String>` as an `Option<usize>` without moving the String. The map method takes the self argument by value, consuming the original, so this technique uses as_ref to first take an Option to a reference to the value inside the original.
>
> ```rust
> let text: Option<String> = Some("Hello, world!".to_string());
> // First, cast `Option<String>` to `Option<&String>` with `as_ref`,
> // then consume *that* with `map`, leaving `text` on the stack.
> let text_length: Option<usize> = text.as_ref().map(|s| s.len());
> println!("still can print text: {text:?}");
> ```

（翻译）
> `pub const fn as_ref(&self) -> Option<&T>`
>
> 从`&Option<T>`转换为`Option<&T>`。
>
> 示例
>
> 在不移动`String`的情况下将`Option<String>`的长度计算为`Option<usize>`。`map`方法按值使用`self`参数，从而消耗了原始文件，因此该技术使用`as_ref`首先将`Option`引用给原始文件中的值。
>
> ```rust
> let text: Option<String> = Some("Hello, world!".to_string());
> // 首先，使用`as_ref`把`Option<String>`强制转换为`Option<&String>`，
> // 然后，使用`map`消耗掉*它*, 在栈上保留`text`。
> let text_length: Option<usize> = text.as_ref().map(|s| s.len());
> println!("still can print text: {text:?}");
> ```

`as_ref`先将`Option<T>`降级为`Option`，然后将其作为对其内部的引用。我们可以使用显式匹配自己完成此操作，但不行。这确实意味着我们需要执行额外的取消引用来消除额外的间接引用，但幸运的是`.`运算符可以为我们处理此问题。

```rust
    pub fn peek(&self) -> Option<&T> {
        self.head.as_ref().map(|node| {
            &node.elem
        })
    }
```

```txt
> cargo build

    Finished dev [unoptimized + debuginfo] target(s) in 0.32s
```

搞定了。

类似的，我们还可以使用`as_mut`创建此方法的可变版本：

```rust
    pub fn peek_mut(&mut self) -> Option<&mut T> {
        self.head.as_mut().map(|node| {
            &mut node.elem
        })
    }
```

```txt
> cargo build

    Finished dev [unoptimized + debuginfo] target(s) in 0.32s
```

很简单。

别忘了测试一下：

```rust
    #[test]
    fn peek() {
        let mut list = List::new();
        assert_eq!(list.peek(), None);
        assert_eq!(list.peek_mut(), None);
        list.push(1); list.push(2); list.push(3);

        assert_eq!(list.peek(), Some(&3));
        assert_eq!(list.peek_mut(), Some(&mut 3));
    }
```

```txt
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 3 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::peek ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured
```

这很好，但我们并没有真正测试过是否可以改变`peek_mut`返回值，不是吗？如果引用是可变的，但没有人改变它，我们真的测试过可变性吗？让我们尝试在这个`Option<&mut T>`上使用`map`来放入一个深刻的值：

```rust
    #[test]
    fn peek() {
        let mut list = List::new();
        assert_eq!(list.peek(), None);
        assert_eq!(list.peek_mut(), None);
        list.push(1); list.push(2); list.push(3);

        assert_eq!(list.peek(), Some(&3));
        assert_eq!(list.peek_mut(), Some(&mut 3));
        list.peek_mut().map(|&mut value| {
            value = 42
        });

        assert_eq!(list.peek(), Some(&42));
        assert_eq!(list.pop(), Some(42));
    }
```

```txt
> cargo test

error[E0384]: cannot assign twice to immutable variable `value`
   --> src/second.rs:100:13
    |
99  |         list.peek_mut().map(|&mut value| {
    |                                   -----
    |                                   |
    |                                   first assignment to `value`
    |                                   help: make this binding mutable: `mut value`
100 |             value = 42
    |             ^^^^^^^^^^ cannot assign twice to immutable variable          ^~~~~
```

编译器抱怨说`value`是不可变的，但我们清楚地写了`&mut value`；怎么回事？事实证明，以这种方式编写闭包的参数并没有指定`value`是可变引用。相反，它创建了一个与闭包参数匹配的模式；`|&mut value|`表示“参数是可变引用，但请将其指向的值复制到`value`中。”如果我们只使用`|value|`，`value`的类型将是`&mut i32`，我们实际上可以改变`value`：

```rust
    #[test]
    fn peek() {
        let mut list = List::new();
        assert_eq!(list.peek(), None);
        assert_eq!(list.peek_mut(), None);
        list.push(1); list.push(2); list.push(3);

        assert_eq!(list.peek(), Some(&3));
        assert_eq!(list.peek_mut(), Some(&mut 3));

        list.peek_mut().map(|value| {
            *value = 42
        });

        assert_eq!(list.peek(), Some(&42));
        assert_eq!(list.pop(), Some(42));
    }
```

```txt
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 3 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::peek ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured
```

好多了！

## 3.4 `IntoIter`

在Rust中，集合使用`Iterator`特征进行迭代。它比`Drop`稍微复杂一些：

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

这里的新成员是`Item`类型。这声明了`Iterator`的每个实现都有一个名为`Item`的关联类型。在本例中，这是调用`next`方法时它可以输出的类型。

`Iterator`返回`Option<Self::Item>`的原因是因为接口合并了`has_next`和`get_next`概念。当集合的迭代器中有下一个值时，会产生`Some(value)`，而当您没有下一个值时，会产生`None`。这使得API通常更符合人体工程学且更易于使用和实现，同时避免了`has_next`和`get_next`之间的冗余检查和逻辑。太棒了！

遗憾的是，Rust还没有类似`yield`语句的东西（目前），所以我们必须自己实现逻辑。此外，每个集合实际上应该努力实现3种不同类型的迭代器：

- `IntoIter` - `T`，迭代得到值，会消耗掉`List`的全部值；
- `IterMut` - `&mut T`，迭代得到可变引用；
- `Iter` - `&T`，迭代得到共享引用，原`List`还存在。

实际上，我们已经拥有使用`List`接口实现`IntoIter`的所有工具：只需反复调用`pop`即可。因此，我们只需将`IntoIter`实现为`List`的新类型包装器：

```rust
// Tuple structs are an alternative form of struct,
// useful for trivial wrappers around other types.
pub struct IntoIter<T>(List<T>);

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        // access fields of a tuple struct numerically
        self.0.pop()
    }
}
```

让我们写一个测试：

```rust
    #[test]
    fn into_iter() {
        let mut list = List::new();
        list.push(1); list.push(2); list.push(3);

        let mut iter = list.into_iter();
        assert_eq!(iter.next(), Some(3));
        assert_eq!(iter.next(), Some(2));
        assert_eq!(iter.next(), Some(1));
        assert_eq!(iter.next(), None);
    }
```

```txt
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 4 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured
```

很好！

## 3.5 `Iter`

好吧，让我们尝试实现`Iter`。这次我们不能依赖`List`来提供我们想要的所有功能。我们需要自己动手。我们想要的基本逻辑是保存指向我们想要接下来产生的当前节点的指针。因为该节点可能不存在（列表为空或我们已完成迭代），所以我们希望该引用是一个`Option`。当我们产生一个元素时，我们希望继续到当前节点的下一个节点。

好吧，让我们试试：

```rust
pub struct Iter<T> {
    next: Option<&Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

impl<T> Iterator for Iter<T> {
    type Item = &T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

```txt
> cargo build

error[E0106]: missing lifetime specifier
  --> src/second.rs:72:18
   |
72 |     next: Option<&Node<T>>,
   |                  ^ expected lifetime parameter

error[E0106]: missing lifetime specifier
  --> src/second.rs:82:17
   |
82 |     type Item = &T;
   |                 ^ expected lifetime parameter
```

天哪。好久不见。我听说过这些事情。我听说它们是一场噩梦。

让我们尝试一些新的东西：看到那个`error[E0106]`了吗？那是编译器错误代码。我们可以要求`rustc`用`--explain`来解释这些错误：

```txt
> rustc --explain E0106
This error indicates that a lifetime is missing from a type. If it is an error
inside a function signature, the problem may be with failing to adhere to the
lifetime elision rules (see below).

Here are some simple examples of where you'll run into this error:

struct Foo { x: &bool }        // error
struct Foo<'a> { x: &'a bool } // correct

enum Bar { A(u8), B(&bool), }        // error
enum Bar<'a> { A(u8), B(&'a bool), } // correct

type MyStr = &str;        // error
type MyStr<'a> = &'a str; //correct
...
```

那呃……那并没有真正澄清很多（这些文档假设我们比现在更了解Rust）。但看起来我们应该将这些`'a`东西添加到我们的结构中？让我们试试吧。先修改结构体定义。

```rust
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}
```

```txt
> cargo build

error[E0106]: missing lifetime specifier
  --> src/second.rs:83:22
   |
83 | impl<T> Iterator for Iter<T> {
   |                      ^^^^^^^ expected lifetime parameter

error[E0106]: missing lifetime specifier
  --> src/second.rs:84:17
   |
84 |     type Item = &T;
   |                 ^ expected lifetime parameter

error: aborting due to 2 previous errors
```

好吧，我开始在这里看到一种模式...让我们把这些小家伙`'a`添加到我们能添加的一切中：

```rust
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<'a, T> List<T> {
    pub fn iter(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &'a node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    fn next(&'a mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &'a node);
            &'a node.elem
        })
    }
}
```

```txt
> cargo build

error: expected `:`, found `node`
  --> src/second.rs:77:47
   |
77 |         Iter { next: self.head.map(|node| &'a node) }
   |         ---- while parsing this struct        ^^^^ expected `:`

error: expected `:`, found `node`
  --> src/second.rs:85:50
   |
85 |             self.next = node.next.map(|node| &'a node);
   |                                                  ^^^^ expected `:`

error[E0063]: missing field `next` in initializer of `second::Iter<'_, _>`
  --> src/second.rs:77:9
   |
77 |         Iter { next: self.head.map(|node| &'a node) }
   |         ^^^^ missing `next`
```

天哪。我们搞坏了Rust。

也许我们应该弄清楚这个`'a`表示的“生命周期”到底是什么意思。

生命周期可能会吓跑很多人，因为它们改变了我们自编程诞生以来就熟知和喜爱的东西。到目前为止，我们实际上已经设法避开了生命周期，尽管它们一直纠缠在我们的程序中。

在垃圾收集语言中，生命周期是不必要的，因为垃圾收集器可以确保所有东西都能神奇地存活到需要的期限。Rust中的大多数数据都是手动管理的，因此数据需要另一种解决方案。C和C++为我们提供了一个清晰的例子，如果你只是让人们将指针指向堆栈上的随机数据，会发生什么：普遍的无法管理的不安全性。这可以大致分为两类错误：

- 持有指向超出范围的东西的指针
- 持有指向已经修改的东西的指针

生命周期解决了这两个问题，并且99％的时间都是以完全透明的方式实现的。

那么什么是生命周期？

简而言之，生命周期是程序中某个代码区域（～块/范围）的名称。就是这样。当引用被标记为生命周期时，我们说它必须对整个区域有效。不同的事情对引用必须和可以有效的时间提出了要求。整个生命周期系统反过来只是一个约束求解系统，它试图最小化每个引用的区域。如果它成功找到一组满足所有约束的生命周期，您的程序就会编译！否则，您会收到一条错误，提示某些东西存活的时间不够长。

在函数体内，通常不能谈论生命周期，而且无论如何也不想谈论。编译器拥有完整的信息，可以推断出所有约束以找到最小生命周期。但是在类型和API级别，编译器并不拥有所有信息。它需要您告诉它不同生命周期之间的关系，以便它能够弄清楚您在做什么。

原则上，这些生命周期也可以省略，但检查所有借用将是一项巨大的全程序分析，会产生令人难以置信的非本地错误。Rust的系统意味着所有借用检查都可以在每个函数体中独立完成，并且所有错误都应该是相当本地的（否则您的类型具有不正确的签名）。

但是我们之前在函数签名中写过引用，这没问题！这是因为有些情况非常常见，Rust 会自动为您选择生命周期。这就是生命周期省略。

生命周期省略的示例如下，在下面的代码中，相邻的两行意义相同：

```rust
// Only one reference in input, so the output must be derived from that input
fn foo(&A) -> &B; // sugar for:
fn foo<'a>(&'a A) -> &'a B;

// Many inputs, assume they're all independent
fn foo(&A, &B, &C); // sugar for:
fn foo<'a, 'b, 'c>(&'a A, &'b B, &'c C);

// Methods, assume all output lifetimes are derived from `self`
fn foo(&self, &B, &C) -> &D; // sugar for:
fn foo<'a, 'b, 'c>(&'a self, &'b B, &'c C) -> &'a D;
```

那么`fn foo<'a>(&'a A) -> &'a B`是什么意思呢？实际上，它意味着输入必须至少与输出一样长。因此，如果你长时间保留输出，这将扩大输入必须有效的区域。一旦你停止使用输出，编译器就会知道输入变得无效也是可以的。

通过设置此系统，Rust可以确保释放后不会使用任何内容，并且当存在未完成引用时不会发生任何变化。它只是确保所有约束都起作用！

好吧。那么回到`Iter`。

让我们回滚到无生命周期状态：

```rust
pub struct Iter<T> {
    next: Option<&Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

impl<T> Iterator for Iter<T> {
    type Item = &T;
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

我们只需要在函数和类型签名中添加生命周期：

```rust
// Iter is generic over *some* lifetime, it doesn't care
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

// No lifetime here, List doesn't have any associated lifetimes
impl<T> List<T> {
    // We declare a fresh lifetime here for the *exact* borrow that
    // creates the iter. Now &self needs to be valid as long as the
    // Iter is around.
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

// We *do* have a lifetime here, because Iter has one that we need to define
impl<'a, T> Iterator for Iter<'a, T> {
    // Need it here too, this is a type declaration
    type Item = &'a T;

    // None of this needs to change, handled by the above.
    // Self continues to be incredibly hype and amazing
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

好吧，我想这次我们明白了。

```txt
> cargo build

error[E0308]: mismatched types
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.map(|node| &node) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

error[E0308]: mismatched types
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.map(|node| &node);
   |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&'a second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`
```

很奇怪。

好的。那么。我们修复了生命周期错误，但现在我们遇到了一些新的类型错误。

我们想要存储`&Node`，但我们得到的是`&Box<Node>`。好的，这很简单，我们只需要在引用之前解引用`Box`：

```rust
impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &*node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &*node);
            &node.elem
        })
    }
}
```

```txt
> cargo build
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
error[E0515]: cannot return reference to local data `*node`
  --> src/second.rs:77:43
   |
77 |         Iter { next: self.head.map(|node| &*node) }
   |                                           ^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.map(|node| &*node) }
   |                      ^^^^^^^^^ cannot move out of borrowed content

error[E0515]: cannot return reference to local data `*node`
  --> src/second.rs:85:46
   |
85 |             self.next = node.next.map(|node| &*node);
   |                                              ^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.map(|node| &*node);
   |                         ^^^^^^^^^ cannot move out of borrowed content
```

还有问题。

我们忘记了使用`as_ref`，所以我们将`Box`移到`map`中，这意味着它将被删除，这意味着我们的引用将会悬空：

```rust
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_ref().map(|node| &*node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_ref().map(|node| &*node);
            &node.elem
        })
    }
}
```

```txt
> cargo build
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
error[E0308]: mismatched types
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.as_ref().map(|node| &*node) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

error[E0308]: mismatched types
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.as_ref().map(|node| &*node);
   |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&'a second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`
```

😭

`as_ref`增加了另一层间接层，我们需要删除它：

```rust
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
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

```txt
> cargo build
```

编译成功了！🎉 🎉 🎉

（原文）
>
> ```rust
> pub fn as_deref(&self) -> Option<&<T as Deref>::Target>
> where
>     T: Deref,
> ```
>
> Converts from `Option<T>` (or `&Option<T>`) to `Option<&T::Target>`.
>
> Leaves the original Option in-place, creating a new one with a reference to the original one, additionally coercing the contents via Deref.
>
> Examples
>
> ```rust
> let x: Option<String> = Some("hey".to_owned());
> assert_eq!(x.as_deref(), Some("hey"));
> 
> let x: Option<String> = None;
> assert_eq!(x.as_deref(), None);
> ```

（翻译）
>
> ```rust
> pub fn as_deref(&self) -> Option<&<T as Deref>::Target>
> where
>     T: Deref,
> ```
>
> 从`Option<T>`（或`&Option<T>`）转换为`Option<&T::Target>`。
>
> 将原始`Option`保留在原位，创建一个带有对原始`Option`的引用的新`Option`，并通过`Deref`强制执行其内容。
>
> 示例
>
> ```rust
> let x: Option<String> = Some("hey".to_owned());
> assert_eq!(x.as_deref(), Some("hey"));
> 
> let x: Option<String> = None;
> assert_eq!(x.as_deref(), None);
> ```

从Rust 1.40开始，`as_deref`和`as_deref_mut`函数就稳定了。在此之前，您需要执行`map(|node| &**node)`和`map(|node| &mut**node)`。您可能会想“哇，那个`&**`东西真的很差劲”，您没有错，但就像一杯好酒一样，Rust会随着时间的推移变得更好，我们不再需要这样做。通常，Rust非常擅长通过称为`deref`强制转换的过程隐式地执行这种转换，基本上它可以在您的代码中插入`*`以进行类型检查。它可以做到这一点，因为我们有借用检查器来确保我们永远不会弄乱指针！

但在这种情况下，闭包与我们拥有`Option<&T>`而不是`&T`的事实相结合，有点太复杂了，所以我们需要通过显式来帮助它。幸运的是，根据我的经验，这种情况非常罕见。

只是为了完整性起见，我们可以用快鱼操作符给它一个不同的提示：

```rust
    self.next = node.next.as_ref().map::<&Node<T>, _>(|node| &node);
```

看看，`map`是一个泛型函数：

```rust
pub fn map<U, F>(self, f: F) -> Option<U>
```

快鱼操作符`::<>`让我们告诉编译器我们认为这些泛型的类型应该是什么。在这种情况下`::<&Node<T>, _>`表示“对于泛型`U`应该是类型`&Node<T>`；而对于第二个泛型类型`F`，我不知道/不关心他是什么类型”，两个泛型类型之间用`,`分隔。

这反过来让编译器知道`&node`应该应用强制解引用（`deref coercion`），这样我们就不需要手动应用所有这些`*`，当然，你加上也不会报错！

但在这种情况下，我认为这并不是真正的改进，这只是一个毫不掩饰的借口，用来炫耀强制解引用和有时有用的快鱼操作符。😅

（就是说，这里还是用`as_deref`和`as_deref_mut`比较好。）

让我们编写一个测试，以确保我们没有对其进行空操作或任何其他操作：

```rust
    #[test]
    fn iter() {
        let mut list = List::new();
        list.push(1); list.push(2); list.push(3);

        let mut iter = list.iter();
        assert_eq!(iter.next(), Some(&3));
        assert_eq!(iter.next(), Some(&2));
        assert_eq!(iter.next(), Some(&1));
    }
```

```txt
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 5 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::peek ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured
```

测试通过了。

最后，需要注意的是，我们实际上可以在这里应用生命周期省略：

```rust
    impl<T> List<T> {
        pub fn iter<'a>(&'a self) -> Iter<'a, T> {
            Iter { next: self.head.as_deref() }
        }
    }
```

等价于：

```rust
impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.as_deref() }
    }
}
```

耶，生命周期更少了！

或者，如果您不愿意“隐藏”结构包含的生命周期，则可以使用Rust 2018“明确省略的生命周期”语法，`'_`:

```rust
impl<T> List<T> {
    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head.as_deref() }
    }
}
```

## 3.6 `IterMut`

说实话，`IterMut`很疯狂。这本身似乎是一件疯狂的事情；它肯定与`Iter`相同！

从语义上讲，是的，但共享和可变引用的性质意味着`Iter`是“微不足道的”，而`IterMut`是合法的巫师魔法。

关键见解来自我们对`Iter`的`Iterator`的实现：

```rust
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> { /* stuff */ }
}
```

可以将其解语法糖，加上明确的生命周期就是这样的：

```rust
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next<'b>(&'b mut self) -> Option<&'a T> { /* stuff */ }
}
```

`next`的签名没有在输入和输出的生命周期之间建立任何约束！我们为什么要关心它？这意味着我们可以无条件地反复调用`next`！

```rust
let mut list = List::new();
list.push(1); list.push(2); list.push(3);

let mut iter = list.iter();
let x = iter.next().unwrap();
let y = iter.next().unwrap();
let z = iter.next().unwrap();
```

这很酷！

这对于共享引用来说绝对没问题，因为关键在于你可以同时拥有大量引用。然而可变引用不能共存。关键在于它们是独占的。

最终结果是使用安全代码编写`IterMut`明显更加困难（我们还没有深入讨论这意味着什么……）。令人惊讶的是，`IterMut`实际上可以完全安全地为许多结构实现！

我们将从获取`Iter`代码并将所有内容更改为可变开始：

```rust
pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}

impl<T> List<T> {
    pub fn iter_mut(&self) -> IterMut<'_, T> {
        IterMut { next: self.head.as_deref_mut() }
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_deref_mut();
            &mut node.elem
        })
    }
}
```

```txt
> cargo build
error[E0596]: cannot borrow `self.head` as mutable, as it is behind a `&` reference
  --> src/second.rs:95:25
   |
94 |     pub fn iter_mut(&self) -> IterMut<'_, T> {
   |                     ----- help: consider changing this to be a mutable reference: `&mut self`
95 |         IterMut { next: self.head.as_deref_mut() }
   |                         ^^^^^^^^^ `self` is a `&` reference, so the data it refers to cannot be borrowed as mutable

error[E0507]: cannot move out of borrowed content
   --> src/second.rs:103:9
    |
103 |         self.next.map(|node| {
    |         ^^^^^^^^^ cannot move out of borrowed content
```

好吧，看起来我们这里有两个不同的错误。第一个看起来非常清楚，它甚至告诉我们如何修复它！您无法将共享引用升级为可变引用，因此`iter_mut`需要采用`&mut self`。只是一个愚蠢的复制粘贴错误。

```rust
    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        IterMut { next: self.head.as_deref_mut() }
    }
```

另一个编译错误要怎么处理呢？

哎呀！实际上，我在上一节中编写`Iter`的实现时不小心犯了一个错误，我们只是很幸运它能正常工作！

我们刚刚第一次体验了复制所有权的魔力。当我们引入[所有权](https://rust-unofficial.github.io/too-many-lists/first-ownership.html)时，我们说过，当你移动东西时，你就不能再使用它了。对于某些类型，这完全说得通。我们的好朋友`Box`为我们管理堆上的分配，我们当然不希望两段代码认为它们需要释放它的内存。

然而对于其他类型来说，这是垃圾。整数没有所有权语义；它们只是毫无意义的数字！这就是整数被标记为`Copy`特征的原因。众所周知，`Copy`类型可以通过按位复制完美复制。因此，它们具有超能力：移动时，旧值仍然可用。因此，您甚至可以将`Copy`类型从引用中移出而无需替换！

Rust中的所有数字原语（`i32`、`u64`、`bool`、`f32`、`char`等）都是`Copy`类型。您还可以将任何用户定义类型声明为`Copy`的，只要其所有内部组件都是`Copy`的。

这段代码之所以能正常工作，关键在于**共享引用也是`Copy`类型**！因为`&T`是`Copy`类型，所以`Option<&>`也是`Copy`类型。因此，当我们执行`self.next.map`时，一切正常，因为`Option`只是被复制了。

现在我们不能这样做，因为**`&mut T`不是`Copy`类型（如果您复制了`&mut`，那么您将有两个`&mut`位于内存中的同一位置，这是被禁止的）**，甚至它也不是`Clone`类型（当对一个`&mut T`显式调用`clone()`方法时会调用`T.clone(&self)`方法，这其实还涉及了重借用即`reborrow`的概念）。因此，我们应该正确使用`Option`的方法来获取它。

```rust
    fn next(&mut self) -> Option<Self::Item> {
        self.next.take().map(|node| {
            self.next = node.next.as_deref_mut();
            &mut node.elem
        })
    }
```

```txt
> cargo build
```

呃……哇哦。天哪！`IterMut`真的有效！

让我们测试一下：

```rust
    #[test]
    fn iter_mut() {
        let mut list = List::new();
        list.push(1); list.push(2); list.push(3);

        let mut iter = list.iter_mut();
        assert_eq!(iter.next(), Some(&mut 3));
        assert_eq!(iter.next(), Some(&mut 2));
        assert_eq!(iter.next(), Some(&mut 1));
    }
```

```txt
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 6 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::peek ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured
```

是的。它有效。

天哪。

什么。

好吧，我的意思是它实际上应该有效，但通常会有一些愚蠢的事情妨碍它！让我们在这里澄清一下： 我们刚刚实现了一段代码，它接受一个单链表，并最多一次返回列表中每个元素的可变引用。并且它经过静态验证可以做到这一点。而且它完全安全。我们不需要做任何疯狂的事情。

如果你问我，这是一件大事。（虽然这个`next`方法只有这么短短几行，）有几个原因可以解释为什么它有效：

- 我们从`Option<&mut>`中`take`到了`&mut T`，这样我们就可以独占访问可变引用。无需担心有人再次查看它。
- Rust知道共享一个可变引用到指向的结构的子字段中是可以的，因为没有办法“返回”，而且它们肯定是不相交的。

事实证明，您也可以应用此基本逻辑来为数组或树获取安全的`IterMut`！您甚至可以将迭代器设为`DoubleEnded`，这样您就可以同时从前端和后端使用迭代器！哇哦！

## 3.7 最终代码

好的，这就是第二个列表；这是最终的代码！

```rust
pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<T> {
        self.head.take().map(|node| {
            self.head = node.next;
            node.elem
        })
    }

    pub fn peek(&self) -> Option<&T> {
        self.head.as_ref().map(|node| {
            &node.elem
        })
    }

    pub fn peek_mut(&mut self) -> Option<&mut T> {
        self.head.as_mut().map(|node| {
            &mut node.elem
        })
    }

    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }

    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head.as_deref() }
    }

    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        IterMut { next: self.head.as_deref_mut() }
    }
}

impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}

pub struct IntoIter<T>(List<T>);

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        // access fields of a tuple struct numerically
        self.0.pop()
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

pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.take().map(|node| {
            self.next = node.next.as_deref_mut();
            &mut node.elem
        })
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

    #[test]
    fn peek() {
        let mut list = List::new();
        assert_eq!(list.peek(), None);
        assert_eq!(list.peek_mut(), None);
        list.push(1); list.push(2); list.push(3);

        assert_eq!(list.peek(), Some(&3));
        assert_eq!(list.peek_mut(), Some(&mut 3));

        list.peek_mut().map(|value| {
            *value = 42
        });

        assert_eq!(list.peek(), Some(&42));
        assert_eq!(list.pop(), Some(42));
    }

    #[test]
    fn into_iter() {
        let mut list = List::new();
        list.push(1); list.push(2); list.push(3);

        let mut iter = list.into_iter();
        assert_eq!(iter.next(), Some(3));
        assert_eq!(iter.next(), Some(2));
        assert_eq!(iter.next(), Some(1));
        assert_eq!(iter.next(), None);
    }

    #[test]
    fn iter() {
        let mut list = List::new();
        list.push(1); list.push(2); list.push(3);

        let mut iter = list.iter();
        assert_eq!(iter.next(), Some(&3));
        assert_eq!(iter.next(), Some(&2));
        assert_eq!(iter.next(), Some(&1));
    }

    #[test]
    fn iter_mut() {
        let mut list = List::new();
        list.push(1); list.push(2); list.push(3);

        let mut iter = list.iter_mut();
        assert_eq!(iter.next(), Some(&mut 3));
        assert_eq!(iter.next(), Some(&mut 2));
        assert_eq!(iter.next(), Some(&mut 1));
    }
}
```

越来越猛了！
