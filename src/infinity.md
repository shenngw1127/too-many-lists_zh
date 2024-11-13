# 8. 一堆愚蠢的列表清单

好吧。就是这样。我们制作了所有列表。

啊哈哈哈哈

不

总是有更多列表。

本章是一份活生生的文档，介绍了更荒谬的链表以及它们如何与Rust交互。

- [双重单链表](https://rust-unofficial.github.io/too-many-lists/infinity-double-single.html)
- [栈上分配的链表](https://rust-unofficial.github.io/too-many-lists/infinity-stack-allocated.html)
- 自引用竞技场列表？
- GhostCell列表？

## 8.1 双重单链表

我们很难处理双向链表，因为它们的所有权语义很复杂：没有一个节点严格地拥有任何其他节点。然而，我们很难处理这个问题，因为我们引入了对链表的先入为主的观念。也就是说，我们假设所有链接都朝着同一个方向。

相反，我们可以将列表分成两半：一半向左，一半向右：

```rust
// lib.rs
// ...
pub mod silly1;     // NEW!
```

```rust
// silly1.rs
use crate::second::List as Stack;

struct List<T> {
    left: Stack<T>,
    right: Stack<T>,
}
```

现在，我们拥有的不再是单纯的安全堆栈，而是一个通用列表。我们可以通过向任一堆栈推送来向左或向右扩展列表。我们还可以通过将值从一端弹出到另一端来“遍历”列表。为了避免不必要的分配，我们将复制安全堆栈的源以访问其私有详细信息：

```rust
pub struct Stack<T> {
    head: Link<T>,
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<T> Stack<T> {
    pub fn new() -> Self {
        Stack { head: None }
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
            let node = *node;
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
}

impl<T> Drop for Stack<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

并且稍微修改一下`push`和`pop`操作：

```rust
    pub fn push(&mut self, elem: T) {
        let new_node = Box::new(Node {
            elem: elem,
            next: None,
        });

        self.push_node(new_node);
    }

    fn push_node(&mut self, mut node: Box<Node<T>>) {
        node.next = self.head.take();
        self.head = Some(node);
    }

    pub fn pop(&mut self) -> Option<T> {
        self.pop_node().map(|node| {
            node.elem
        })
    }

    fn pop_node(&mut self) -> Option<Box<Node<T>>> {
        self.head.take().map(|mut node| {
            self.head = node.next.take();
            node
        })
    }
```

现在我们可以制作我们的列表了：

```rust
pub struct List<T> {
    left: Stack<T>,
    right: Stack<T>,
}

impl<T> List<T> {
    fn new() -> Self {
        List { left: Stack::new(), right: Stack::new() }
    }
}
```

我们可以在列表上做一些常见的操作：

```rust
    pub fn push_left(&mut self, elem: T) { self.left.push(elem) }
    pub fn push_right(&mut self, elem: T) { self.right.push(elem) }
    pub fn pop_left(&mut self) -> Option<T> { self.left.pop() }
    pub fn pop_right(&mut self) -> Option<T> { self.right.pop() }
    pub fn peek_left(&self) -> Option<&T> { self.left.peek() }
    pub fn peek_right(&self) -> Option<&T> { self.right.peek() }
    pub fn peek_left_mut(&mut self) -> Option<&mut T> { self.left.peek_mut() }
    pub fn peek_right_mut(&mut self) -> Option<&mut T> { self.right.peek_mut() }
```

但最有趣的是，我们可以四处走走！

```rust
    pub fn go_left(&mut self) -> bool {
        self.left.pop_node().map(|node| {
            self.right.push_node(node);
        }).is_some()
    }

    pub fn go_right(&mut self) -> bool {
        self.right.pop_node().map(|node| {
            self.left.push_node(node);
        }).is_some()
    }
```

我们在这里返回布尔值只是为了方便指示我们是否真的成功移动。现在让我们测试一下这个宝贝：

```rust
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn walk_aboot() {
        let mut list = List::new();             // [_]

        list.push_left(0);                      // [0,_]
        list.push_right(1);                     // [0, _, 1]
        assert_eq!(list.peek_left(), Some(&0));
        assert_eq!(list.peek_right(), Some(&1));

        list.push_left(2);                      // [0, 2, _, 1]
        list.push_left(3);                      // [0, 2, 3, _, 1]
        list.push_right(4);                     // [0, 2, 3, _, 4, 1]

        while list.go_left() {}                 // [_, 0, 2, 3, 4, 1]

        assert_eq!(list.pop_left(), None);
        assert_eq!(list.pop_right(), Some(0));  // [_, 2, 3, 4, 1]
        assert_eq!(list.pop_right(), Some(2));  // [_, 3, 4, 1]

        list.push_left(5);                      // [5, _, 3, 4, 1]
        assert_eq!(list.pop_right(), Some(3));  // [5, _, 4, 1]
        assert_eq!(list.pop_left(), Some(5));   // [_, 4, 1]
        assert_eq!(list.pop_right(), Some(4));  // [_, 1]
        assert_eq!(list.pop_right(), Some(1));  // [_]

        assert_eq!(list.pop_right(), None);
        assert_eq!(list.pop_left(), None);

    }
}
```

```txt
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 16 tests
test fifth::test::into_iter ... ok
test fifth::test::basics ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test fourth::test::into_iter ... ok
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test first::test::basics ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test third::test::basics ... ok
test third::test::iter ... ok
test second::test::peek ... ok
test silly1::test::walk_aboot ... ok

test result: ok. 16 passed; 0 failed; 0 ignored; 0 measured
```

这是手指数据结构的一个极端示例，我们在结构中维护某种手指，因此可以支持与手指距离成比例的时间位置操作。

我们可以对手指周围的列表进行非常快速的更改，但如果我们想对远离手指的位置进行更改，我们必须一路走到那里。我们可以通过将元素从一个堆栈移到另一个堆栈来永久地走到那里，或者我们可以暂时使用`&mut`沿着链接行走以进行更改。但是`&mut`永远无法返回列表，而我们的手指可以！

## 8.2 栈上分配的链表

本书主要关注堆分配的链表，因为它们是最常见和最实用的，但我们不必使用堆分配。堆分配很好，因为它可以轻松动态分配内存。堆栈分配在这方面不太友好——像C的`alloca`这样的东西被广泛认为是非常诅咒和有问题的。

所以让我们用简单的方法在堆栈上分配内存：通过调用一个函数并获取一个具有更多空间的新堆栈框架！这是我们问题的一个非常愚蠢的解决方案，但也真正实用和有用。它一直在做，可能实际上甚至没有把它当作一个链表来考虑！

任何时候你递归地做某事，你都可以将指向当前步骤状态的指针传递给下一步。如果该指针本身是状态的一部分，那么你就创建了一个栈上分配的链表！

现在我们当然已经到了本书的愚蠢部分，所以我们将以一种愚蠢的方式来做到这一点：通过使链接列表成为星号并强制所有用户的代码生活在回调的沼泽中。每个人都喜欢嵌套回调！

我们的`List`类型只是一个引用另一个`Node`的`Node`：

```rust
pub struct List<'a, T> {
    pub data: T,
    pub prev: Option<&'a List<'a, T>>,
}
```

而且它只有一个操作，即`push`，它将接受旧列表、当前节点的状态和一个回调。新列表将在回调中生成。我们还将让回调返回任何值，`push`将在完成时返回该值：

```rust
impl<'a, T> List<'a, T> {
    pub fn push<U>(
        prev: Option<&'a List<'a, T>>, 
        data: T, 
        callback: impl FnOnce(&List<'a, T>) -> U,
    ) -> U {
        let list = List { data, prev };
        callback(&list)
    }
}
```

就这样！我们可以像这样使用它：

```rust
    List::push(None, 3, |list| {
        println!("{}", list.data);
        List::push(Some(list), 5, |list| {
            println!("{}", list.data);
            List::push(Some(list), 13, |list| {
                println!("{}", list.data);
            })
        })
    })
```

很漂亮。😿

用户已经可以使用`while-let`遍历此列表来遍历先前的值（`prev`），但只是为了好玩，让我们实现一个迭代器，它是通常的：

```rust
impl<'a, T> List<'a, T> {
    pub fn iter(&'a self) -> Iter<'a, T> {
        Iter { next: Some(self) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.prev;
            &node.data
        })
    }
}
```

让我们测试一下：

```rust
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn elegance() {
        List::push(None, 3, |list| {
            assert_eq!(list.iter().copied().sum::<i32>(), 3);
            List::push(Some(list), 5, |list| {
                assert_eq!(list.iter().copied().sum::<i32>(), 5 + 3);
                List::push(Some(list), 13, |list| {
                    assert_eq!(list.iter().copied().sum::<i32>(), 13 + 5 + 3);
                })
            })
        })
    }
}
```

```txt
> cargo test

running 18 tests
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test fifth::test::basics ... ok
test fifth::test::miri_food ... ok
test first::test::basics ... ok
test second::test::into_iter ... ok
test fourth::test::peek ... ok
test fourth::test::into_iter ... ok
test second::test::iter_mut ... ok
test fourth::test::basics ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test third::test::basics ... ok
test silly1::test::walk_aboot ... ok
test silly2::test::elegance ... ok
test second::test::peek ... ok
test third::test::iter ... ok

test result: ok. 18 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out;
```

现在你可能会想“嘿，我可以改变存储在节点中的数据吗？”也许吧！让我们尝试让列表使用可变引用而不是共享引用：

```rust
pub struct List<'a, T> {
    pub data: T,
    pub prev: Option<&'a mut List<'a, T>>,
}

pub struct Iter<'a, T> {
    next: Option<&'a List<'a, T>>,
}

impl<'a, T> List<'a, T> {
    pub fn push<U>(
        prev: Option<&'a mut List<'a, T>>, 
        data: T, 
        callback: impl FnOnce(&mut List<'a, T>) -> U,
    ) -> U {
        let mut list = List { data, prev };
        callback(&mut list)
    }

    pub fn iter(&'a self) -> Iter<'a, T> {
        Iter { next: Some(self) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.prev.as_ref().map(|prev| &**prev);
            &node.data
        })
    }
}
```

```txt
> cargo test

error[E0521]: borrowed data escapes outside of closure
  --> src\silly2.rs:47:32
   |
46 |  List::push(Some(list), 13, |list| {
   |                              ----
   |                              |
   |              `list` declared here, outside of the closure body
   |              `list` is a reference that is only valid in the closure body
47 |      assert_eq!(list.iter().copied().sum::<i32>(), 13 + 5 + 3);
   |                 ^^^^^^^^^^^ `list` escapes the closure body here

error[E0521]: borrowed data escapes outside of closure
  --> src\silly2.rs:45:28
   |
44 |  List::push(Some(list), 5, |list| {
   |                             ----
   |                             |
   |              `list` declared here, outside of the closure body
   |              `list` is a reference that is only valid in the closure body
45 |      assert_eq!(list.iter().copied().sum::<i32>(), 5 + 3);
   |                 ^^^^^^^^^^^ `list` escapes the closure body here


<ad infinitum>
```

好啦。看来它不喜欢我们的迭代器。也许我们搞砸了？让我们简化一下测试来检查一下：

```rust
    #[test]
    fn elegance() {
        List::push(None, 3, |list| {
            assert_eq!(list.data, 3);
            List::push(Some(list), 5, |list| {
                assert_eq!(list.data, 5);
                List::push(Some(list), 13, |list| {
                    assert_eq!(list.data, 13);
                })
            })
        })
    }
```

```txt
> cargo test

error[E0521]: borrowed data escapes outside of closure
  --> src\silly2.rs:46:17
   |
44 |   List::push(Some(list), 5, |list| {
   |                              ----
   |                              |
   |              `list` declared here, outside of the closure body
   |              `list` is a reference that is only valid in the closure body
45 |       assert_eq!(list.data, 5);
46 | /     List::push(Some(list), 13, |list| {
47 | |         assert_eq!(list.data, 13);
48 | |     })
   | |______^ `list` escapes the closure body here

error[E0521]: borrowed data escapes outside of closure
  --> src\silly2.rs:44:13
   |
42 |   List::push(None, 3, |list| {
   |                        ----
   |                        |
   |              `list` declared here, outside of the closure body
   |              `list` is a reference that is only valid in the closure body
43 |       assert_eq!(list.data, 3);
44 | /     List::push(Some(list), 5, |list| {
45 | |         assert_eq!(list.data, 5);
46 | |         List::push(Some(list), 13, |list| {
47 | |             assert_eq!(list.data, 13);
48 | |         })
49 | |     })
   | |______________^ `list` escapes the closure body here
```

嗯，不，这仍然是一些热门垃圾。 问题是我们的列表意外地（😉）依赖于型变（`variance`）。[型变是一个复杂的主题](https://doc.rust-lang.org/nomicon/subtyping.html)，但让我们在这里用简单的术语来看一下：

每个列表都包含一个与自身类型完全相同的列表的引用。从最内层列表的角度来看，这意味着所有列表都使用与自身相同的生命周期，但这在客观上是错误的：列表中的每个节点的寿命都严格长于下一个节点，因为它们实际上处于嵌套作用域中！

那么……为什么当我们使用共享引用时代码可以编译？因为在很多情况下，编译器知道某些东西存在的时间“太长”是安全的！当我们将对一个列表的引用填充到下一个列表中时，编译器会悄悄地“缩短”生命周期，以使其符合新列表的期望。这种生命周期缩短就是型变。

在具有继承性的语言中，这和在期望`Animal`（`Cat`类型的超类型）时传递`Cat`的技巧完全相同。直观地，我们知道在期望`Animal`时传递`Cat`是可以的，因为`Cat`只是`Animal`而已。暂时忘记“还有更多”部分也没关系，对吧？

与之相似，较长的生命周期只是较短的生命周期而已。所以在这里忘记“还有更多”也没关系！

但你现在当然会想：那么为什么可变引用版本不起作用！？

嗯，型变并不总是安全的。如果我们的代码确实编译了，我们可以编写这样的“释放之后使用”的代码：

```rust
    List::push(None, 3, |list| {
        List::push(Some(list), 5, |list| {
            List::push(Some(list), 13, |list| {
                // HAHAHA all the lifetimes are the same, so the compiler will
                // let me rewrite my parent to hold a mutable reference to myself!
                // I will create all the use-after-frees!!
                *list.prev.as_mut().unwrap().prev = Some(list);
            })
        })
    })
```

忘记细节的问题在于，其他地方可能会记住这些细节并期望它们保持真实。一旦引入变异，这将是一个非常大的问题。如果你不小心，不记得我们丢弃的“and more”的代码可能会认为将东西写入“记得”的地方是可以的，并且期望“and more”仍然存在。

用继承来表达：此代码必须是非法的：

```rust
    let mut my_kitty = Cat;                  // Make a Cat (long lifetime)
    let animal: &mut Animal = &mut my_kitty; // Forget it's a Cat (shorten lifetime)
    *animal = Dog;                           // Write a Dog (short lifetime)
    my_kitty.meow();                         // Meowing Dog! (Use After Free)
```

因此，虽然您可以缩短可变引用的生命周期，但一旦开始嵌套它们，事物就会变得“不变”，并且您不能再缩短生命周期。

具体来说，`&mut &'big mut T`不能转换为`&mut &'small mut T`，其中`'big`大于`'small`。或者更正式地说，`&'a mut T`对`'a`是协变的，但对`T`是不变的。

有趣的事实：Java语言实际上专门允许您做这种事情，但它[会在运行时进行检查以防止狗喵喵叫](https://docs.oracle.com/javase/7/docs/api/java/lang/ArrayStoreException.html)。

----
那么我们可以做些什么来改变数据呢？使用内部可变性！这让我们告诉编译器我们只想改变数据，但不会触及引用。

我们可以恢复到具有共享引用的先前版本的代码，并在新测试中使用`Cell`：

```rust
    #[test]
    fn cell() {
        use std::cell::Cell;

        List::push(None, Cell::new(3), |list| {
            List::push(Some(list), Cell::new(5), |list| {
                List::push(Some(list), Cell::new(13), |list| {
                    // Multiply every value in the list by 10
                    for val in list.iter() {
                        val.set(val.get() * 10)
                    }

                    let mut vals = list.iter();
                    assert_eq!(vals.next().unwrap().get(), 130);
                    assert_eq!(vals.next().unwrap().get(), 50);
                    assert_eq!(vals.next().unwrap().get(), 30);
                    assert_eq!(vals.next(), None);
                    assert_eq!(vals.next(), None);
                })
            })
        })
    }
```

```txt
> cargo test

running 19 tests
test fifth::test::into_iter ... ok
test fifth::test::basics ... ok
test fifth::test::iter_mut ... ok
test fifth::test::iter ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test second::test::into_iter ... ok
test first::test::basics ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test fifth::test::miri_food ... ok
test silly2::test::cell ... ok
test third::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test silly1::test::walk_aboot ... ok
test silly2::test::elegance ... ok
test third::test::basics ... ok
test second::test::iter ... ok

test result: ok. 19 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out;
```

简单得就像递归饼！✨
