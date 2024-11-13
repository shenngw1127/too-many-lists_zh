# 7. 产品级的不安全的双向链接双端队列

我们终于成功了。我最大的敌人是[**`std::collections::LinkedList`**](https://github.com/rust-lang/rust/blob/master/library/alloc/src/collections/linked_list.rs)，即双向链表。

我试图摧毁它，但失败了。

我们的故事开始于2014年即将结束之时，我们正迅速接近Rust 1.0的发布，这是Rust 的第一个稳定版本。我发现自己正在负责照看`std::collections`，或者像我们当时亲切地称之为`libcollections`。

多年来，`libcollections`一直是每个人的可爱想法和模糊有用的东西的倾倒场。当Rust还是一种刚刚起步的实验性语言时，这一切都很好，但如果我的孩子们要离开巢穴并稳定下来，他们就必须证明自己的价值。

在那之前，我一直鼓励和培养他们所有人，但现在是时候让他们为自己的失败接受审判了。

我把爪子伸进基岩，为我最愚蠢的孩子们刻下墓碑。我在镇广场上竖立了一座可怕的纪念碑，供所有人观看：

[**杀死TreeMap、TreeSet、TrieMap、TrieSet、LruCache和EnumSet**](https://github.com/rust-lang/rust/pull/19955)

他们的命运已成定局，因为我的话是绝对的。其他的孩子们对我的残忍感到震惊，但他们还无法逃脱母亲的愤怒。我很快就带着另外两块墓碑回来了：

[**弃用BitSet和BitVec**](https://github.com/rust-lang/rust/pull/26034)

比特双胞胎（即`BitSet`和`BitVec`）比他们倒下的同伴更狡猾，但他们没有力量逃脱我。大多数人认为我的任务完成了，但我很快又完成了一项任务：

[**弃用VecMap**](https://github.com/rust-lang/rust/pull/26734)

`VecMap`曾试图通过隐身技术生存下来——它非常小而且无害！但这对于我所预见的未来中的`libcollections`来说还不够。

我勘察了这片土地，看到了剩下的东西：

- `Vec`和`VecDeque`： 热情而简单，计算的心脏。
- `HashMap`和`HashSet`： 强大而智慧，计算的大脑。
- `BTreeMap`和`BTreeSet`： 笨拙而必要，计算的肝脏。
- `BinaryHeap`：狡猾而灵巧，计算的脚踝。

我满意地点点头。简单又有效。我的工作是……

不，[`DList`](https://github.com/rust-lang/rust/blob/0a84308ebaaafb8fd89b2fd7c235198e3ec21384/src/libcollections/dlist.rs)，不可能！我以为你在那次悲惨的垃圾收集事件中死了！那绝对是一场意外，根本不是故意的！

他们假装死亡，并改了个新名字，但他们仍然是他们：`LinkedList`，计算领域的神秘而不可信任的阴谋家。

我把他们的罪行告诉了所有愿意听的人，但他们无动于衷。`LinkedList`是一个巧舌如簧的魔鬼，它让我身边的每个人都相信它是某种基本的、自然的计算数据结构。它甚至让C++相信它就是列表！

“没有LinkedList，怎么会有标准库？”

很简单！它很简单！

“这是不简单的不安全代码，因此将其放在标准库中是有意义的！”

GPU驱动程序和视频编解码器也是如此，`libcollections`是极简主义的！

但遗憾的是，当我被它的同类分散注意力时，`LinkedList`已经聚集了太多盟友并且变得太强大了。

我逃进实验室，试图设计某种能够与它抗衡并摧毁它的[邪恶克隆](https://github.com/contain-rs/linked-list)或[增强型机器人复制品](https://github.com/contain-rs/blist)，但我的资助资金用完了，因为我的研究“太过邪恶”或类似的废话。

`LinkedList`赢了。我被打败并被迫流放。

但你现在在这里。你已经走到这一步了。现在你肯定能理解`LinkedList`的放荡程度了！来吧，我会向你展示你需要知道的一切，帮助我一劳永逸地摧毁它——你需要知道的一切，以实现不安全的产品级质量的双链双端队列。

如何达到产品级的质量？好吧，我们将完全重写我古老的Rust 1.0链表箱，客观上比标准库中的更好。2015年稳定版Rust上带有游标的那个！2022年的标准库仍然没有的东西！

## 7.1 布局

让我们首先研究一下敌人的结构。双向链表在概念上很简单，但它就是这样欺骗和操纵你的。它和我们反复研究过的链表是一样的，但链接是双向的。链接加倍，邪恶加倍。

所以单向链表象这样（为了保持简洁，我将在展示布局时删除有关`Some`/`None`的内容）：

```txt
... -> (A, ptr) -> (B, ptr) -> ...
```

而双向链表象这样：

```txt
... <-> (ptr, A, ptr) <-> (ptr, B, ptr) <-> ...
```

这使您可以从任一方向遍历列表，或者使用[游标](https://doc.rust-lang.org/std/collections/struct.LinkedList.html#method.cursor_back_mut)来回查找。

为了实现这种灵活性，每个节点必须存储两倍的指针，并且每个操作都必须修改更多的指针。这是一个非常复杂的问题，很容易出错，所以我们将进行大量测试。

您可能还注意到，我故意没有画出列表的末尾。这是因为这是我们实现中真正有可选择的地方之一。我们的实现肯定需要有两个指针：一个指向列表的开头，一个指向列表的结尾。

我认为有两种值得注意的方法可以做到这一点：“传统”和“虚拟节点”。

传统方法是我们创建栈的简单扩展——只需将头指针和尾指针存储在栈上：

```txt
[ptr, ptr] <-> (ptr, A, ptr) <-> (ptr, B, ptr)
  ^                                        ^
  +----------------------------------------+
```

这很好，但它有一个缺点：极端情况。现在我们的列表有两处边缘，这意味着极端情况是原来的两倍。很容易忘记一个，从而产生严重的错误。

虚拟节点方法试图通过在我们的列表中添加一个额外的节点来消除这些极端情况，该节点不包含数据，但将两端连接在一起形成一个环：

```txt
[ptr] -> (ptr, ?DUMMY?, ptr) <-> (ptr, A, ptr) <-> (ptr, B, ptr)
           ^                                                 ^
           +-------------------------------------------------+ 
```

我觉得这非常令人满意且优雅。不幸的是，它有几个实际问题：

问题 1：额外的间接和分配，特别是对于空列表，它必须包含虚拟节点。潜在的解决方案包括：

- 在插入内容之前不要分配虚拟节点：简单有效，但它又增加了一些我们试图通过使用虚拟指针来避免的极端情况！
- 使用静态写时复制空单例虚拟节点，使用一些非常巧妙的方案，让写时复制检查搭载在正常检查上：看，我真的很想这样做，我真的很喜欢这个东西，但我们不能在这本书中走那条黑暗的道路。如果您想看到这种变态的东西，请阅读[`ThinVec`的源代码](https://docs.rs/thin-vec/0.2.4/src/thin_vec/lib.rs.html#319-325)。
- 将虚拟节点存储在栈上——在没有C++风格的移动构造函数的语言中是不切实际的。我敢肯定，我们可以在这里用[`pin`](https://doc.rust-lang.org/std/pin/index.html)做一些奇怪的事情，但我们不会这样做。

问题 2：虚拟节点中存储了什么值？当然，如果它是一个整数，那就没问题，但如果我们存储了一个满是`Box`的列表怎么办？我们可能无法初始化这个值！潜在的解决方案包括：

- 让每个节点都存储`Option<T>`：简单有效，但也臃肿烦人。
- 让每个节点都存储[`MaybeUninit<T>`](https://doc.rust-lang.org/std/mem/union.MaybeUninit.html)。可怕又烦人。
- 非常谨慎和巧妙的继承式类型双关语，因此虚拟节点不包含数据字段。这也很诱人，但非常危险和烦人。如果您想看到这种变态的东西，请阅读[`BTreeMap`的源代码](https://doc.rust-lang.org/1.55.0/src/alloc/collections/btree/node.rs.html#49-104)。

对于Rust这样的语言来说，问题确实大于便利性，所以我们将坚持传统方式的布局。我们将使用与上一章中不安全队列相同的基本设计：

（这一次我们不创建新文件，直接在`lib.rs`文件中写代码。）

```rust
pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
}

type Link<T> = *mut Node<T>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}
```

（现在我们已经开始写双链双端队列了，我们​​终于有权称自己为`LinkedList`，因为这才是真正的链表。）

这还不是一个真正的产品级质量级别的布局。这很好，但我们可以使用一些魔术技巧来更好地告诉Rust我们正在做什么。要做到这一点，我们需要更深入地了解。

## 7.2 变异性（`variance`）和幽灵数据（`PhantomData`）

现在搁置这个问题，以后再修复，会很烦人，所以我们现在就要做硬核布局的事情。

有五个可怕的因素会导致Rust集合不安全：

- [变异性（`variance`）](https://doc.rust-lang.org/nightly/nomicon/subtyping.html)
- [析构（`drop`）检查](https://doc.rust-lang.org/nightly/nomicon/dropck.html)
- [非空优化](https://doc.rust-lang.org/nightly/std/ptr/struct.NonNull.html)
- [`isize::MAX`分配内存规则](https://doc.rust-lang.org/nightly/nomicon/vec/vec-alloc.html)
- [零大小类型](https://doc.rust-lang.org/nightly/nomicon/vec/vec-zsts.html)

幸运的是，最后两个对我们来说不是问题。

我们可以把第三个变成我们的问题，但它带来的麻烦比它的价值更大——如果你选择了`LinkedList`，那么你已经100倍地放弃了内存效率的斗争。

第二个是我曾经坚持认为非常重要的事情，并且标准库会对其进行处理，但默认值是安全的，处理它的方式是不稳定的，您需要非常努力才能注意到默认值的局限性，所以，不要担心。

剩下的就只有变异性（`variance`）了。说实话，你可能也会放弃这个，但我仍然为自己作为集合库的作者感到自豪，所以我们要去做变异性的事情。

因此，令人惊讶的是：Rust具有子类型（`subtype`）。特别是，`&'big T`是`&'small T`的子类型。为什么？因为如果某些代码需要一个存在于程序某个特定区域的引用，那么通常可以给它一个存在时间更长的引用。直觉上这是正确的，对吧？

为什么这很重要？想象一下一些采用相同类型的两个值的代码：

```rust
fn take_two<T>(_val1: T, _val2: T) { }
```

这是一些非常无聊的代码，所以我们应该期望它能与`T=&u32`很好地协同工作，对吗？

```rust
#![allow(unused)]
fn main() {
    fn two_refs<'big: 'small, 'small>(
        big: &'big u32, 
        small: &'small u32,
    ) {
        take_two(big, small);
    }

    fn take_two<T>(_val1: T, _val2: T) { }
}
```

是的，编译成功了！

现在我们来玩点有趣的东西，把它包装在，哦，我不知道，`std::cell::Cell`

```rust
#![allow(unused)]
fn main() {
    use std::cell::Cell;

    fn two_refs<'big: 'small, 'small>(
        // NOTE: these two lines changed
        big: Cell<&'big u32>, 
        small: Cell<&'small u32>,
    ) {
        take_two(big, small);
    }

    fn take_two<T>(_val1: T, _val2: T) { }
}
```

```txt
> cargo build
error[E0623]: lifetime mismatch
 --> src/main.rs:7:19
  |
4 |     big: Cell<&'big u32>, 
  |               ---------
5 |     small: Cell<&'small u32>,
  |                 ----------- these two types are declared with different lifetimes...
6 | ) {
7 |     take_two(big, small);
  |                   ^^^^^ ...but data from `small` flows into `big` here
```

啊？我们并没有触及生命周期，为什么编译器现在这么生气？

嗯，生命周期“子类型”的东西一定非常简单，所以如果你用任何东西包装引用它就会失败，看看把它放到`Vec`类型里面，程序会不会也编译失败：

```rust
#![allow(unused)]
fn main() {
    fn two_refs<'big: 'small, 'small>(
        big: Vec<&'big u32>, 
        small: Vec<&'small u32>,
    ) {
        take_two(big, small);
    }

    fn take_two<T>(_val1: T, _val2: T) { }
}
```

```txt
    Finished dev [unoptimized + debuginfo] target(s) in 1.07s
     Running `target/debug/playground`
```

发现它也无法编译，等等……什么？？？居然成功了！`Vec`是有什么魔法吗？？？？？？

嗯，是的。但也不是。魔法一直存在于我们内心，而这种魔法就是 ✨变异性✨。

如果您想要了解所有细节，请阅读[nomicon关于子类型的章节](https://doc.rust-lang.org/nightly/nomicon/subtyping.html)，但基本上子类型并不总是安全的。特别是当涉及可变引用时，它并不安全，因为您可以使用诸如`mem::swap`之类的东西，然后突然出现悬垂指针！

“类似可变引用”的东西是不变的（`invariant`），这意味着它们会阻止子类型在其通用参数上发生。因此，为了安全起见，`&mut T`对`T`是不变的，而`Cell<T>`对`T`也是不变的，因为`&Cell<T>`基本上就是`&mut T`（因为内部可变性）。

几乎所有不是不变的（`invariant`）东西都是协变的（`covariant`），这只意味着子类型可以“穿过”它并继续正常工作（也有逆变的（`contravariant`）类型使子类型倒退，但它们非常罕见，没有人喜欢它们，所以我不会再提及它们）。

集合通常包含指向其数据的可变指针，因此您可能希望它们也是不变的，但实际上，它们不需要如此！由于Rust的所有权系统，`Vec<T>`在语义上等同于`T`，这意味着它可以安全地协变！

不幸的是，这个定义是不变的：

```rust
pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
}

type Link<T> = *mut Node<T>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}
```

（如果不处理这个定义中的`*mut Node<T>`，因为它的不变性（`invariant`）会引发很多问题，后面我们将寸步难行，所以必须现在就解决这个变异性问题。）

但是Rust实际上是怎样决定事物的变异性的呢？在1.0之前的美好时光里，我们只是让人们指定他们想要的变异性，结果……这简直是一场灾难！子类型和变异性真的很难理解，核心开发人员对基本术语的看法也大相径庭！因此，我们转向了“通过示例确定变异性”的方法：编译器只需查看您的字段并复制它们的变异性。如果存在任何分歧，那么不变性总是胜出，因为这是安全的。

那么我们的类型定义中什么让Rust感到抓狂呢？`*mut`！

Rust中的原始指针实际上只是试图让你做任何事情，但它们只有一个安全特性：因为大多数人不知道变异性和子类型是Rust中的东西，并且错误的协变会非常危险，而`*mut T`是不变的，因为很有可能它被用作“`&mut T`”。

对于花了很多时间用Rust编写集合的人来说，这非常烦人。这就是为什么当我创建[`std::ptr::NonNull`](https://doc.rust-lang.org/std/ptr/struct.NonNull.html)时，我添加了这个小魔法：

（原文）
> Unlike `*mut T`, `NonNull<T>` was chosen to be covariant over `T`. This makes it possible to use `NonNull<T>` when building covariant types, but introduces the risk of unsoundness if used in a type that shouldn’t actually be covariant.

（翻译）
> 与`*mut T`不同，`NonNull<T>` 被选为`T`的协变。这使得在构建协变类型时可以使用`NonNull<T>`，但如果在实际上不应该协变的类型中使用，则会引入不健全的风险。

但是，它的接口是围绕`*mut T`构建的，这到底是怎么回事！这难道不是魔法吗？让我们来看看：

```rust
pub struct NonNull<T> {
    pointer: *const T,
}

impl<T> NonNull<T> {
    pub unsafe fn new_unchecked(ptr: *mut T) -> Self {
        // SAFETY: the caller must guarantee that `ptr` is non-null.
        unsafe { NonNull { pointer: ptr as *const T } }
    }
}
```

不。这里没有魔法！`NonNull`只是滥用了`*const T`是协变的事实，并将其存储起来，在API边界上在`*mut T`之间来回转换，使其“看起来像”存储了一个`*mut T`。这就是整个技巧！这样Rust中的集合就做到了是协变的！这太糟糕了！所以我设计了这个“好指针类型”（`NonNull`）为你做了这件事！不客气！好好享受这个用来处理子类型的脚枪吧！

解决所有问题的方法是使用`NonNull`，然后如果您想再次拥有可空指针，请使用`Option<NonNull<T>>`。我们真的要费心去做这件事吗？

是的！这么写看上去很糟糕，但我们正在制作产品级的链表，所以我们要努力吃掉所有的蔬菜，用困难的方式做事（当然我们也可以只使用原始指针`*const T`并在任何地方进行强制转换，但我真的想看看这有多痛苦……在人体工程学的角度）。

所以这是我们的最终类型定义：

```rust
use std::ptr::NonNull;

// !!!This changed!!!
pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
}

type Link<T> = Option<NonNull<Node<T>>>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}
```

...等等，还有最后一件事。任何时候你做原始指针操作，你都应该添加一个幽灵数据来保护你的指针：

```rust
use std::marker::PhantomData;

pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    /// We semantically store values of T by-value.
    _boo: PhantomData<T>,
}
```

在这种情况下，我认为我们实际上并不需要[`PhantomData`](https://doc.rust-lang.org/std/marker/struct.PhantomData.html)，但任何时候您使用 `NonNull`（或者只是一般的原始指针），您应该始终添加它以确保安全，并向编译器和其他人清楚地说明您认为自己正在做什么。

`PhantomData`是一种为编译器提供额外“示例”字段的方法，该字段在概念上存在于您的类型中，但由于各种原因（间接、类型擦除等）而不存在。在我们的代码中使用了`NonNull`，因为我们声称我们的类型表现得“好像”它存储了一个值`T`，所以我们需要添加一个`PhantomData`来明确这一点。

标准库实际上还有其他原因这样做，因为它可以访问该死的[析构检查覆盖](https://doc.rust-lang.org/nightly/nomicon/dropck.html)，但该功能已被重新设计了很多次，以至于我实际上不知道`PhantomData`是否还适用于它。我仍然会永远地崇拜它，因为析构检查魔法已经刻在我的脑海里了！

（`Node`结构体确实存储了一个`T`，所以它不必这样做，不用使用`PhantomData`，耶！）

...好吧，我们现在真的完成了布局！开始实际的基本功能！

## 7.3 基本操作

好吧，这是这本书最烂的部分，也是我花了7年时间才写完这一章的原因！是时候把我们已经做过5遍的一大堆无聊的东西都看完了，但这些内容特别冗长冗长，因为是双向链表，所以我们必须每件事都做两遍，而且还要用到有这么多层嵌套的类型`Option<NonNull<Node<T>>>`！

让我们慢慢来，首先还是`new`方法

```rust
impl<T> LinkedList<T> {
    pub fn new() -> Self {
        Self {
            front: None,
            back: None,
            len: 0,
            _boo: PhantomData,
        }
    }
}
```

`PhantomData`是一种奇怪的类型，没有字段，因此您只需说出其类型名称即可创建一个。耸肩

```rust
    pub fn push_front(&mut self, elem: T) {
        // SAFETY: it's a linked-list, what do you want?
        unsafe {
            let new = NonNull::new_unchecked(Box::into_raw(Box::new(Node {
                front: None,
                back: None,
                elem,
            })));
            if let Some(old) = self.front {
                // Put the new front before the old one
                (*old).front = Some(new);
                (*new).back = Some(old);
            } else {
                // If there's no front, then we're the empty list and need 
                // to set the back too. Also here's some integrity checks
                // for testing, in case we mess up.
                debug_assert!(self.back.is_none());
                debug_assert!(self.front.is_none());
                debug_assert!(self.len == 0);
                self.back = Some(new);
            }
            self.front = Some(new);
            self.len += 1;
        }
    }
```

```txt
error[E0614]: type `NonNull<Node<T>>` cannot be dereferenced
  --> src\lib.rs:39:17
   |
39 |                 (*old).front = Some(new);
   |                 ^^^^^^
```

啊，是的，我真的很讨厌我的指针型孩子。我们需要使用`as_ptr`明确地从`NonNull`中获取原始指针，因为`DerefMut`是根据`&mut`定义的，我们不想将安全引用随机引入到我们的不安全代码中！

```rust
            (*old.as_ptr()).front = Some(new);
            (*new.as_ptr()).back = Some(old);
```

```txt
   Compiling linked-list v0.0.3
warning: field is never read: `elem`
  --> src\lib.rs:16:5
   |
16 |     elem: T,
   |     ^^^^^^^
   |
   = note: `#[warn(dead_code)]` on by default

warning: `linked-list` (lib) generated 1 warning (1 duplicate)
warning: `linked-list` (lib test) generated 1 warning
    Finished test [unoptimized + debuginfo] target(s) in 0.33s
```

很好，现在来看看`pop`（和`len`）方法：

```rust
    pub fn pop_front(&mut self) -> Option<T> {
        unsafe {
            // Only have to do stuff if there is a front node to pop.
            // Note that we don't need to mess around with `take` anymore
            // because everything is Copy and there are no dtors that will
            // run if we mess up... right? :) Riiiight? :)))
            self.front.map(|node| {
                // Bring the Box back to life so we can move out its value and
                // Drop it (Box continues to magically understand this for us).
                let boxed_node = Box::from_raw(node.as_ptr());
                let result = boxed_node.elem;

                // Make the next node into the new front.
                self.front = boxed_node.back;
                if let Some(new) = self.front {
                    // Cleanup its reference to the removed node
                    (*new.as_ptr()).front = None;
                } else {
                    // If the front is now null, then this list is now empty!
                    debug_assert!(self.len == 1);
                    self.back = None;
                }

                self.len -= 1;
                result
                // Box gets implicitly freed here, knows there is no T.
            })
        }
    }

    pub fn len(&self) -> usize {
        self.len
    }
```

```txt
   Compiling linked-list v0.0.3
    Finished dev [unoptimized + debuginfo] target(s) in 0.37s
```

对我来说这似乎是合法的，是时候写一个测试了！

```rust
#[cfg(test)]
mod test {
    use super::LinkedList;

    #[test]
    fn test_basic_front() {
        let mut list = LinkedList::new();

        // Try to break an empty list
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // Try to break a one item list
        list.push_front(10);
        assert_eq!(list.len(), 1);
        assert_eq!(list.pop_front(), Some(10));
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // Mess around
        list.push_front(10);
        assert_eq!(list.len(), 1);
        list.push_front(20);
        assert_eq!(list.len(), 2);
        list.push_front(30);
        assert_eq!(list.len(), 3);
        assert_eq!(list.pop_front(), Some(30));
        assert_eq!(list.len(), 2);
        list.push_front(40);
        assert_eq!(list.len(), 3);
        assert_eq!(list.pop_front(), Some(40));
        assert_eq!(list.len(), 2);
        assert_eq!(list.pop_front(), Some(20));
        assert_eq!(list.len(), 1);
        assert_eq!(list.pop_front(), Some(10));
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);
    }
}
```

```txt
> cargo test
   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 0.40s
     Running unittests src\lib.rs

running 1 test
test test::test_basic_front ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

哇，我们太完美了！

……对吧？

## 7.4 析构和恐慌安全

嘿，你注意到这个注释了吗：

（原文）

```rust
// Note that we don't need to mess around with `take` anymore
// because everything is Copy and there are no dtors that will
// run if we mess up... right? :) Riiiight? :)))
```

（翻译）

```rust
// 请注意，我们不再需要摆弄“take”了，
// 因为一切都是复制的（实现了`Copy`特征），而且如果我们搞砸了，没有dtor会
// 运行......对吧？:) 对吧？:)))
```

对吗？

抱歉，你忘了你正在读的书吗？当然错了！（有点。）

我们再看看`pop_front`的内部结构：

```rust
        // Bring the Box back to life so we can move out its value and
        // Drop it (Box continues to magically understand this for us).
        let boxed_node = Box::from_raw(node.as_ptr());
        let result = boxed_node.elem;

        // Make the next node into the new front.
        self.front = boxed_node.back;
        if let Some(new) = self.front {
            // Cleanup its reference to the removed node
            (*new.as_ptr()).front = None;
        } else {
            // If the front is now null, then this list is now empty!
            debug_assert!(self.len == 1);
            self.back = None;
        }

        self.len -= 1;
        result
        // Box gets implicitly freed here, knows there is no T.
```

你看到这个错误了吗？令人震惊的是，它实际上是这一行：

```rust
            debug_assert!(self.len == 1);
```

真的吗？我们为了测试写的完整性检查是个bug？？是的！！！好吧，如果我们正确实现了我们的集合，它就不应该是bug，但它可以将一些无害的事情（例如“哦，我们没有及时更新`len`”）变成一个可利用的内存安全bug！为什么？因为它会引发恐慌（`panic`）！大多数时候，您不必考虑或担心恐慌，但一旦您开始编写真正不安全的代码并随意使用“不变量”，您就需要对恐慌保持高度警惕！

我们必须讨论[异常安全](https://doc.rust-lang.org/nightly/nomicon/exception-safety.html)（又名恐慌安全即`panic safety`、又名放松安全即`unwind safety`，...）

所以事情是这样的：默认情况下，恐慌是解除的。解除只是一种奇特的说法，即“让每个函数立即返回”。你可能会想“好吧，如果每个人都返回，那么程序就要死了，那又何必在意呢？”，但你错了！

我们必须小心，原因有两个：析构函数在函数返回时运行，并且可以捕获展开。在这两种情况下，代码在恐慌之后仍可以继续运行，因此我们需要非常小心，并确保我们的不安全集合在可能发生恐慌时始终处于某种连贯状态，因为每个恐慌都是隐式的提前返回！

让我们想想当我们到达那一行时我们的集合处于什么状态：

我们在堆栈上有`boxed_node`，并且从中提取了元素。如果我们此时返回，`Box`将被丢弃，节点将被释放。你现在看到了吗？`self.back`仍然指向那个被释放的节点！一旦我们实现集合的其余部分并开始使用`self.back`进行操作，这可能会导致释放后使用！哎呀！

有趣的是，这行代码有类似的问题，但它更安全：

```rust
        self.len -= 1;
```

默认情况下，在调试版本中，Rust会检查下溢和上溢，并在发生溢出时发出恐慌。是的，每个算术运算都存在恐慌安全隐患！这个更安全的原因是，它发生在我们修复了所有不变量之后，所以它不会导致内存安全问题……只要我们不相信`len`是正确的，但是，如果我们下溢，它肯定是错误的，所以无论如何我们都会死！调试断言在某种意义上更糟糕，因为它可能会将小问题升级为严重问题！

我多次提到“不变量”这个术语，这是因为它对于恐慌安全来说是一个非常有用的概念！基本上，对于我们集合的外部观察者来说，我们始终坚持某些属性。对于`LinkedList`，其中之一就是列表中可访问的任何节点仍被分配和初始化。

在实现中，我们可以更灵活地暂时打破不变量，只要我们确保在任何人注意到之前修复它们即可。这实际上是Rust集合所有权和借用系统的“杀手级应用”之一：如果操作需要`&mut Self`，那么我们可以保证我们对集合拥有独占访问权，并且可以暂时打破不变量，因为我们知道没有人可以偷偷地破坏它。

也许最能体现这一点的是[`Vec::drain`](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.drain)，它实际上可以让你完全打破`Vec`的核心不变量，并开始将值从`Vec`的前面甚至中间移出。这是合理的，因为我们返回的`Drain`迭代器持有`Vec`的`&mut`，因此所有访问都受其控制！在`Drain`迭代器消失之前，没有人可以观察到`Vec`，然后它的析构函数可以在任何人注意到之前“修复”`Vec`，这是完美的——

[它并不完美](https://doc.rust-lang.org/nightly/nomicon/leaking.html#drain)。不幸的是，你[不能依赖你无法控制的代码中的析构函数来运行](https://doc.rust-lang.org/std/mem/fn.forget.html)，因此即使使用`Drain`，我们也需要做一些额外的工作来使我们的类型始终保持不变性，但以一种愚蠢的方式：[我们只是在开始时将`Vec`的长度设置为`0`](https://doc.rust-lang.org/std/mem/fn.forget.html)，所以如果有人泄漏了`Drain`，那么他们将拥有一个安全的`Vec`……但他们也会丢失一堆数据。你泄漏我？我泄漏你！以牙还牙！真正的正义！

对于你可以实际使用析构函数来实现恐慌安全的情况，请查看[`BinaryHeap::sift_up`案例研究](https://doc.rust-lang.org/nightly/nomicon/exception-safety.html#binaryheapsift_up)。

无论如何，我们不需要为`LinkedList`添加所有这些花哨的东西，我们只需要更加警惕我们在哪里破坏了不变量，我们相信/要求什么是正确的，并避免在棘手的任务中引入不必要的展开。

在这种情况下，我们有两个选项可以使我们的代码更加健壮：

- 更积极地使用`Option::take`之类的操作，因为它们更具“事务性”，并且倾向于保留不变量。
- 删除`debug_asserts`，并相信我们自己能够使用专用的“完整性检查”函数编写更好的测试，这些函数永远不会在用户代码中运行。

原则上我喜欢第一个选项，但它实际上对双向链表不太适用，因为所有内容都是双重冗余编码的。`Option::take`无法解决此问题，但将`debug_assert`向下移动一行可以。但说真的，为什么要让事情变得更难呢？让我们删除那些`debug_asserts`，并确保任何可能产生恐慌的地方都位于我们方法的开头或结尾，我们的不变量应该在那里保持不变。

（这样，将它们视为先决条件和后置条件可能更准确，但您确实应该尽可能将它们视为不变量！）

这是我们现在的完整实现：

```rust
use std::ptr::NonNull;
use std::marker::PhantomData;

pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<T>,
}

type Link<T> = Option<NonNull<Node<T>>>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}

impl<T> LinkedList<T> {
    pub fn new() -> Self {
        Self {
            front: None,
            back: None,
            len: 0,
            _boo: PhantomData,
        }
    }

    pub fn push_front(&mut self, elem: T) {
        // SAFETY: it's a linked-list, what do you want?
        unsafe {
            let new = NonNull::new_unchecked(Box::into_raw(Box::new(Node {
                front: None,
                back: None,
                elem,
            })));
            if let Some(old) = self.front {
                // Put the new front before the old one
                (*old.as_ptr()).front = Some(new);
                (*new.as_ptr()).back = Some(old);
            } else {
                // If there's no front, then we're the empty list and need 
                // to set the back too.
                self.back = Some(new);
            }
            // These things always happen!
            self.front = Some(new);
            self.len += 1;
        }
    }

    pub fn pop_front(&mut self) -> Option<T> {
        unsafe {
            // Only have to do stuff if there is a front node to pop.
            self.front.map(|node| {
                // Bring the Box back to life so we can move out its value and
                // Drop it (Box continues to magically understand this for us).
                let boxed_node = Box::from_raw(node.as_ptr());
                let result = boxed_node.elem;

                // Make the next node into the new front.
                self.front = boxed_node.back;
                if let Some(new) = self.front {
                    // Cleanup its reference to the removed node
                    (*new.as_ptr()).front = None;
                } else {
                    // If the front is now null, then this list is now empty!
                    self.back = None;
                }

                self.len -= 1;
                result
                // Box gets implicitly freed here, knows there is no T.
            })
        }
    }

    pub fn len(&self) -> usize {
        self.len
    }
}
```

这里什么地方会产生恐慌呢？好吧，说实话，要知道这一点，你需要成为Rust专家，但幸运的是，我就是！

我能看到的代码中唯一可能产生恐慌的地方（除非有人在启用`debug_asserts`的情况下重新编译`stdlib`，但你永远不应该这样做）是`Box::new`（用于内存不足的情况）和`len`的计算。所有这些东西都在我们的方法的最后或最开始，所以是的，我们很安全！

...`Box::new`能够产生恐慌让你感到惊讶吗？恐慌会让你感到如此！尝试保留这些不变量，这样你就不必担心了！

## 7.5 无聊的组合

好的，回到我们链表的处理清单！

首先让我们淘汰掉`Drop`，这用`pop`实现很简单：

```rust
impl<T> Drop for LinkedList<T> {
    fn drop(&mut self) {
        // Pop until we have to stop
        while let Some(_) = self.pop_front() { }
    }
}
```

我们必须填写一堆非常无聊的组合实现，如`front`、`front_mut`、`back`、`back_mut`、`iter`、`iter_mut`、`into_iter`……

你可以用宏或其他东西来实现它们，但老实说，这比复制粘贴更糟糕。我们只是要做很多复制粘贴。我已经非常仔细地设计了之前的押入/弹出实现，以便我们能够真正地交换前后，代码会做/说正确的事情！为痛苦的经历欢呼！（谈论节点的“上一个和下一个”很诱人，但我发现尽可能多地讨论“前面”和“后面”以避免错误是非常值得的。） 好吧，首先是前面即`front`：

```rust
    pub fn front(&self) -> Option<&T> {
        unsafe {
            self.front.map(|node| &(*node.as_ptr()).elem)
        }
    }
```

嘿，实际上，这本书真的很老了，Rust已经添加了一些不错的新东西，比如`?`运算符，它可以提前返回`Option::None`，这会让我们的代码更漂亮吗？

```rust
    pub fn front(&self) -> Option<&T> {
        unsafe {
            Some(&(*self.front?.as_ptr()).elem)
        }
    }
```

也许吧？对于这么简单的事情来说，这有点难以理解，而且上一节都是关于尽早返回对我们来说有多可怕，所以也许我们应该在这里更明确一点（所以我更坚持使用`map`的实现）。

关于`front_mut`：

```rust
    pub fn front_mut(&mut self) -> Option<&mut T> {
        unsafe {
            self.front.map(|node| &mut (*node.as_ptr()).elem)
        }
    }
```

我会在最后列出所有后面即`back`版本。（太无聊了，这里先省略吧！）

接下来是迭代器。与我们之前的所有列表不同，我们终于解锁了执行[`DoubleEndedIterator`](https://doc.rust-lang.org/std/iter/trait.DoubleEndedIterator.html)的能力，如果我们要追求产品级质量，我们也将实现[`ExactSizeIterator`](https://doc.rust-lang.org/std/iter/trait.ExactSizeIterator.html)。

因此，除了`next`和`size_hint`之外，我们还将支持`next_back`和`len`方法。

你们中警惕的人可能会注意到`IterMut`在双端迭代方面似乎更加粗略，但它实际上仍然健全！

...天哪，这将是很多的无聊重复代码。也许我真的应该写一个宏...不，不，那仍然是更糟糕的命运。

```rust
pub struct Iter<'a, T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<&'a T>,
}

impl<T> LinkedList<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { 
            front: self.front, 
            back: self.back,
            len: self.len,
            _boo: PhantomData,
        }
    }
}


impl<'a, T> IntoIterator for &'a LinkedList<T> {
    type IntoIter = Iter<'a, T>;
    type Item = &'a T;

    fn into_iter(self) -> Self::IntoIter {
        self.iter()
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    
    fn next(&mut self) -> Option<Self::Item> {
        // While self.front == self.back is a tempting condition to check here,
        // it won't do the right for yielding the last element! That sort of
        // thing only works for arrays because of "one-past-the-end" pointers.
        if self.len > 0 {
            // We could unwrap front, but this is safer and easier
            self.front.map(|node| unsafe {
                self.len -= 1;
                self.front = (*node.as_ptr()).back;
                &(*node.as_ptr()).elem
            })
        } else {
            None
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.len, Some(self.len))
    }
}

impl<'a, T> DoubleEndedIterator for Iter<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        if self.len > 0 {
            self.back.map(|node| unsafe {
                self.len -= 1;
                self.back = (*node.as_ptr()).front;
                &(*node.as_ptr()).elem
            })
        } else {
            None
        }
    }
}

impl<'a, T> ExactSizeIterator for Iter<'a, T> {
    fn len(&self) -> usize {
        self.len
    }
}
```

...这只是完成了链表的`.iter()`方法...

我们将`IterMut`的部分粘贴在最后，它实际上与上面的的代码完全相同，只是在很多地方加上了`mut`，我们先把`into_iter`的部分敲掉。幸运的是，我们仍然可以依靠我们久经考验的解决方案，只需让它包装我们的集合并使用`pop`实现`next`方法：

```rust
pub struct IntoIter<T> {
    list: LinkedList<T>,
}

impl<T> LinkedList<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter { 
            list: self
        }
    }
}

impl<T> IntoIterator for LinkedList<T> {
    type IntoIter = IntoIter<T>;
    type Item = T;

    fn into_iter(self) -> Self::IntoIter {
        self.into_iter()
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;

    fn next(&mut self) -> Option<Self::Item> {
        self.list.pop_front()
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.list.len, Some(self.list.len))
    }
}

impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        self.list.pop_back()
    }
}

impl<T> ExactSizeIterator for IntoIter<T> {
    fn len(&self) -> usize {
        self.list.len
    }
}
```

仍然是一堆样板，但至少是令人满意的样板。

好吧，下面是我们填写好了所有组合的全部代码：

```rust
use std::ptr::NonNull;
use std::marker::PhantomData;

pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<T>,
}

type Link<T> = Option<NonNull<Node<T>>>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}

pub struct Iter<'a, T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<&'a T>,
}

pub struct IterMut<'a, T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<&'a mut T>,
}

pub struct IntoIter<T> {
    list: LinkedList<T>,
}

impl<T> LinkedList<T> {
    pub fn new() -> Self {
        Self {
            front: None,
            back: None,
            len: 0,
            _boo: PhantomData,
        }
    }

    pub fn push_front(&mut self, elem: T) {
        // SAFETY: it's a linked-list, what do you want?
        unsafe {
            let new = NonNull::new_unchecked(Box::into_raw(Box::new(Node {
                front: None,
                back: None,
                elem,
            })));
            if let Some(old) = self.front {
                // Put the new front before the old one
                (*old.as_ptr()).front = Some(new);
                (*new.as_ptr()).back = Some(old);
            } else {
                // If there's no front, then we're the empty list and need 
                // to set the back too.
                self.back = Some(new);
            }
            // These things always happen!
            self.front = Some(new);
            self.len += 1;
        }
    }

    pub fn push_back(&mut self, elem: T) {
        // SAFETY: it's a linked-list, what do you want?
        unsafe {
            let new = NonNull::new_unchecked(Box::into_raw(Box::new(Node {
                back: None,
                front: None,
                elem,
            })));
            if let Some(old) = self.back {
                // Put the new back before the old one
                (*old.as_ptr()).back = Some(new);
                (*new.as_ptr()).front = Some(old);
            } else {
                // If there's no back, then we're the empty list and need 
                // to set the front too.
                self.front = Some(new);
            }
            // These things always happen!
            self.back = Some(new);
            self.len += 1;
        }
    }

    pub fn pop_front(&mut self) -> Option<T> {
        unsafe {
            // Only have to do stuff if there is a front node to pop.
            self.front.map(|node| {
                // Bring the Box back to life so we can move out its value and
                // Drop it (Box continues to magically understand this for us).
                let boxed_node = Box::from_raw(node.as_ptr());
                let result = boxed_node.elem;

                // Make the next node into the new front.
                self.front = boxed_node.back;
                if let Some(new) = self.front {
                    // Cleanup its reference to the removed node
                    (*new.as_ptr()).front = None;
                } else {
                    // If the front is now null, then this list is now empty!
                    self.back = None;
                }

                self.len -= 1;
                result
                // Box gets implicitly freed here, knows there is no T.
            })
        }
    }

    pub fn pop_back(&mut self) -> Option<T> {
        unsafe {
            // Only have to do stuff if there is a back node to pop.
            self.back.map(|node| {
                // Bring the Box front to life so we can move out its value and
                // Drop it (Box continues to magically understand this for us).
                let boxed_node = Box::from_raw(node.as_ptr());
                let result = boxed_node.elem;

                // Make the next node into the new back.
                self.back = boxed_node.front;
                if let Some(new) = self.back {
                    // Cleanup its reference to the removed node
                    (*new.as_ptr()).back = None;
                } else {
                    // If the back is now null, then this list is now empty!
                    self.front = None;
                }

                self.len -= 1;
                result
                // Box gets implicitly freed here, knows there is no T.
            })
        }
    }

    pub fn front(&self) -> Option<&T> {
        unsafe {
            self.front.map(|node| &(*node.as_ptr()).elem)
        }
    }

    pub fn front_mut(&mut self) -> Option<&mut T> {
        unsafe {
            self.front.map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn back(&self) -> Option<&T> {
        unsafe {
            self.back.map(|node| &(*node.as_ptr()).elem)
        }
    }

    pub fn back_mut(&mut self) -> Option<&mut T> {
        unsafe {
            self.back.map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn len(&self) -> usize {
        self.len
    }

    pub fn iter(&self) -> Iter<T> {
        Iter { 
            front: self.front, 
            back: self.back,
            len: self.len,
            _boo: PhantomData,
        }
    }

    pub fn iter_mut(&mut self) -> IterMut<T> {
        IterMut { 
            front: self.front, 
            back: self.back,
            len: self.len,
            _boo: PhantomData,
        }
    }

    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter { 
            list: self
        }
    }
}

impl<T> Drop for LinkedList<T> {
    fn drop(&mut self) {
        // Pop until we have to stop
        while let Some(_) = self.pop_front() { }
    }
}

impl<'a, T> IntoIterator for &'a LinkedList<T> {
    type IntoIter = Iter<'a, T>;
    type Item = &'a T;

    fn into_iter(self) -> Self::IntoIter {
        self.iter()
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        // While self.front == self.back is a tempting condition to check here,
        // it won't do the right for yielding the last element! That sort of
        // thing only works for arrays because of "one-past-the-end" pointers.
        if self.len > 0 {
            // We could unwrap front, but this is safer and easier
            self.front.map(|node| unsafe {
                self.len -= 1;
                self.front = (*node.as_ptr()).back;
                &(*node.as_ptr()).elem
            })
        } else {
            None
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.len, Some(self.len))
    }
}

impl<'a, T> DoubleEndedIterator for Iter<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        if self.len > 0 {
            self.back.map(|node| unsafe {
                self.len -= 1;
                self.back = (*node.as_ptr()).front;
                &(*node.as_ptr()).elem
            })
        } else {
            None
        }
    }
}

impl<'a, T> ExactSizeIterator for Iter<'a, T> {
    fn len(&self) -> usize {
        self.len
    }
}

impl<'a, T> IntoIterator for &'a mut LinkedList<T> {
    type IntoIter = IterMut<'a, T>;
    type Item = &'a mut T;

    fn into_iter(self) -> Self::IntoIter {
        self.iter_mut()
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        // While self.front == self.back is a tempting condition to check here,
        // it won't do the right for yielding the last element! That sort of
        // thing only works for arrays because of "one-past-the-end" pointers.
        if self.len > 0 {
            // We could unwrap front, but this is safer and easier
            self.front.map(|node| unsafe {
                self.len -= 1;
                self.front = (*node.as_ptr()).back;
                &mut (*node.as_ptr()).elem
            })
        } else {
            None
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.len, Some(self.len))
    }
}

impl<'a, T> DoubleEndedIterator for IterMut<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        if self.len > 0 {
            self.back.map(|node| unsafe {
                self.len -= 1;
                self.back = (*node.as_ptr()).front;
                &mut (*node.as_ptr()).elem
            })
        } else {
            None
        }
    }
}

impl<'a, T> ExactSizeIterator for IterMut<'a, T> {
    fn len(&self) -> usize {
        self.len
    }
}

impl<T> IntoIterator for LinkedList<T> {
    type IntoIter = IntoIter<T>;
    type Item = T;

    fn into_iter(self) -> Self::IntoIter {
        self.into_iter()
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;

    fn next(&mut self) -> Option<Self::Item> {
        self.list.pop_front()
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.list.len, Some(self.list.len))
    }
}

impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        self.list.pop_back()
    }
}

impl<T> ExactSizeIterator for IntoIter<T> {
    fn len(&self) -> usize {
        self.list.len
    }
}


#[cfg(test)]
mod test {
    use super::LinkedList;

    #[test]
    fn test_basic_front() {
        let mut list = LinkedList::new();

        // Try to break an empty list
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // Try to break a one item list
        list.push_front(10);
        assert_eq!(list.len(), 1);
        assert_eq!(list.pop_front(), Some(10));
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // Mess around
        list.push_front(10);
        assert_eq!(list.len(), 1);
        list.push_front(20);
        assert_eq!(list.len(), 2);
        list.push_front(30);
        assert_eq!(list.len(), 3);
        assert_eq!(list.pop_front(), Some(30));
        assert_eq!(list.len(), 2);
        list.push_front(40);
        assert_eq!(list.len(), 3);
        assert_eq!(list.pop_front(), Some(40));
        assert_eq!(list.len(), 2);
        assert_eq!(list.pop_front(), Some(20));
        assert_eq!(list.len(), 1);
        assert_eq!(list.pop_front(), Some(10));
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);
    }
}
```

## 7.6 填充随机位

嘿，你说过你想要达到产品级质量，不是吗？

这里还有一些可以随意添加进去以成为“好”集合的东西：

```rust
impl<T> LinkedList<T> {
    pub fn is_empty(&self) -> bool {
        self.len == 0
    }

    pub fn clear(&mut self) {
        // Oh look it's drop again
        while let Some(_) = self.pop_front() { }
    }
}
```

现在，我们需要实现一系列大家都期望的特征：

```rust
impl<T> Default for LinkedList<T> {
    fn default() -> Self {
        Self::new()
    }
}

impl<T: Clone> Clone for LinkedList<T> {
    fn clone(&self) -> Self {
        let mut new_list = Self::new();
        for item in self {
            new_list.push_back(item.clone());
        }
        new_list
    }
}

impl<T> Extend<T> for LinkedList<T> {
    fn extend<I: IntoIterator<Item = T>>(&mut self, iter: I) {
        for item in iter {
            self.push_back(item);
        }
    }
}

impl<T> FromIterator<T> for LinkedList<T> {
    fn from_iter<I: IntoIterator<Item = T>>(iter: I) -> Self {
        let mut list = Self::new();
        list.extend(iter);
        list
    }
}

impl<T: Debug> Debug for LinkedList<T> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_list().entries(self).finish()
    }
}

impl<T: PartialEq> PartialEq for LinkedList<T> {
    fn eq(&self, other: &Self) -> bool {
        self.len() == other.len() && self.iter().eq(other)
    }

    fn ne(&self, other: &Self) -> bool {
        self.len() != other.len() || self.iter().ne(other)
    }
}

impl<T: Eq> Eq for LinkedList<T> { }

impl<T: PartialOrd> PartialOrd for LinkedList<T> {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        self.iter().partial_cmp(other)
    }
}

impl<T: Ord> Ord for LinkedList<T> {
    fn cmp(&self, other: &Self) -> Ordering {
        self.iter().cmp(other)
    }
}

impl<T: Hash> Hash for LinkedList<T> {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.len().hash(state);
        for item in self {
            item.hash(state);
        }
    }
}
```

我绝对是从头开始编写所有这些的，而不是仅仅复制标准库的实现。因为它们非常有趣，而且我绝对记得手动实现`Hash`特征的微妙之处。是的，这是我一直在思考的事情…… 好吧，这里实际上有几件事值得注意。

首先，令人讨厌的命名空间冲突。无论出于什么原因，标准库现在都有名为`Hash`和`Debug`的宏，因此如果您没有导入特征，您将收到关于宏的真正神秘的错误，而不是正确的“缺失特征”。

另一个有趣的事情是哈希本身。你看到我们如何对长度进行哈希了吗？这实际上非常重要！如果集合没有对长度进行哈希，[它们可能会意外地使自己容易受到前缀冲突的影响](https://doc.rust-lang.org/std/hash/trait.Hash.html#prefix-collisions)。例如，`[“he”、“llo”]`与`[“hello”]`有什么区别？如果没有人对长度或其他“分隔符”进行哈希处理，那就什么都没有了！让哈希冲突太容易意外或恶意发生可能会导致严重的悲伤，所以就这么做吧！

好吧，这是我们当前的代码：

```rust
use std::cmp::Ordering;
use std::fmt::{self, Debug};
use std::hash::{Hash, Hasher};
use std::iter::FromIterator;
use std::ptr::NonNull;
use std::marker::PhantomData;

pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<T>,
}

type Link<T> = Option<NonNull<Node<T>>>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}

pub struct Iter<'a, T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<&'a T>,
}

pub struct IterMut<'a, T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<&'a mut T>,
}

pub struct IntoIter<T> {
    list: LinkedList<T>,
}

impl<T> LinkedList<T> {
    pub fn new() -> Self {
        Self {
            front: None,
            back: None,
            len: 0,
            _boo: PhantomData,
        }
    }

    pub fn push_front(&mut self, elem: T) {
        // SAFETY: it's a linked-list, what do you want?
        unsafe {
            let new = NonNull::new_unchecked(Box::into_raw(Box::new(Node {
                front: None,
                back: None,
                elem,
            })));
            if let Some(old) = self.front {
                // Put the new front before the old one
                (*old.as_ptr()).front = Some(new);
                (*new.as_ptr()).back = Some(old);
            } else {
                // If there's no front, then we're the empty list and need 
                // to set the back too.
                self.back = Some(new);
            }
            // These things always happen!
            self.front = Some(new);
            self.len += 1;
        }
    }

    pub fn push_back(&mut self, elem: T) {
        // SAFETY: it's a linked-list, what do you want?
        unsafe {
            let new = NonNull::new_unchecked(Box::into_raw(Box::new(Node {
                back: None,
                front: None,
                elem,
            })));
            if let Some(old) = self.back {
                // Put the new back before the old one
                (*old.as_ptr()).back = Some(new);
                (*new.as_ptr()).front = Some(old);
            } else {
                // If there's no back, then we're the empty list and need 
                // to set the front too.
                self.front = Some(new);
            }
            // These things always happen!
            self.back = Some(new);
            self.len += 1;
        }
    }

    pub fn pop_front(&mut self) -> Option<T> {
        unsafe {
            // Only have to do stuff if there is a front node to pop.
            self.front.map(|node| {
                // Bring the Box back to life so we can move out its value and
                // Drop it (Box continues to magically understand this for us).
                let boxed_node = Box::from_raw(node.as_ptr());
                let result = boxed_node.elem;

                // Make the next node into the new front.
                self.front = boxed_node.back;
                if let Some(new) = self.front {
                    // Cleanup its reference to the removed node
                    (*new.as_ptr()).front = None;
                } else {
                    // If the front is now null, then this list is now empty!
                    self.back = None;
                }

                self.len -= 1;
                result
                // Box gets implicitly freed here, knows there is no T.
            })
        }
    }

    pub fn pop_back(&mut self) -> Option<T> {
        unsafe {
            // Only have to do stuff if there is a back node to pop.
            self.back.map(|node| {
                // Bring the Box front to life so we can move out its value and
                // Drop it (Box continues to magically understand this for us).
                let boxed_node = Box::from_raw(node.as_ptr());
                let result = boxed_node.elem;

                // Make the next node into the new back.
                self.back = boxed_node.front;
                if let Some(new) = self.back {
                    // Cleanup its reference to the removed node
                    (*new.as_ptr()).back = None;
                } else {
                    // If the back is now null, then this list is now empty!
                    self.front = None;
                }

                self.len -= 1;
                result
                // Box gets implicitly freed here, knows there is no T.
            })
        }
    }

    pub fn front(&self) -> Option<&T> {
        unsafe {
            self.front.map(|node| &(*node.as_ptr()).elem)
        }
    }

    pub fn front_mut(&mut self) -> Option<&mut T> {
        unsafe {
            self.front.map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn back(&self) -> Option<&T> {
        unsafe {
            self.back.map(|node| &(*node.as_ptr()).elem)
        }
    }

    pub fn back_mut(&mut self) -> Option<&mut T> {
        unsafe {
            self.back.map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn len(&self) -> usize {
        self.len
    }

    pub fn is_empty(&self) -> bool {
        self.len == 0
    }

    pub fn clear(&mut self) {
        // Oh look it's drop again
        while let Some(_) = self.pop_front() { }
    }

    pub fn iter(&self) -> Iter<T> {
        Iter { 
            front: self.front, 
            back: self.back,
            len: self.len,
            _boo: PhantomData,
        }
    }

    pub fn iter_mut(&mut self) -> IterMut<T> {
        IterMut { 
            front: self.front, 
            back: self.back,
            len: self.len,
            _boo: PhantomData,
        }
    }

    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter { 
            list: self
        }
    }
}

impl<T> Drop for LinkedList<T> {
    fn drop(&mut self) {
        // Pop until we have to stop
        while let Some(_) = self.pop_front() { }
    }
}

impl<T> Default for LinkedList<T> {
    fn default() -> Self {
        Self::new()
    }
}

impl<T: Clone> Clone for LinkedList<T> {
    fn clone(&self) -> Self {
        let mut new_list = Self::new();
        for item in self {
            new_list.push_back(item.clone());
        }
        new_list
    }
}

impl<T> Extend<T> for LinkedList<T> {
    fn extend<I: IntoIterator<Item = T>>(&mut self, iter: I) {
        for item in iter {
            self.push_back(item);
        }
    }
}

impl<T> FromIterator<T> for LinkedList<T> {
    fn from_iter<I: IntoIterator<Item = T>>(iter: I) -> Self {
        let mut list = Self::new();
        list.extend(iter);
        list
    }
}

impl<T: Debug> Debug for LinkedList<T> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_list().entries(self).finish()
    }
}

impl<T: PartialEq> PartialEq for LinkedList<T> {
    fn eq(&self, other: &Self) -> bool {
        self.len() == other.len() && self.iter().eq(other)
    }

    fn ne(&self, other: &Self) -> bool {
        self.len() != other.len() || self.iter().ne(other)
    }
}

impl<T: Eq> Eq for LinkedList<T> { }

impl<T: PartialOrd> PartialOrd for LinkedList<T> {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        self.iter().partial_cmp(other)
    }
}

impl<T: Ord> Ord for LinkedList<T> {
    fn cmp(&self, other: &Self) -> Ordering {
        self.iter().cmp(other)
    }
}

impl<T: Hash> Hash for LinkedList<T> {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.len().hash(state);
        for item in self {
            item.hash(state);
        }
    }
}

impl<'a, T> IntoIterator for &'a LinkedList<T> {
    type IntoIter = Iter<'a, T>;
    type Item = &'a T;

    fn into_iter(self) -> Self::IntoIter {
        self.iter()
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        // While self.front == self.back is a tempting condition to check here,
        // it won't do the right for yielding the last element! That sort of
        // thing only works for arrays because of "one-past-the-end" pointers.
        if self.len > 0 {
            // We could unwrap front, but this is safer and easier
            self.front.map(|node| unsafe {
                self.len -= 1;
                self.front = (*node.as_ptr()).back;
                &(*node.as_ptr()).elem
            })
        } else {
            None
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.len, Some(self.len))
    }
}

impl<'a, T> DoubleEndedIterator for Iter<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        if self.len > 0 {
            self.back.map(|node| unsafe {
                self.len -= 1;
                self.back = (*node.as_ptr()).front;
                &(*node.as_ptr()).elem
            })
        } else {
            None
        }
    }
}

impl<'a, T> ExactSizeIterator for Iter<'a, T> {
    fn len(&self) -> usize {
        self.len
    }
}

impl<'a, T> IntoIterator for &'a mut LinkedList<T> {
    type IntoIter = IterMut<'a, T>;
    type Item = &'a mut T;

    fn into_iter(self) -> Self::IntoIter {
        self.iter_mut()
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        // While self.front == self.back is a tempting condition to check here,
        // it won't do the right for yielding the last element! That sort of
        // thing only works for arrays because of "one-past-the-end" pointers.
        if self.len > 0 {
            // We could unwrap front, but this is safer and easier
            self.front.map(|node| unsafe {
                self.len -= 1;
                self.front = (*node.as_ptr()).back;
                &mut (*node.as_ptr()).elem
            })
        } else {
            None
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.len, Some(self.len))
    }
}

impl<'a, T> DoubleEndedIterator for IterMut<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        if self.len > 0 {
            self.back.map(|node| unsafe {
                self.len -= 1;
                self.back = (*node.as_ptr()).front;
                &mut (*node.as_ptr()).elem
            })
        } else {
            None
        }
    }
}

impl<'a, T> ExactSizeIterator for IterMut<'a, T> {
    fn len(&self) -> usize {
        self.len
    }
}

impl<T> IntoIterator for LinkedList<T> {
    type IntoIter = IntoIter<T>;
    type Item = T;

    fn into_iter(self) -> Self::IntoIter {
        self.into_iter()
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;

    fn next(&mut self) -> Option<Self::Item> {
        self.list.pop_front()
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.list.len, Some(self.list.len))
    }
}

impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        self.list.pop_back()
    }
}

impl<T> ExactSizeIterator for IntoIter<T> {
    fn len(&self) -> usize {
        self.list.len
    }
}


#[cfg(test)]
mod test {
    use super::LinkedList;

    #[test]
    fn test_basic_front() {
        let mut list = LinkedList::new();

        // Try to break an empty list
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // Try to break a one item list
        list.push_front(10);
        assert_eq!(list.len(), 1);
        assert_eq!(list.pop_front(), Some(10));
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // Mess around
        list.push_front(10);
        assert_eq!(list.len(), 1);
        list.push_front(20);
        assert_eq!(list.len(), 2);
        list.push_front(30);
        assert_eq!(list.len(), 3);
        assert_eq!(list.pop_front(), Some(30));
        assert_eq!(list.len(), 2);
        list.push_front(40);
        assert_eq!(list.len(), 3);
        assert_eq!(list.pop_front(), Some(40));
        assert_eq!(list.len(), 2);
        assert_eq!(list.pop_front(), Some(20));
        assert_eq!(list.len(), 1);
        assert_eq!(list.pop_front(), Some(10));
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);
    }
}
```

## 7.7 测试

好吧，我推迟了一段时间的测试，因为我们都知道我们现在是Rust大师，我们不会再犯错误了！另外，这只是对旧板条箱的重写，所以我已经进行了所有测试。它们是测试，您已经见过很多测试了。它们在这里：

```rust
#[cfg(test)]
mod test {
    use super::LinkedList;

    fn generate_test() -> LinkedList<i32> {
        list_from(&[0, 1, 2, 3, 4, 5, 6])
    }

    fn list_from<T: Clone>(v: &[T]) -> LinkedList<T> {
        v.iter().map(|x| (*x).clone()).collect()
    }

    #[test]
    fn test_basic_front() {
        let mut list = LinkedList::new();

        // Try to break an empty list
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // Try to break a one item list
        list.push_front(10);
        assert_eq!(list.len(), 1);
        assert_eq!(list.pop_front(), Some(10));
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // Mess around
        list.push_front(10);
        assert_eq!(list.len(), 1);
        list.push_front(20);
        assert_eq!(list.len(), 2);
        list.push_front(30);
        assert_eq!(list.len(), 3);
        assert_eq!(list.pop_front(), Some(30));
        assert_eq!(list.len(), 2);
        list.push_front(40);
        assert_eq!(list.len(), 3);
        assert_eq!(list.pop_front(), Some(40));
        assert_eq!(list.len(), 2);
        assert_eq!(list.pop_front(), Some(20));
        assert_eq!(list.len(), 1);
        assert_eq!(list.pop_front(), Some(10));
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);
    }

    #[test]
    fn test_basic() {
        let mut m = LinkedList::new();
        assert_eq!(m.pop_front(), None);
        assert_eq!(m.pop_back(), None);
        assert_eq!(m.pop_front(), None);
        m.push_front(1);
        assert_eq!(m.pop_front(), Some(1));
        m.push_back(2);
        m.push_back(3);
        assert_eq!(m.len(), 2);
        assert_eq!(m.pop_front(), Some(2));
        assert_eq!(m.pop_front(), Some(3));
        assert_eq!(m.len(), 0);
        assert_eq!(m.pop_front(), None);
        m.push_back(1);
        m.push_back(3);
        m.push_back(5);
        m.push_back(7);
        assert_eq!(m.pop_front(), Some(1));

        let mut n = LinkedList::new();
        n.push_front(2);
        n.push_front(3);
        {
            assert_eq!(n.front().unwrap(), &3);
            let x = n.front_mut().unwrap();
            assert_eq!(*x, 3);
            *x = 0;
        }
        {
            assert_eq!(n.back().unwrap(), &2);
            let y = n.back_mut().unwrap();
            assert_eq!(*y, 2);
            *y = 1;
        }
        assert_eq!(n.pop_front(), Some(0));
        assert_eq!(n.pop_front(), Some(1));
    }

    #[test]
    fn test_iterator() {
        let m = generate_test();
        for (i, elt) in m.iter().enumerate() {
            assert_eq!(i as i32, *elt);
        }
        let mut n = LinkedList::new();
        assert_eq!(n.iter().next(), None);
        n.push_front(4);
        let mut it = n.iter();
        assert_eq!(it.size_hint(), (1, Some(1)));
        assert_eq!(it.next().unwrap(), &4);
        assert_eq!(it.size_hint(), (0, Some(0)));
        assert_eq!(it.next(), None);
    }

    #[test]
    fn test_iterator_double_end() {
        let mut n = LinkedList::new();
        assert_eq!(n.iter().next(), None);
        n.push_front(4);
        n.push_front(5);
        n.push_front(6);
        let mut it = n.iter();
        assert_eq!(it.size_hint(), (3, Some(3)));
        assert_eq!(it.next().unwrap(), &6);
        assert_eq!(it.size_hint(), (2, Some(2)));
        assert_eq!(it.next_back().unwrap(), &4);
        assert_eq!(it.size_hint(), (1, Some(1)));
        assert_eq!(it.next_back().unwrap(), &5);
        assert_eq!(it.next_back(), None);
        assert_eq!(it.next(), None);
    }

    #[test]
    fn test_rev_iter() {
        let m = generate_test();
        for (i, elt) in m.iter().rev().enumerate() {
            assert_eq!(6 - i as i32, *elt);
        }
        let mut n = LinkedList::new();
        assert_eq!(n.iter().rev().next(), None);
        n.push_front(4);
        let mut it = n.iter().rev();
        assert_eq!(it.size_hint(), (1, Some(1)));
        assert_eq!(it.next().unwrap(), &4);
        assert_eq!(it.size_hint(), (0, Some(0)));
        assert_eq!(it.next(), None);
    }

    #[test]
    fn test_mut_iter() {
        let mut m = generate_test();
        let mut len = m.len();
        for (i, elt) in m.iter_mut().enumerate() {
            assert_eq!(i as i32, *elt);
            len -= 1;
        }
        assert_eq!(len, 0);
        let mut n = LinkedList::new();
        assert!(n.iter_mut().next().is_none());
        n.push_front(4);
        n.push_back(5);
        let mut it = n.iter_mut();
        assert_eq!(it.size_hint(), (2, Some(2)));
        assert!(it.next().is_some());
        assert!(it.next().is_some());
        assert_eq!(it.size_hint(), (0, Some(0)));
        assert!(it.next().is_none());
    }

    #[test]
    fn test_iterator_mut_double_end() {
        let mut n = LinkedList::new();
        assert!(n.iter_mut().next_back().is_none());
        n.push_front(4);
        n.push_front(5);
        n.push_front(6);
        let mut it = n.iter_mut();
        assert_eq!(it.size_hint(), (3, Some(3)));
        assert_eq!(*it.next().unwrap(), 6);
        assert_eq!(it.size_hint(), (2, Some(2)));
        assert_eq!(*it.next_back().unwrap(), 4);
        assert_eq!(it.size_hint(), (1, Some(1)));
        assert_eq!(*it.next_back().unwrap(), 5);
        assert!(it.next_back().is_none());
        assert!(it.next().is_none());
    }

    #[test]
    fn test_eq() {
        let mut n: LinkedList<u8> = list_from(&[]);
        let mut m = list_from(&[]);
        assert!(n == m);
        n.push_front(1);
        assert!(n != m);
        m.push_back(1);
        assert!(n == m);

        let n = list_from(&[2, 3, 4]);
        let m = list_from(&[1, 2, 3]);
        assert!(n != m);
    }

    #[test]
    fn test_ord() {
        let n = list_from(&[]);
        let m = list_from(&[1, 2, 3]);
        assert!(n < m);
        assert!(m > n);
        assert!(n <= n);
        assert!(n >= n);
    }

    #[test]
    fn test_ord_nan() {
        let nan = 0.0f64 / 0.0;
        let n = list_from(&[nan]);
        let m = list_from(&[nan]);
        assert!(!(n < m));
        assert!(!(n > m));
        assert!(!(n <= m));
        assert!(!(n >= m));

        let n = list_from(&[nan]);
        let one = list_from(&[1.0f64]);
        assert!(!(n < one));
        assert!(!(n > one));
        assert!(!(n <= one));
        assert!(!(n >= one));

        let u = list_from(&[1.0f64, 2.0, nan]);
        let v = list_from(&[1.0f64, 2.0, 3.0]);
        assert!(!(u < v));
        assert!(!(u > v));
        assert!(!(u <= v));
        assert!(!(u >= v));

        let s = list_from(&[1.0f64, 2.0, 4.0, 2.0]);
        let t = list_from(&[1.0f64, 2.0, 3.0, 2.0]);
        assert!(!(s < t));
        assert!(s > one);
        assert!(!(s <= one));
        assert!(s >= one);
    }

    #[test]
    fn test_debug() {
        let list: LinkedList<i32> = (0..10).collect();
        assert_eq!(format!("{:?}", list), "[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]");

        let list: LinkedList<&str> = vec!["just", "one", "test", "more"]
            .iter().copied()
            .collect();
        assert_eq!(format!("{:?}", list), r#"["just", "one", "test", "more"]"#);
    }

    #[test]
    fn test_hashmap() {
        // Check that HashMap works with this as a key

        let list1: LinkedList<i32> = (0..10).collect();
        let list2: LinkedList<i32> = (1..11).collect();
        let mut map = std::collections::HashMap::new();

        assert_eq!(map.insert(list1.clone(), "list1"), None);
        assert_eq!(map.insert(list2.clone(), "list2"), None);

        assert_eq!(map.len(), 2);

        assert_eq!(map.get(&list1), Some(&"list1"));
        assert_eq!(map.get(&list2), Some(&"list2"));

        assert_eq!(map.remove(&list1), Some("list1"));
        assert_eq!(map.remove(&list2), Some("list2"));

        assert!(map.is_empty());
    }
}
```

现在到了关键时刻：

```txt
cargo test
    Finished test [unoptimized + debuginfo] target(s) in 0.00s
     Running unittests src\lib.rs

running 12 tests
test test::test_basic ... ok
test test::test_basic_front ... ok
test test::test_eq ... ok
test test::test_iterator ... ok
test test::test_iterator_mut_double_end ... ok
test test::test_ord_nan ... ok
test test::test_iterator_double_end ... ok
test test::test_mut_iter ... ok
test test::test_rev_iter ... ok
test test::test_hashmap ... ok
test test::test_ord ... ok
test test::test_debug ... ok

test result: ok. 12 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

```txt
$env:MIRIFLAGS="-Zmiri-tag-raw-pointers"
cargo +nightly-2022-02-14 miri test
   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 0.35s
     Running unittests src\lib.rs

running 12 tests
test test::test_basic ... ok
test test::test_basic_front ... ok
test test::test_debug ... ok
test test::test_eq ... ok
test test::test_hashmap ... ok
test test::test_iterator ... ok
test test::test_iterator_double_end ... ok
test test::test_iterator_mut_double_end ... ok
test test::test_mut_iter ... ok
test test::test_ord ... ok
test test::test_ord_nan ... ok
test test::test_rev_iter ... ok

test result: ok. 12 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

😭

我们做到了，我们实际上没有搞砸。这不是诡计！我们所有的练习和训练终于值得了，我们终于写出了好的代码！！！

现在，所有那些垃圾组合方法都解决了，我们可以回到有趣的东西上了！

## 7.8 `Send`、`Sync`和编译测试

好吧，实际上我们还有一对特征需要考虑，但它们很特殊。我们必须处理Rust的神圣罗马帝国：不安全的可选内置特征 (`OIBIT`，Opt-In Built-In Traits的缩写)：[`Send`和`Sync`](https://doc.rust-lang.org/nomicon/send-and-sync.html)，它们实际上是可选和内置的。

与`Copy`特征一样，这些特征绝对没有与之关联的代码，只是标记您的类型具有特定属性。`Send`表示您的类型可以安全地发送到另一个线程。`Sync`表示您的类型可以安全地在线程之间共享（`&Self: Send`含义是`Sync`的一般都是`Send`的）。

`LinkedList`是协变的相同的逻辑也适用于此：通常不使用花哨的内部可变性技巧的普通集合可以安全地成为`Send`和`Sync`的。

但我说过它们是可选的。那么实际上，我们已经这样做了吗？我们怎么知道？

让我们为我们的代码添加一些新的魔法：随机的私有垃圾，除非我们的类型具有我们期望的属性，否则不会编译：

```rust
#[allow(dead_code)]
fn assert_properties() {
    fn is_send<T: Send>() {}
    fn is_sync<T: Sync>() {}

    is_send::<LinkedList<i32>>();
    is_sync::<LinkedList<i32>>();

    is_send::<IntoIter<i32>>();
    is_sync::<IntoIter<i32>>();

    is_send::<Iter<i32>>();
    is_sync::<Iter<i32>>();

    is_send::<IterMut<i32>>();
    is_sync::<IterMut<i32>>();

    is_send::<Cursor<i32>>();
    is_sync::<Cursor<i32>>();

    fn linked_list_covariant<'a, T>(x: LinkedList<&'static T>) -> LinkedList<&'a T> { x }
    fn iter_covariant<'i, 'a, T>(x: Iter<'i, &'static T>) -> Iter<'i, &'a T> { x }
    fn into_iter_covariant<'a, T>(x: IntoIter<&'static T>) -> IntoIter<&'a T> { x }
}
```

```txt
cargo build
   Compiling linked-list v0.0.3 
error[E0277]: `NonNull<Node<i32>>` cannot be sent between threads safely
   --> src\lib.rs:433:5
    |
433 |     is_send::<LinkedList<i32>>();
    |     ^^^^^^^^^^^^^^^^^^^^^^^^^^ `NonNull<Node<i32>>` cannot be sent between threads safely
    |
    = help: within `LinkedList<i32>`, the trait `Send` is not implemented for `NonNull<Node<i32>>`
    = note: required because it appears within the type `Option<NonNull<Node<i32>>>`
note: required because it appears within the type `LinkedList<i32>`
   --> src\lib.rs:8:12
    |
8   | pub struct LinkedList<T> {
    |            ^^^^^^^^^^
note: required by a bound in `is_send`
   --> src\lib.rs:430:19
    |
430 |     fn is_send<T: Send>() {}
    |                   ^^^^ required by this bound in `is_send`

<a million more errors>
```

哦，天哪，怎么回事！我开了个很棒的神圣罗马帝国的玩笑！

好吧，我说原始指针只有一个安全保护时骗了你：这是另一个。`*const`和`*mut`明确选择退出`Send`和`Sync`以确保安全，因此我们实际上必须选择重新加入：

```rust
unsafe impl<T: Send> Send for LinkedList<T> {}
unsafe impl<T: Sync> Sync for LinkedList<T> {}

unsafe impl<'a, T: Send> Send for Iter<'a, T> {}
unsafe impl<'a, T: Sync> Sync for Iter<'a, T> {}

unsafe impl<'a, T: Send> Send for IterMut<'a, T> {}
unsafe impl<'a, T: Sync> Sync for IterMut<'a, T> {}
```

请注意，我们必须在此处写入`unsafe impl`：这些是不安全的特征！不安全的代码（如并发库）依赖于我们仅正确实现这些特征！由于没有实际的代码，我们所做的保证只是，是的，我们在线程之间发送或共享确实是安全的！

不要只是轻描淡写地贴上这些，但我是一名认证专业人士，可以肯定地说：是的，这些完全没问题。请注意，我们不需要为`IntoIter`实现`Send`和`Sync`：它只包含`LinkedList`，因此它会自动派生`Send`和`Sync` —— 我告诉过你它们实际上是选择退出的！（您可以使用`impl !Send for MyType {}`的搞笑语法选择退出，即放弃实现`Send`特征。）

```txt
cargo build
   Compiling linked-list v0.0.3
    Finished dev [unoptimized + debuginfo] target(s) in 0.18s
```

好的，不错！

...等等，实际上，如果不应该是这些东西的东西不是的话，那将非常危险。特别是，`IterMut`绝对不应该是协变的，因为它“像”`&mut T`。但我们如何检查呢？

使用魔法！好吧，实际上，是使用`rustdoc`！好吧，我们不必为此使用`rustdoc`，但这是最有趣的方法。看，如果你编写一个文档注释并包含一个代码块，那么`rustdoc`将尝试编译并运行它，因此我们可以使用它来制作不影响主程序的新匿名“程序”：

```rust
    /// ```
    /// use linked_list::IterMut;
    /// 
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
```

```txt
cargo test

...

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 458) ... FAILED

failures:

---- src\lib.rs - assert_properties::iter_mut_invariant (line 458) stdout ----
error[E0308]: mismatched types
 --> src\lib.rs:461:86
  |
6 | fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
  |                                                                                      ^ lifetime mismatch
  |
  = note: expected struct `linked_list::IterMut<'_, &'a T>`
             found struct `linked_list::IterMut<'_, &'static T>`
```

好的，很酷，我们已经证明了它是不变的，但是呃，现在我们的测试失败了。不用担心，`rustdoc`让你说这是预料之中的，方法是用`compile_fail`注释包裹住它！

（实际上我们只证明了它“不是协变的”，但老实说，如果你设法让一个类型“意外地和错误地逆变”，那么恭喜你？）

```rust
    /// ```compile_fail
    /// use linked_list::IterMut;
    /// 
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
```

```txt
cargo test
   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 0.49s
     Running unittests src\lib.rs

...

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 458) - compile fail ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.12s
```

太棒了！我建议始终在不使用`compile_fail`的情况下进行测试，这样您就可以确认它因正确的原因而无法编译。例如，如果您忘记了使用，该测试也会失败（因此通过），这不是我们想要的！虽然能够“要求”编译器产生特定错误在概念上很有吸引力，但这绝对是一场噩梦，实际上会使编译器产生更好的错误成为一项重大改变。我们希望编译器变得更好，所以，不，你不会拥有它。

（哦，等等，我们实际上可以在`compile_fail`旁边指定我们想要的错误代码，**但这只在夜间版有效，并且由于上述原因，依赖它不是一个好主意。它将在非夜间版本中被默默忽略。**）

```rust
    /// ```compile_fail,E0308
    /// use linked_list::IterMut;
    /// 
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
```

...另外，你有没有注意到我们实际上使`IterMut`不变的部分？很容易错过，因为我“只是”复制粘贴了`Iter`并将其转储到最后。这是这里的最后一行：

```rust
pub struct IterMut<'a, T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<&'a mut T>,
}
```

让我们尝试删除这里的`PhantomData`：

```txt
 cargo build
   Compiling linked-list v0.0.3 (C:\Users\ninte\dev\contain\linked-list)
error[E0392]: parameter `'a` is never used
  --> src\lib.rs:30:20
   |
30 | pub struct IterMut<'a, T> {
   |                    ^^ unused parameter
   |
   = help: consider removing `'a`, referring to it in a field, or using a marker such as `PhantomData`
```

哈！编译器支持我们，不会让我们不使用生命周期。让我们尝试使用错误的示例：

```rust
    _boo: PhantomData<&'a T>,
```

```txt
cargo build
   Compiling linked-list v0.0.3 (C:\Users\ninte\dev\contain\linked-list)
    Finished dev [unoptimized + debuginfo] target(s) in 0.17s
```

构建成功了！我们的测试现在发现问题了吗？

```txt
cargo test

...

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 458) - compile fail ... FAILED

failures:

---- src\lib.rs - assert_properties::iter_mut_invariant (line 458) stdout ----
Test compiled successfully, but it's marked `compile_fail`.

failures:
    src\lib.rs - assert_properties::iter_mut_invariant (line 458)

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.15s
```

哎呀！！！果然测试失败了！这个系统有效！我喜欢进行真正发挥作用的测试，这样我就不必那么害怕即将出现的错误了！

## 7.9 游标介绍

好！！！我们现在有一个与Rust 1.0版本的标准库实现相当的`LinkedList`！当然，这意味着我们的`LinkedList`仍然完全没用。我们将`Deque`实现为链表，这会带来巨大的性能损失，**而且我们没有任何API可以让它真正有用**。

以下是我们对链表“杀手级应用”的处理方式：

- 🚫 做一些[奇怪的侵入性的事情](https://docs.rs/linked-hash-map/latest/linked_hash_map/)
- 🚫 做一些[奇怪的无锁的事情](https://doc.rust-lang.org/std/sync/mpsc/)
- 🚫 可以存储[动态大小的类型](https://doc.rust-lang.org/nomicon/exotic-sizes.html#dynamically-sized-types-dsts)
- 🌟 `O(1)`性能的推送/弹出，无需[额外开销销](https://en.wikipedia.org/wiki/Amortized_analysis)（如果你愿意相信`malloc`是`O(1)`）
- 🚫 `O(1)`性能的列表拆分
- 🚫 `O(1)`性能的列表拼接

好吧...6个中实现了1个... 比没有好！你知道我为什么要把这个东西从标准库中撕下来吗？

我们不会让我们的列表支持“奇怪”的东西，因为那都是临时的和特定于领域的。但是分割和拼接，现在我们可以做到了！

但问题是：实际上到达`LinkedList`中的第`k`个元素需要`O(k)`时间，那么我们怎么可能在`O(1)`内进行任意分割和合并呢？好吧，诀窍在于您没有像`split_at(index)`这样的API——您创建了一个系统，用户可以有状态地迭代到列表中的某个位置并在该位置进行`O(1)`修改！

嘿，我们已经有了迭代器！我们可以将它们用于此吗？有点……但是它们的超能力之一会妨碍我们。您可能还记得，我们为按引用迭代器写出生命周期的方式意味着它们返回的引用与迭代器无关。这让我们可以反复调用`next`并保留元素：

```rust
    let mut list = ...;
    let iter = list.iter_mut();
    let elem1 = list.next();
    let elem2 = list.next();

    if elem1 == elem2 { ... }
```

如果返回的引用借用了迭代器，那么这段代码根本行不通。编译器只会抱怨对`next`的第二次调用！这种灵活性很棒，但它给我们带来了一些隐含的限制：

- `By-Mutable-Ref`迭代器永远不能返回并再次产生一个元素，因为用户能够获得对同一元素的两个`&mut`，从而违反Rust语言的基本规则。
- `By-Ref`迭代器不能有额外的方法，这些方法可能会以一种使已经产生的任何引用无效的方式修改底层集合。

不幸的是，这两件事正是我们希望`LinkedList`的API能够做到的！所以我们不能只使用迭代器，我们需要一些新东西：游标（`cursor`）。

游标就像您在计算机上编辑文本时看到的小闪烁`|`。它是序列（文本）中的一个位置，您可以移动它（使用箭头键），并且每当您键入时，编辑操作都会在该位置发生。

See if I just

press

enter

the whole

text

gets broken in half.

很抱歉你站在我身后看着我打字，对吧？所以这完全说得通，对吧？对。

现在，如果你曾经不幸拥有一个带有“插入”键的键盘并真的按下了它，你就会知道游标实际上在技术上有两种解释：它们可以位于元素（字符）之间或元素上。我很确定没有人在他们的生活中故意按下“插入”，它纯粹是一个受苦按钮，所以很明显哪一个更好和正确：当然是游标在元素之间移动！

那里的逻辑非常牢固，我想没有人会不同意我的观点。

抱歉什么？有一份[2018年RFC要求将游标添加到Rust的`LinkedList`中](https://github.com/rust-lang/rfcs/blob/master/text/2570-linked-list-cursors.md)？

（原文）
> With a Cursor one can seek back and forth through a list and get the current element. With a CursorMut One can seek back and forth and get mutable references to elements, and it can insert and delete elements before and behind the current element (along with performing several list operations such as splitting and splicing).

（翻译）
> 使用`Cursor`可以来回搜索列表并获取当前元素。使用`CursorMut`可以来回搜索并获取元素的可变引用，并且可以在当前元素之前和之后插入和删除元素（以及执行拆分和拼接等多种列表操作）。

当前元素？此游标位于元素上，而不是元素之间！我不敢相信他们不接受我坚如磐石的论点！所以，是的，您可以在标准库中使用`Cursor`...等等，现在是[2022年了，Rust 1.60仍然将`Cursor`标记为不稳定](https://doc.rust-lang.org/1.60.0/std/collections/linked_list/struct.CursorMut.html)？

嘿，等一下：

（原文）
> Cursors always rest between two elements in the list, and index in a logically circular way. To accommodate this, there is a "ghost" non-element that yields None between the head and tail of the list.

（翻译）
> 游标始终位于列表中的两个元素之间，并以逻辑循环方式进行索引。为了适应这种情况，在列表的头部和尾部之间有一个“幽灵”非元素，其结果为 `None`。

（注：因为我们把链表看作是逻辑上循环的，所以是在头部之前，尾部之后的那个位置，就是“幽灵”非元素的位置。）

嘿，等一下。这与RFC所说的正好相反？？？但是，等一下，所有关于方法的文档仍然引用“当前”元素……等等，我以前在哪里见过这种鬼东西。哦，等等，我不是在我制作原型的[旧链表分叉](https://docs.rs/linked-list/0.0.3/linked_list/struct.Cursor.html)中这样做了吗？

（原文）
> Cursors always rest between two elements in the list, and index in a logically circular way. To accomadate this, there is a "ghost" non-element that yields None between the head and tail of the List.

（翻译）
> 游标始终位于列表中的两个元素之间，并以逻辑循环的方式进行索引。为了适应这一点，在列表的头部和尾部之间有一个“幽灵”非元素，其结果为 `None`。

等等，他妈的。这不是开玩笑，我现在正在试着阅读文档。标准库真的在RFC中提出了与我在2015年提出的设计不同的设计，然后从我的原型中复制粘贴文档吗？标准库是不是因为我写了一本关于我有多讨厌`LinkedList`的书而对我进行元垃圾评论？？？就像是的，我构建了那个原型来演示这个概念，这样人们就会让我把它添加到标准库中，让`LinkedList`不再无用，但是，这他妈的是什么？？？？？？？？？？？？？

好吧，你知道吗，显然标准库认可我的设计是客观上更优越的设计，所以我们要采用我的设计。这也很好，因为整个章节都是我实际上从头开始重写该库，所以不改变API对我来说听起来不错！

这是我写的完整顶级文档：

（原文）
> A Cursor is like an iterator, except that it can freely seek back-and-forth, and can safely mutate the list during iteration. This is because the lifetime of its yielded references are tied to its own lifetime, instead of just the underlying list. This means cursors cannot yield multiple elements at once.
>
> Cursors always rest between two elements in the list, and index in a logically circular way. To accomadate this, there is a "ghost" non-element that yields None between the head and tail of the List.
>
> When created, cursors start between the ghost and the front of the list. That is, next will yield the front of the list, and prev will yield None. Calling prev again will yield the tail.

（翻译）
> 游标类似于迭代器，不同之处在于它可以自由地来回寻觅，并且可以在迭代过程中安全地改变列表。这是因为其产生的引用的生命周期与其自身的生命周期相关联，而不仅仅是底层列表。这意味着游标不能一次产生多个元素。
>
> 游标始终位于列表中的两个元素之间，并以逻辑循环的方式进行索引。为了适应这一点，在列表的头部和尾部之间有一个“幽灵”非元素，它会产生`None`。
>
> 创建后，游标从幽灵和列表的前端之间开始。也就是说，从此位置调用`next`方法将产生列表的最前端（`front`），而调用`prev`方法将产生`None`。再次调用`prev`方法将产生尾部。

很可爱，尽管我们得出结论，引入整套“哨兵节点”的概念可能麻烦比带来的价值多，但我们最终还是会得到“假装”有一个哨兵节点的语义，这样游标就可以绕到列表的另一边。

再浏览一下我的旧API

```rust
    fn splice(&mut self, other: &mut LinkedList<T>)
```

（原文）
> Inserts the entire list's contents right after the cursor.

（翻译）
> 在游标后插入整个列表的内容。

哦，是的，我又想起这个了。当我对组合爆炸非常着迷时，我写了这篇文章，并试图想出一种方法，让每个操作只有一个副本。不幸的是，这在语义上是有问题的。看，当用户想要将一个列表拼接到另一个列表时，他们可能希望游标位于拼接之前或之后。插入的列表可以任意大，因此我们只允许一个列表并期望用户遍历整个插入的列表是一个真正的问题！

毕竟，我们必须从头开始重新设计这个设计。我们的游标类型需要什么？嗯，它需要：

- 指向两个元素“之间”
- 作为一项不错的小功能，可以跟踪下一个“索引”是什么
- 更新列表本身以修改它的前方指针/后方指针/长度。

如何指向两个元素之间？其实不需要。你只需指向“下一个”元素。所以，虽然我们公开了“游标位于两者之间”的语义，但我们实际上将其实现为“游标处于打开状态”，并假装一切都发生在该点之前或之后。

但这是有原因的！`splice`用例希望让用户选择他们是在列表之前还是之后结束，但这...用标准库的API来表达非常复杂！它们有`splice_after`和`splice_before`，但都不会改变游标的位置，所以你就真的还需要`splice_after_before`和`splice_after_after`...

等等，我有点犯傻。在标准库的API中，你可以选择你想要结束的节点，然后根据需要使用`splice_after`/`splice_before`。

斜视

等一下，标准库的API真的好吗？

浏览代码

好的，标准库的API确实不错。

好吧，算了，我们要[实现这个RFC](https://github.com/rust-lang/rfcs/blob/master/text/2570-linked-list-cursors.md)。或者至少是其中有趣的部分。

我对标准库使用的某些术语有异议，因为提及游标总是有点令人费解：`iter().next_back()`会让您`back()`，这很好，但随后的每个`next_back()`实际上都会让您更接近链表的“前部”，事实上，我们跟随的每个指针都是“前方”指针！如果我过多考虑这个看似矛盾的问题，我的大脑就会受到伤害，因此，我当然可以尊重使用不同的术语来避免这种情况。

标准库的API讨论“之前”（朝向前方）和“之后”（朝向后方）之前的操作，而不是`next`和`next_back`，它...调用`move_next`和`move_prev`。唔。好吧，他们正在使用一些迭代器术语，但至少`next`不会区分前/后，并帮助您确定事物与迭代器相比的行为方式。

我们可以解决这个问题。

## 7.10 实现游标

好的，所以我们只考虑标准库的的`CursorMut`，因为不可变版本实际上没啥意思（用游标就是要分拆和合并，这都会改变链表）。就像我最初的设计一样，它有一个包含`None`的“幽灵”元素来指示`List`的开始/结束，你可以“走过它”来绕到列表的另一边。为了实现它，我们需要：

- 一个指向当前节点的指针
- 一个指向列表的指针
- 当前索引（序号）`index`

等一下，当我们指向“幽灵”时，索引是什么？

皱起眉头...检查标准库...不喜欢标准库的答案

好吧，所以游标上的索引（`index`）更合理地是返回一个`Option<usize>`（这样在指向幽灵时，所以就可以是`None`）。而标准库实现做了一堆垃圾以避免将其存储为`Option`，但是...我们是一个链表，没问题。此外，标准库有`cursor_front`/`cursor_back`之类的东西，它们将游标从前/后元素上启动，这感觉很直观，但当列表为空时必须做一些奇怪的事情。

如果您愿意，您可以实现这些东西，但我将减少所有重复的垃圾和极端情况，只制作一个从幽灵开始的`cursor_mut`方法，人们可以使用`move_next`/`move_prev`来获取他们想要的那个（然后您可以将其包装为象标准库那样的`cursor_front`，如果您真的想要的话）。

让我们开始吧：

```rust
pub struct CursorMut<'a, T> {
    cur: Link<T>,
    list: &'a mut LinkedList<T>,
    index: Option<usize>,
}
```

非常简单，前面列表的每个条目都有一个字段来满足！现在实现`cursor_mut`方法：

```rust
impl<T> LinkedList<T> {
    pub fn cursor_mut(&mut self) -> CursorMut<T> {
        CursorMut { 
            list: self, 
            cur: None, 
            index: None,
        }
    }
}
```

因为我们是从幽灵开始的，所以我们可以直接将一切都设为`None`，简单又好用！接下来，实现移动方法：

```rust
impl<'a, T> CursorMut<'a, T> {
    pub fn index(&self) -> Option<usize> {
        self.index
    }

    pub fn move_next(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // We're on a real element, go to its next (back)
                self.cur = (*cur.as_ptr()).back;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() += 1;
                } else {
                    // We just walked to the ghost, no more index
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // We're at the ghost, and there is a real front, so move to it!
            self.cur = self.list.front;
            self.index = Some(0)
        } else {
            // We're at the ghost, but that's the only element... do nothing.
        }
    }
}
```

因此，有4种有趣的情况：

- 正常情况
- 正常情况，但我们到达了幽灵
- 幽灵情况，我们转到列表的前面
- 幽灵情况，但列表为空，因此不执行任何操作

`move_prev`的逻辑完全相同，但是前后颠倒并且索引变化也颠倒：

```rust
    pub fn move_prev(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // We're on a real element, go to its previous (front)
                self.cur = (*cur.as_ptr()).front;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() -= 1;
                } else {
                    // We just walked to the ghost, no more index
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // We're at the ghost, and there is a real back, so move to it!
            self.cur = self.list.back;
            self.index = Some(self.list.len - 1)
        } else {
            // We're at the ghost, but that's the only element... do nothing.
        }
    }
```

接下来，让我们添加一些方法来查看游标周围的元素：`current`、`peek_next`和`peek_prev`。**一个非常重要的注意事项**：这些方法必须通过`&mut self`借用我们的游标，并且结果必须与该借用相绑定。我们不能让用户获得一个可变引用的多个副本，也不能让他们在持有可变引用的同时使用我们的任何插入/删除/拆分/拼接API！

值得庆幸的是，这是Rust在使用生命周期省略时做出的默认假设，因此，我们只会默认做正确的事情！

```rust
    pub fn current(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur.map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn peek_next(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur
                .and_then(|node| (*node.as_ptr()).back)
                .map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn peek_prev(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur
                .and_then(|node| (*node.as_ptr()).front)
                .map(|node| &mut (*node.as_ptr()).elem)
        }
    }
```

头脑一片空白，`Option`方法和（省略的）编译器错误现在全都开始思考了。我对`Option<NonNull>`持怀疑态度，但是，该死的，它真的让我可以自动运行这段代码。我花了太多时间编写基于数组的集合，而你永远都不会使用`Option`，哇，这太棒了！（`(*node.as_ptr())`仍然很糟糕，但这只是Rust的原始指针……）

接下来我们有一个选择：我们可以直接跳到拆分（`split`）和拼接（`splice`），这些API的全部要点，或者我们可以迈出一小步，插入（`insert`）/删除（`remove`）单个元素。我觉得我们只是想在拆分和拼接方面实现插入/删除，所以……让我们先做这些，看看牌落在哪里（在我输入这些的时候真的不知道）。

### 拆分

首先，让我们来看`split_before`和`split_after`，它们将当前元素之前/之后的所有内容作为`LinkedList`返回（在幽灵元素处停止，除非您位于幽灵处，在这种情况下我们只返回整个列表，并且游标现在指向一个空列表）：

斜视好吧，这实际上是一些不平凡的逻辑，所以我们必须一步一步地讨论它。

我发现`split_before`有4个可能有趣的情况：

- 正常情况
- 正常情况，但`prev`是幽灵
- 幽灵情况，我们返回整个列表，并把列表变为空
- 幽灵情况，但列表为空，因此不执行任何操作并返回空列表

让我们先从极端情况开始。我认为第三种情况就是

```rust
    mem::replace(self.list, LinkedList::new())
```

对吧？我们变成空的，我们返回整个列表，并且我们的字段已经为`None`，因此无需更新。很好。哦，嘿，这对第四种情况也做了正确的事情！

现在来看看正常情况……好吧，我需要一些ASCII图表。在最普遍的情况下，我们有这样的情况：

```txt
list.front -> A <-> B <-> C <-> D <- list.back
                          ^
                         cur
```

我们想要产生这个：

```txt
list.front -> C <-> D <- list.back
              ^
             cur

return.front -> A <-> B <- return.back
```

所以我们需要打破`cur`和`prev`之间的联系，而且……天哪，需要改变的东西太多了。好吧，我只需要把它分解成几个步骤，这样我就能说服自己这是有意义的。这会有点冗长，但我至少能理解它：

```rust
    pub fn split_before(&mut self) -> LinkedList<T> {
        if let Some(cur) = self.cur {
            // We are pointing at a real element, so the list is non-empty.
            unsafe {
                // Current state
                let old_len = self.list.len;
                let old_idx = self.index.unwrap();
                let prev = (*cur.as_ptr()).front;

                // What self will become
                let new_len = old_len - old_idx;
                let new_front = self.cur;
                let new_back = self.list.back;
                let new_idx = Some(0);

                // What the output will become
                let output_len = old_len - new_len;
                let output_front = self.list.front;
                let output_back = prev;

                // Break the links between cur and prev
                if let Some(prev) = prev {
                    (*cur.as_ptr()).front = None;
                    (*prev.as_ptr()).back = None;
                }

                // Produce the result:
                self.list.len = new_len;
                self.list.front = new_front;
                self.list.back = new_back;
                self.index = new_idx;

                LinkedList {
                    front: output_front,
                    back: output_back,
                    len: output_len,
                    _boo: PhantomData,
                }
            }
        } else {
            // We're at the ghost, just replace our list with an empty one.
            // No other state needs to be changed.
            std::mem::replace(self.list, LinkedList::new())
        }
    }
```

注意下面这个`if-let`处理的是“正常情况，但`prev`是幽灵”的情况：

```rust
        if let Some(prev) = prev {
            (*cur.as_ptr()).front = None;
            (*prev.as_ptr()).back = None;
        }
```

如果你愿意的话，你可以把所有这些压缩在一起并应用以下优化：

- 将对`(*cur.as_ptr()).front`的两个访问折叠为`(*cur.as_ptr()).front.take()`
- 注意`new_back`是空操作，只需删除两者即可

据我所知，其他一切都只是偶然做了正确的事情。我们在编写测试时会看到！（复制粘贴以制作`split_after`）

我已经不再犯错误了，我只想尝试编写最万无一失的代码。实际上，这就是我编写集合的方式：将事情分解为简单的步骤和案例，直到它适合我的头脑并且看起来万无一失。然后编写大量测试，直到我确信我没有搞砸它。

因为我所做的大多数集合工作都极其不安全，所以我通常不会依赖编译器来捕获错误，而且`miri`当时还不存在！所以我只需要眯着眼睛看问题，直到我的头疼，然后尽我最大的努力永远不要犯错。

不要写不安全的Rust代码！安全的Rust就好多了！！！！

### 拼接

只剩下一个大Boss需要对付，`splice_before`和`splice_after`，我预计它们是最容易出问题的。这两个函数接受一个`LinkedList`并将其内容嫁接进来。我们的列表可能是空的，传入的参数列表也可能是空的，我们还有幽灵要处理……唉，让我们一步一步地处理`splice_before`吧。

- 如果参数列表为空，我们不需要做任何事情。
- 如果我们的列表为空，那么我们的列表就变成了参数列表。
- 如果我们指向幽灵，那么参数列表会附加到后面（更改`list.back`）
- 如果我们指向第一个元素（0），参数列表会附加到前面（更改`list.front`）
- 在一般情况下，我们会做很多指针操作。

一般情况是这样的：

```txt
input.front -> 1 <-> 2 <- input.back

 list.front -> A <-> B <-> C <- list.back
                     ^
                    cur
```

`splice_before`拼接后变成这样：

```txt
list.front -> A <-> 1 <-> 2 <-> B <-> C <- list.back
```

好吗？好。我们把它写出来吧……深吸一口气，然后跳了进去：

```rust
    pub fn splice_before(&mut self, mut input: LinkedList<T>) {
        unsafe {
            if input.is_empty() {
                // Input is empty, do nothing.
            } else if let Some(cur) = self.cur {
                if let Some(0) = self.index {
                    // We're appending to the front, see append to back
                    (*cur.as_ptr()).front = input.back.take();
                    (*input.back.unwrap().as_ptr()).back = Some(cur);
                    self.list.front = input.front.take();

                    // Index moves forward by input length
                    *self.index.as_mut().unwrap() += input.len;
                    self.list.len += input.len;
                    input.len = 0;
                } else {
                    // General Case, no boundaries, just internal fixups
                    let prev = (*cur.as_ptr()).front.unwrap();
                    let in_front = input.front.take().unwrap();
                    let in_back = input.back.take().unwrap();

                    (*prev.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(prev);
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);

                    // Index moves forward by input length
                    *self.index.as_mut().unwrap() += input.len;
                    self.list.len += input.len;
                    input.len = 0;
                }
            } else if let Some(back) = self.list.back {
                // We're on the ghost but non-empty, append to the back
                // We can either `take` the input's pointers or `mem::forget`
                // it. Using take is more responsible in case we do custom
                // allocators or something that also needs to be cleaned up!
                (*back.as_ptr()).back = input.front.take();
                (*input.front.unwrap().as_ptr()).front = Some(back);
                self.list.back = input.back.take();
                self.list.len += input.len;
                // Not necessary but Polite To Do
                input.len = 0;
            } else {
                // We're empty, become the input, remain on the ghost
                *self.list = input;
            }
        }
    }
```

好吧，这个确实很糟糕，现在真的感觉到了`Option<NonNull>`的痛苦。但我们可以做很多清理工作。首先，我们可以把这段代码拉到最后，因为我们总是想这样做。我不喜欢（虽然有时它是一个无操作，并且设置`input.len`更像是对代码未来扩展的偏执）：

```rust
        self.list.len += input.len;
        input.len = 0;
```

（错误信息）
> 使用了移动的值：input

啊，对了，在“我们为空”的情况下，我们要移动列表。让我们用交换来代替它：

```rust
        // We're empty, become the input, remain on the ghost
        std::mem::swap(self.list, &mut input);
```

在这种情况下，写入将毫无意义，但是它们仍然有效（我们可能还可以在此分支中提前返回以安抚编译器）。

这种展开只是我反向思考案例的结果，可以通过让`if-let`提出正确的问题来解决：

```rust
        if let Some(0) = self.index {
        
        } else {
            let prev = (*cur.as_ptr()).front.unwrap();
        }
```

调整索引在分支内部重复，因此也可以调提到外面：

```rust
        *self.index.as_mut().unwrap() += input.len;
```

好的，把所有这些放在一起我们得到了这个：

```rust
        if input.is_empty() {
            // Input is empty, do nothing.
        } else if let Some(cur) = self.cur {
            // Both lists are non-empty
            if let Some(prev) = (*cur.as_ptr()).front {
                // General Case, no boundaries, just internal fixups
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                (*prev.as_ptr()).back = Some(in_front);
                (*in_front.as_ptr()).front = Some(prev);
                (*cur.as_ptr()).front = Some(in_back);
                (*in_back.as_ptr()).back = Some(cur);
            } else {
                // We're appending to the front, see append to back below
                (*cur.as_ptr()).front = input.back.take();
                (*input.back.unwrap().as_ptr()).back = Some(cur);
                self.list.front = input.front.take();
            }
            // Index moves forward by input length
            *self.index.as_mut().unwrap() += input.len;
        } else if let Some(back) = self.list.back {
            // We're on the ghost but non-empty, append to the back
            // We can either `take` the input's pointers or `mem::forget`
            // it. Using take is more responsible in case we do custom
            // allocators or something that also needs to be cleaned up!
            (*back.as_ptr()).back = input.front.take();
            (*input.front.unwrap().as_ptr()).front = Some(back);
            self.list.back = input.back.take();

        } else {
            // We're empty, become the input, remain on the ghost
            std::mem::swap(self.list, &mut input);
        }

        self.list.len += input.len;
        // Not necessary but Polite To Do
        input.len = 0;

        // Input dropped here
```

好吧，这仍然很糟糕，但主要是因为——不，刚刚发现了一个bug：

```rust
            (*back.as_ptr()).back = input.front.take();
            (*input.front.unwrap().as_ptr()).front = Some(back);
```

我们获取`input.front`，然后在下一行将其解包！叹息，我们在等效镜像情况下做同样的事情。我们本可以在测试中立即发现这一点，但是，我们现在正试图做到完美，我只是在现场做这件事，而这正是我看到它的确切时刻。这就是我不再像往常一样乏味，而是分阶段做事的结果。更明确！

```rust
        // We can either `take` the input's pointers or `mem::forget`
        // it. Using `take` is more responsible in case we ever do custom
        // allocators or something that also needs to be cleaned up!
        if input.is_empty() {
            // Input is empty, do nothing.
        } else if let Some(cur) = self.cur {
            // Both lists are non-empty
            let in_front = input.front.take().unwrap();
            let in_back = input.back.take().unwrap();

            if let Some(prev) = (*cur.as_ptr()).front {
                // General Case, no boundaries, just internal fixups
                (*prev.as_ptr()).back = Some(in_front);
                (*in_front.as_ptr()).front = Some(prev);
                (*cur.as_ptr()).front = Some(in_back);
                (*in_back.as_ptr()).back = Some(cur);
            } else {
                // No prev, we're appending to the front
                (*cur.as_ptr()).front = Some(in_back);
                (*in_back.as_ptr()).back = Some(cur);
                self.list.front = Some(in_front);
            }
            // Index moves forward by input length
            *self.index.as_mut().unwrap() += input.len;
        } else if let Some(back) = self.list.back {
            // We're on the ghost but non-empty, append to the back
            let in_front = input.front.take().unwrap();
            let in_back = input.back.take().unwrap();

            (*back.as_ptr()).back = Some(in_front);
            (*in_front.as_ptr()).front = Some(back);
            self.list.back = Some(in_back);
        } else {
            // We're empty, become the input, remain on the ghost
            std::mem::swap(self.list, &mut input);
        }

        self.list.len += input.len;
        // Not necessary but Polite To Do
        input.len = 0;

        // Input dropped here
```

好了，现在我可以忍受了。我唯一的抱怨是我们没有对`in_front`/`in_back`进行重复数据删除（也许我们可以重新调整我们的条件，但无论如何）。实际上，这基本上就是你用C编写的，但`Option<NonNull>`垃圾让它变得乏味。我可以忍受。好吧，我们应该让原始指针更好地适应这些东西。但是，超出了本书的范围。

无论如何，在那之后我已经筋疲力尽了，所以，插入和删除以及所有其他API可以留给读者作为练习。

这是我们的游标的最终代码，我尝试复制粘贴组合。我做对了吗？我只有在写下一章并测试这个怪物时才会发现！

```rust
pub struct CursorMut<'a, T> {
    list: &'a mut LinkedList<T>,
    cur: Link<T>,
    index: Option<usize>,
}

impl<T> LinkedList<T> {
    pub fn cursor_mut(&mut self) -> CursorMut<T> {
        CursorMut { 
            list: self, 
            cur: None, 
            index: None,
        }
    }
}

impl<'a, T> CursorMut<'a, T> {
    pub fn index(&self) -> Option<usize> {
        self.index
    }

    pub fn move_next(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // We're on a real element, go to its next (back)
                self.cur = (*cur.as_ptr()).back;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() += 1;
                } else {
                    // We just walked to the ghost, no more index
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // We're at the ghost, and there is a real front, so move to it!
            self.cur = self.list.front;
            self.index = Some(0)
        } else {
            // We're at the ghost, but that's the only element... do nothing.
        }
    }

    pub fn move_prev(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // We're on a real element, go to its previous (front)
                self.cur = (*cur.as_ptr()).front;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() -= 1;
                } else {
                    // We just walked to the ghost, no more index
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // We're at the ghost, and there is a real back, so move to it!
            self.cur = self.list.back;
            self.index = Some(self.list.len - 1)
        } else {
            // We're at the ghost, but that's the only element... do nothing.
        }
    }

    pub fn current(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur.map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn peek_next(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur
                .and_then(|node| (*node.as_ptr()).back)
                .map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn peek_prev(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur
                .and_then(|node| (*node.as_ptr()).front)
                .map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn split_before(&mut self) -> LinkedList<T> {
        // We have this:
        //
        //     list.front -> A <-> B <-> C <-> D <- list.back
        //                               ^
        //                              cur
        // 
        //
        // And we want to produce this:
        // 
        //     list.front -> C <-> D <- list.back
        //                   ^
        //                  cur
        //
        // 
        //    return.front -> A <-> B <- return.back
        //
        if let Some(cur) = self.cur {
            // We are pointing at a real element, so the list is non-empty.
            unsafe {
                // Current state
                let old_len = self.list.len;
                let old_idx = self.index.unwrap();
                let prev = (*cur.as_ptr()).front;
                
                // What self will become
                let new_len = old_len - old_idx;
                let new_front = self.cur;
                let new_back = self.list.back;
                let new_idx = Some(0);

                // What the output will become
                let output_len = old_len - new_len;
                let output_front = self.list.front;
                let output_back = prev;

                // Break the links between cur and prev
                if let Some(prev) = prev {
                    (*cur.as_ptr()).front = None;
                    (*prev.as_ptr()).back = None;
                }

                // Produce the result:
                self.list.len = new_len;
                self.list.front = new_front;
                self.list.back = new_back;
                self.index = new_idx;

                LinkedList {
                    front: output_front,
                    back: output_back,
                    len: output_len,
                    _boo: PhantomData,
                }
            }
        } else {
            // We're at the ghost, just replace our list with an empty one.
            // No other state needs to be changed.
            std::mem::replace(self.list, LinkedList::new())
        }
    }

    pub fn split_after(&mut self) -> LinkedList<T> {
        // We have this:
        //
        //     list.front -> A <-> B <-> C <-> D <- list.back
        //                         ^
        //                        cur
        // 
        //
        // And we want to produce this:
        // 
        //     list.front -> A <-> B <- list.back
        //                         ^
        //                        cur
        //
        // 
        //    return.front -> C <-> D <- return.back
        //
        if let Some(cur) = self.cur {
            // We are pointing at a real element, so the list is non-empty.
            unsafe {
                // Current state
                let old_len = self.list.len;
                let old_idx = self.index.unwrap();
                let next = (*cur.as_ptr()).back;
                
                // What self will become
                let new_len = old_idx + 1;
                let new_back = self.cur;
                let new_front = self.list.front;
                let new_idx = Some(old_idx);

                // What the output will become
                let output_len = old_len - new_len;
                let output_front = next;
                let output_back = self.list.back;

                // Break the links between cur and next
                if let Some(next) = next {
                    (*cur.as_ptr()).back = None;
                    (*next.as_ptr()).front = None;
                }

                // Produce the result:
                self.list.len = new_len;
                self.list.front = new_front;
                self.list.back = new_back;
                self.index = new_idx;

                LinkedList {
                    front: output_front,
                    back: output_back,
                    len: output_len,
                    _boo: PhantomData,
                }
            }
        } else {
            // We're at the ghost, just replace our list with an empty one.
            // No other state needs to be changed.
            std::mem::replace(self.list, LinkedList::new())
        }
    }

    pub fn splice_before(&mut self, mut input: LinkedList<T>) {
        // We have this:
        //
        // input.front -> 1 <-> 2 <- input.back
        //
        // list.front -> A <-> B <-> C <- list.back
        //                     ^
        //                    cur
        //
        //
        // Becoming this:
        //
        // list.front -> A <-> 1 <-> 2 <-> B <-> C <- list.back
        //                                 ^
        //                                cur
        //
        unsafe {
            // We can either `take` the input's pointers or `mem::forget`
            // it. Using `take` is more responsible in case we ever do custom
            // allocators or something that also needs to be cleaned up!
            if input.is_empty() {
                // Input is empty, do nothing.
            } else if let Some(cur) = self.cur {
                // Both lists are non-empty
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                if let Some(prev) = (*cur.as_ptr()).front {
                    // General Case, no boundaries, just internal fixups
                    (*prev.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(prev);
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);
                } else {
                    // No prev, we're appending to the front
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);
                    self.list.front = Some(in_front);
                }
                // Index moves forward by input length
                *self.index.as_mut().unwrap() += input.len;
            } else if let Some(back) = self.list.back {
                // We're on the ghost but non-empty, append to the back
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                (*back.as_ptr()).back = Some(in_front);
                (*in_front.as_ptr()).front = Some(back);
                self.list.back = Some(in_back);
            } else {
                // We're empty, become the input, remain on the ghost
                std::mem::swap(self.list, &mut input);
            }

            self.list.len += input.len;
            // Not necessary but Polite To Do
            input.len = 0;
            
            // Input dropped here
        }        
    }

    pub fn splice_after(&mut self, mut input: LinkedList<T>) {
        // We have this:
        //
        // input.front -> 1 <-> 2 <- input.back
        //
        // list.front -> A <-> B <-> C <- list.back
        //                     ^
        //                    cur
        //
        //
        // Becoming this:
        //
        // list.front -> A <-> B <-> 1 <-> 2 <-> C <- list.back
        //                     ^
        //                    cur
        //
        unsafe {
            // We can either `take` the input's pointers or `mem::forget`
            // it. Using `take` is more responsible in case we ever do custom
            // allocators or something that also needs to be cleaned up!
            if input.is_empty() {
                // Input is empty, do nothing.
            } else if let Some(cur) = self.cur {
                // Both lists are non-empty
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                if let Some(next) = (*cur.as_ptr()).back {
                    // General Case, no boundaries, just internal fixups
                    (*next.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(next);
                    (*cur.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(cur);
                } else {
                    // No next, we're appending to the back
                    (*cur.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(cur);
                    self.list.back = Some(in_back);
                }
                // Index doesn't change
            } else if let Some(front) = self.list.front {
                // We're on the ghost but non-empty, append to the front
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                (*front.as_ptr()).front = Some(in_back);
                (*in_back.as_ptr()).back = Some(front);
                self.list.front = Some(in_front);
            } else {
                // We're empty, become the input, remain on the ghost
                std::mem::swap(self.list, &mut input);
            }

            self.list.len += input.len;
            // Not necessary but Polite To Do
            input.len = 0;
            
            // Input dropped here
        }        
    }
}
```

## 7.11 测试游标

是时候看看我在上一节中犯了多少令人尴尬的错误了！

哦天哪，我们制作的API与标准库和旧的实现都不一样。好吧，我只是想匆匆忙忙地从它们两者中拼凑出一些东西。是的，让我们从标准库中“借用”这些测试：

```rust
    #[test]
    fn test_cursor_move_peek() {
        let mut m: LinkedList<u32> = LinkedList::new();
        m.extend([1, 2, 3, 4, 5, 6]);
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        assert_eq!(cursor.current(), Some(&mut 1));
        assert_eq!(cursor.peek_next(), Some(&mut 2));
        assert_eq!(cursor.peek_prev(), None);
        assert_eq!(cursor.index(), Some(0));
        cursor.move_prev();
        assert_eq!(cursor.current(), None);
        assert_eq!(cursor.peek_next(), Some(&mut 1));
        assert_eq!(cursor.peek_prev(), Some(&mut 6));
        assert_eq!(cursor.index(), None);
        cursor.move_next();
        cursor.move_next();
        assert_eq!(cursor.current(), Some(&mut 2));
        assert_eq!(cursor.peek_next(), Some(&mut 3));
        assert_eq!(cursor.peek_prev(), Some(&mut 1));
        assert_eq!(cursor.index(), Some(1));

        let mut cursor = m.cursor_mut();
        cursor.move_prev();
        assert_eq!(cursor.current(), Some(&mut 6));
        assert_eq!(cursor.peek_next(), None);
        assert_eq!(cursor.peek_prev(), Some(&mut 5));
        assert_eq!(cursor.index(), Some(5));
        cursor.move_next();
        assert_eq!(cursor.current(), None);
        assert_eq!(cursor.peek_next(), Some(&mut 1));
        assert_eq!(cursor.peek_prev(), Some(&mut 6));
        assert_eq!(cursor.index(), None);
        cursor.move_prev();
        cursor.move_prev();
        assert_eq!(cursor.current(), Some(&mut 5));
        assert_eq!(cursor.peek_next(), Some(&mut 6));
        assert_eq!(cursor.peek_prev(), Some(&mut 4));
        assert_eq!(cursor.index(), Some(4));
    }

    #[test]
    fn test_cursor_mut_insert() {
        let mut m: LinkedList<u32> = LinkedList::new();
        m.extend([1, 2, 3, 4, 5, 6]);
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.splice_before(Some(7).into_iter().collect());
        cursor.splice_after(Some(8).into_iter().collect());
        // check_links(&m);
        assert_eq!(m.iter().cloned().collect::<Vec<_>>(), &[7, 1, 8, 2, 3, 4, 5, 6]);
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_prev();
        cursor.splice_before(Some(9).into_iter().collect());
        cursor.splice_after(Some(10).into_iter().collect());
        check_links(&m);
        assert_eq!(m.iter().cloned().collect::<Vec<_>>(), &[10, 7, 1, 8, 2, 3, 4, 5, 6, 9]);
        
        /* remove_current not impl'd
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_prev();
        assert_eq!(cursor.remove_current(), None);
        cursor.move_next();
        cursor.move_next();
        assert_eq!(cursor.remove_current(), Some(7));
        cursor.move_prev();
        cursor.move_prev();
        cursor.move_prev();
        assert_eq!(cursor.remove_current(), Some(9));
        cursor.move_next();
        assert_eq!(cursor.remove_current(), Some(10));
        check_links(&m);
        assert_eq!(m.iter().cloned().collect::<Vec<_>>(), &[1, 8, 2, 3, 4, 5, 6]);
        */

        let mut cursor = m.cursor_mut();
        cursor.move_next();
        let mut p: LinkedList<u32> = LinkedList::new();
        p.extend([100, 101, 102, 103]);
        let mut q: LinkedList<u32> = LinkedList::new();
        q.extend([200, 201, 202, 203]);
        cursor.splice_after(p);
        cursor.splice_before(q);
        check_links(&m);
        assert_eq!(
            m.iter().cloned().collect::<Vec<_>>(),
            &[200, 201, 202, 203, 1, 100, 101, 102, 103, 8, 2, 3, 4, 5, 6]
        );
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_prev();
        let tmp = cursor.split_before();
        assert_eq!(m.into_iter().collect::<Vec<_>>(), &[]);
        m = tmp;
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        let tmp = cursor.split_after();
        assert_eq!(tmp.into_iter().collect::<Vec<_>>(), &[102, 103, 8, 2, 3, 4, 5, 6]);
        check_links(&m);
        assert_eq!(m.iter().cloned().collect::<Vec<_>>(), &[200, 201, 202, 203, 1, 100, 101]);
    }

    fn check_links<T>(_list: &LinkedList<T>) {
        // would be good to do this!
    }
```

关键时刻！

```txt
cargo test

   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 1.03s
     Running unittests src\lib.rs

running 14 tests
test test::test_basic_front ... ok
test test::test_basic ... ok
test test::test_debug ... ok
test test::test_iterator_mut_double_end ... ok
test test::test_ord ... ok
test test::test_cursor_move_peek ... FAILED
test test::test_cursor_mut_insert ... FAILED
test test::test_iterator ... ok
test test::test_mut_iter ... ok
test test::test_eq ... ok
test test::test_rev_iter ... ok
test test::test_iterator_double_end ... ok
test test::test_hashmap ... ok
test test::test_ord_nan ... ok

failures:

---- test::test_cursor_move_peek stdout ----
thread 'test::test_cursor_move_peek' panicked at 'assertion failed: `(left == right)`
  left: `None`,
 right: `Some(1)`', src\lib.rs:1079:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace

---- test::test_cursor_mut_insert stdout ----
thread 'test::test_cursor_mut_insert' panicked at 'assertion failed: `(left == right)`
  left: `[200, 201, 202, 203, 10, 100, 101, 102, 103, 7, 1, 8, 2, 3, 4, 5, 6, 9]`,
 right: `[200, 201, 202, 203, 1, 100, 101, 102, 103, 8, 2, 3, 4, 5, 6]`', src\lib.rs:1153:9


failures:
    test::test_cursor_move_peek
    test::test_cursor_mut_insert

test result: FAILED. 12 passed; 2 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

我承认，我当时有些自大，希望自己能成功。这就是我们编写测试的原因（但也许我只是在移植测试方面做得不好……？）。

第一次失败是什么？

```rust
            let mut m: LinkedList<u32> = LinkedList::new();
            m.extend([1, 2, 3, 4, 5, 6]);
            let mut cursor = m.cursor_mut();

            cursor.move_next();
            assert_eq!(cursor.current(), Some(&mut 1));
            assert_eq!(cursor.peek_next(), Some(&mut 2));
            assert_eq!(cursor.peek_prev(), None);
            assert_eq!(cursor.index(), Some(0));

            cursor.move_prev();
            assert_eq!(cursor.current(), None);
            assert_eq!(cursor.peek_next(), Some(&mut 1)); // DIES HERE
```

天哪，我把一些基本功能搞砸了。等等，

> 头脑一片空白，`Option`的方法和（省略的）编译器错误现在占据了所有的思考。

如果我不够诚实，我就什么都不是。

```rust
    pub fn peek_next(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur
                .and_then(|node| (*node.as_ptr()).back)
                .map(|node| &mut (*node.as_ptr()).elem)
        }
    }
```

...是的，这完全是错误的。如果`self.cur`为`None`，我们不应该就此放弃，我们还需要检查`self.list.front`，因为我们在幽灵身上！所以我们只需要在链中添加一个`or_else`：

```rust
    pub fn peek_next(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur
                .and_then(|node| (*node.as_ptr()).back)
                .or_else(|| self.list.front)
                .map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn peek_prev(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur
                .and_then(|node| (*node.as_ptr()).front)
                .or_else(|| self.list.back)
                .map(|node| &mut (*node.as_ptr()).elem)
        }
    }
```

问题解决了吗？

```txt
---- test::test_cursor_move_peek stdout ----
thread 'test::test_cursor_move_peek' panicked at 'assertion failed: `(left == right)`
  left: `Some(6)`,
 right: `None`', src\lib.rs:1078:9
```

等一下，现在再往前看就错了。好吧，我需要停止头脑空空的`peek`，因为显然这比我愿意相信的要难得多。只是试图盲目地将这些情况串联起来是一场灾难，让我们对幽灵和非幽灵的情况有一个适当的判断：

```rust
    pub fn peek_next(&mut self) -> Option<&mut T> {
        unsafe {
            let next = if let Some(cur) = self.cur {
                // Normal case, try to follow the cur node's back pointer
                (*cur.as_ptr()).back
            } else {
                // Ghost case, try to use the list's front pointer
                self.list.front
            };

            // Yield the element if the next node exists
            next.map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn peek_prev(&mut self) -> Option<&mut T> {
        unsafe {
            let prev = if let Some(cur) = self.cur {
                // Normal case, try to follow the cur node's front pointer
                (*cur.as_ptr()).front
            } else {
                // Ghost case, try to use the list's back pointer
                self.list.back
            };

            // Yield the element if the prev node exists
            prev.map(|node| &mut (*node.as_ptr()).elem)
        }
    }
```

我对这个很有信心！

```txt
failures:

---- test::test_cursor_mut_insert stdout ----
thread 'test::test_cursor_mut_insert' panicked at 'assertion failed: `(left == right)`
  left: `[200, 201, 202, 203, 10, 100, 101, 102, 103, 7, 1, 8, 2, 3, 4, 5, 6, 9]`,
 right: `[200, 201, 202, 203, 1, 100, 101, 102, 103, 8, 2, 3, 4, 5, 6]`', src\lib.rs:1168:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    test::test_cursor_mut_insert

test result: FAILED. 13 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

是的。好吧，还有一次失败……哦。

你注意到我注释掉一些用于测试`remove_current`的代码了吗？是的，我没有注意到这个测试是有状态的。让我们创建一个新的列表，其中包含`remove_current`部分留给我们的状态：

```rust
            let mut m: LinkedList<u32> = LinkedList::new();
            m.extend([1, 8, 2, 3, 4, 5, 6]);
```

```txt
 cargo test
   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 0.70s
     Running unittests src\lib.rs

running 14 tests
test test::test_basic_front ... ok
test test::test_basic ... ok
test test::test_cursor_move_peek ... ok
test test::test_eq ... ok
test test::test_cursor_mut_insert ... ok
test test::test_iterator ... ok
test test::test_iterator_double_end ... ok
test test::test_ord_nan ... ok
test test::test_mut_iter ... ok
test test::test_hashmap ... ok
test test::test_debug ... ok
test test::test_ord ... ok
test test::test_iterator_mut_double_end ... ok
test test::test_rev_iter ... ok

test result: ok. 14 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 803) - compile fail ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.12s
```

嘿，看看那个……好吧，现在我有点偏执了。让我们正确填写`check_links`并在`miri`下测试它

```rust
    fn check_links<T: Eq + std::fmt::Debug>(list: &LinkedList<T>) {
        let from_front: Vec<_> = list.iter().collect();
        let from_back: Vec<_> = list.iter().rev().collect();
        let re_reved: Vec<_> = from_back.into_iter().rev().collect();

        assert_eq!(from_front, re_reved);
    }
```

```txt
$env:MIRIFLAGS="-Zmiri-tag-raw-pointers"
cargo +nightly-2023-02-14 miri test
   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 0.25s
     Running unittests src\lib.rs

running 14 tests
test test::test_basic ... ok
test test::test_basic_front ... ok
test test::test_cursor_move_peek ... ok
test test::test_cursor_mut_insert ... ok
test test::test_debug ... ok
test test::test_eq ... ok
test test::test_hashmap ... ok
test test::test_iterator ... ok
test test::test_iterator_double_end ... ok
test test::test_iterator_mut_double_end ... ok
test test::test_mut_iter ... ok
test test::test_ord ... ok
test test::test_ord_nan ... ok
test test::test_rev_iter ... ok

test result: ok. 14 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 803) - compile fail ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.10s
```

完成。

完成。

我们做到了。我们制作了一个该死的产品级质量的`LinkedList`，基本上具有与标准库中相同的所有功能。我们是否遗漏了一些方便的小方法？当然。我会将它们添加到最终发布的`crate`版本中吗？可能！

但是，我真的很累。

所以。我们赢了。

等等，我们要达到产品级质量。好的，最后一个最终Boss：`clippy`。

```txt
cargo clippy

cargo clippy
    Checking linked-list v0.0.3 (C:\Users\ninte\dev\contain\linked-list)
warning: redundant pattern matching, consider using `is_some()`
   --> src\lib.rs:189:19
    |
189 |         while let Some(_) = self.pop_front() { }
    |         ----------^^^^^^^------------------- help: try this: `while self.pop_front().is_some()`
    |
    = note: `#[warn(clippy::redundant_pattern_matching)]` on by default
    = note: this will change drop order of the result, as well as all temporaries
    = note: add `#[allow(clippy::redundant_pattern_matching)]` if this is important
    = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#redundant_pattern_matching

warning: method `into_iter` can be confused for the standard trait method `std::iter::IntoIterator::into_iter`
   --> src\lib.rs:210:5
    |
210 | /     pub fn into_iter(self) -> IntoIter<T> {
211 | |         IntoIter {
212 | |             list: self
213 | |         }
214 | |     }
    | |_____^
    |
    = note: `#[warn(clippy::should_implement_trait)]` on by default
    = help: consider implementing the trait `std::iter::IntoIterator` or choosing a less ambiguous method name
    = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#should_implement_trait

warning: redundant pattern matching, consider using `is_some()`
   --> src\lib.rs:228:19
    |
228 |         while let Some(_) = self.pop_front() { }
    |         ----------^^^^^^^------------------- help: try this: `while self.pop_front().is_some()`
    |
    = note: this will change drop order of the result, as well as all temporaries
    = note: add `#[allow(clippy::redundant_pattern_matching)]` if this is important
    = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#redundant_pattern_matching

warning: re-implementing `PartialEq::ne` is unnecessary
   --> src\lib.rs:275:5
    |
275 | /     fn ne(&self, other: &Self) -> bool {
276 | |         self.len() != other.len() || self.iter().ne(other)
277 | |     }
    | |_____^
    |
    = note: `#[warn(clippy::partialeq_ne_impl)]` on by default
    = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#partialeq_ne_impl

warning: `linked-list` (lib) generated 4 warnings
    Finished dev [unoptimized + debuginfo] target(s) in 0.29s
```

好吧，`clippy`，让我们开始吧。

投诉1（和3）：我们使用`while let Some(_) =`而不是`while .is_some()`。循环是空的，所以这真的无关紧要，但没关系，`clippy`，我会按照你的方式做事。

投诉2：我们有一个实际的固有`into_iter`方法。等等，检查标准库是否正确，指向clippy。`IntoIterator`在`prelude`中（基本上是一个 lang 项目），所以我们也不需要固有版本。

投诉 4：我们从标准库复制了一个奇怪的内容。耸耸肩，好吧，我会删除它。

```txt
cargo clippy
    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
```

...是的，它添加了一些换行符并删除了一些尾随空格。没什么有趣的。

我们现在终于完成了！！！！！！！！！！！！！

## 7.12 最终代码

真不敢相信我竟然让你从头开始重新实现`std::collections::LinkedList`，一路上还犯了各种繁琐的教条和错误。

我做到了，这本书写完了，我终于可以休息了。

好了，这是我们完全重写的1200行代码，非常精彩。这应该和这个[提交](https://github.com/contain-rs/linked-list/commit/5b69cc29454595172a5167a09277660342b78092)的文本一样。

我会重新完善一些文档，稍后发布`0.1.0`。

```rust
use std::cmp::Ordering;
use std::fmt::{self, Debug};
use std::hash::{Hash, Hasher};
use std::iter::FromIterator;
use std::marker::PhantomData;
use std::ptr::NonNull;

pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<T>,
}

type Link<T> = Option<NonNull<Node<T>>>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T,
}

pub struct Iter<'a, T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<&'a T>,
}

pub struct IterMut<'a, T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<&'a mut T>,
}

pub struct IntoIter<T> {
    list: LinkedList<T>,
}

pub struct CursorMut<'a, T> {
    list: &'a mut LinkedList<T>,
    cur: Link<T>,
    index: Option<usize>,
}

impl<T> LinkedList<T> {
    pub fn new() -> Self {
        Self {
            front: None,
            back: None,
            len: 0,
            _boo: PhantomData,
        }
    }

    pub fn push_front(&mut self, elem: T) {
        // SAFETY: it's a linked-list, what do you want?
        unsafe {
            let new = NonNull::new_unchecked(Box::into_raw(Box::new(Node {
                front: None,
                back: None,
                elem,
            })));
            if let Some(old) = self.front {
                // Put the new front before the old one
                (*old.as_ptr()).front = Some(new);
                (*new.as_ptr()).back = Some(old);
            } else {
                // If there's no front, then we're the empty list and need
                // to set the back too.
                self.back = Some(new);
            }
            // These things always happen!
            self.front = Some(new);
            self.len += 1;
        }
    }

    pub fn push_back(&mut self, elem: T) {
        // SAFETY: it's a linked-list, what do you want?
        unsafe {
            let new = NonNull::new_unchecked(Box::into_raw(Box::new(Node {
                back: None,
                front: None,
                elem,
            })));
            if let Some(old) = self.back {
                // Put the new back before the old one
                (*old.as_ptr()).back = Some(new);
                (*new.as_ptr()).front = Some(old);
            } else {
                // If there's no back, then we're the empty list and need
                // to set the front too.
                self.front = Some(new);
            }
            // These things always happen!
            self.back = Some(new);
            self.len += 1;
        }
    }

    pub fn pop_front(&mut self) -> Option<T> {
        unsafe {
            // Only have to do stuff if there is a front node to pop.
            self.front.map(|node| {
                // Bring the Box back to life so we can move out its value and
                // Drop it (Box continues to magically understand this for us).
                let boxed_node = Box::from_raw(node.as_ptr());
                let result = boxed_node.elem;

                // Make the next node into the new front.
                self.front = boxed_node.back;
                if let Some(new) = self.front {
                    // Cleanup its reference to the removed node
                    (*new.as_ptr()).front = None;
                } else {
                    // If the front is now null, then this list is now empty!
                    self.back = None;
                }

                self.len -= 1;
                result
                // Box gets implicitly freed here, knows there is no T.
            })
        }
    }

    pub fn pop_back(&mut self) -> Option<T> {
        unsafe {
            // Only have to do stuff if there is a back node to pop.
            self.back.map(|node| {
                // Bring the Box front to life so we can move out its value and
                // Drop it (Box continues to magically understand this for us).
                let boxed_node = Box::from_raw(node.as_ptr());
                let result = boxed_node.elem;

                // Make the next node into the new back.
                self.back = boxed_node.front;
                if let Some(new) = self.back {
                    // Cleanup its reference to the removed node
                    (*new.as_ptr()).back = None;
                } else {
                    // If the back is now null, then this list is now empty!
                    self.front = None;
                }

                self.len -= 1;
                result
                // Box gets implicitly freed here, knows there is no T.
            })
        }
    }

    pub fn front(&self) -> Option<&T> {
        unsafe { self.front.map(|node| &(*node.as_ptr()).elem) }
    }

    pub fn front_mut(&mut self) -> Option<&mut T> {
        unsafe { self.front.map(|node| &mut (*node.as_ptr()).elem) }
    }

    pub fn back(&self) -> Option<&T> {
        unsafe { self.back.map(|node| &(*node.as_ptr()).elem) }
    }

    pub fn back_mut(&mut self) -> Option<&mut T> {
        unsafe { self.back.map(|node| &mut (*node.as_ptr()).elem) }
    }

    pub fn len(&self) -> usize {
        self.len
    }

    pub fn is_empty(&self) -> bool {
        self.len == 0
    }

    pub fn clear(&mut self) {
        // Oh look it's drop again
        while self.pop_front().is_some() {}
    }

    pub fn iter(&self) -> Iter<T> {
        Iter {
            front: self.front,
            back: self.back,
            len: self.len,
            _boo: PhantomData,
        }
    }

    pub fn iter_mut(&mut self) -> IterMut<T> {
        IterMut {
            front: self.front,
            back: self.back,
            len: self.len,
            _boo: PhantomData,
        }
    }

    pub fn cursor_mut(&mut self) -> CursorMut<T> {
        CursorMut {
            list: self,
            cur: None,
            index: None,
        }
    }
}

impl<T> Drop for LinkedList<T> {
    fn drop(&mut self) {
        // Pop until we have to stop
        while self.pop_front().is_some() {}
    }
}

impl<T> Default for LinkedList<T> {
    fn default() -> Self {
        Self::new()
    }
}

impl<T: Clone> Clone for LinkedList<T> {
    fn clone(&self) -> Self {
        let mut new_list = Self::new();
        for item in self {
            new_list.push_back(item.clone());
        }
        new_list
    }
}

impl<T> Extend<T> for LinkedList<T> {
    fn extend<I: IntoIterator<Item = T>>(&mut self, iter: I) {
        for item in iter {
            self.push_back(item);
        }
    }
}

impl<T> FromIterator<T> for LinkedList<T> {
    fn from_iter<I: IntoIterator<Item = T>>(iter: I) -> Self {
        let mut list = Self::new();
        list.extend(iter);
        list
    }
}

impl<T: Debug> Debug for LinkedList<T> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_list().entries(self).finish()
    }
}

impl<T: PartialEq> PartialEq for LinkedList<T> {
    fn eq(&self, other: &Self) -> bool {
        self.len() == other.len() && self.iter().eq(other)
    }
}

impl<T: Eq> Eq for LinkedList<T> {}

impl<T: PartialOrd> PartialOrd for LinkedList<T> {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        self.iter().partial_cmp(other)
    }
}

impl<T: Ord> Ord for LinkedList<T> {
    fn cmp(&self, other: &Self) -> Ordering {
        self.iter().cmp(other)
    }
}

impl<T: Hash> Hash for LinkedList<T> {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.len().hash(state);
        for item in self {
            item.hash(state);
        }
    }
}

impl<'a, T> IntoIterator for &'a LinkedList<T> {
    type IntoIter = Iter<'a, T>;
    type Item = &'a T;

    fn into_iter(self) -> Self::IntoIter {
        self.iter()
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        // While self.front == self.back is a tempting condition to check here,
        // it won't do the right for yielding the last element! That sort of
        // thing only works for arrays because of "one-past-the-end" pointers.
        if self.len > 0 {
            // We could unwrap front, but this is safer and easier
            self.front.map(|node| unsafe {
                self.len -= 1;
                self.front = (*node.as_ptr()).back;
                &(*node.as_ptr()).elem
            })
        } else {
            None
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.len, Some(self.len))
    }
}

impl<'a, T> DoubleEndedIterator for Iter<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        if self.len > 0 {
            self.back.map(|node| unsafe {
                self.len -= 1;
                self.back = (*node.as_ptr()).front;
                &(*node.as_ptr()).elem
            })
        } else {
            None
        }
    }
}

impl<'a, T> ExactSizeIterator for Iter<'a, T> {
    fn len(&self) -> usize {
        self.len
    }
}

impl<'a, T> IntoIterator for &'a mut LinkedList<T> {
    type IntoIter = IterMut<'a, T>;
    type Item = &'a mut T;

    fn into_iter(self) -> Self::IntoIter {
        self.iter_mut()
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        // While self.front == self.back is a tempting condition to check here,
        // it won't do the right for yielding the last element! That sort of
        // thing only works for arrays because of "one-past-the-end" pointers.
        if self.len > 0 {
            // We could unwrap front, but this is safer and easier
            self.front.map(|node| unsafe {
                self.len -= 1;
                self.front = (*node.as_ptr()).back;
                &mut (*node.as_ptr()).elem
            })
        } else {
            None
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.len, Some(self.len))
    }
}

impl<'a, T> DoubleEndedIterator for IterMut<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        if self.len > 0 {
            self.back.map(|node| unsafe {
                self.len -= 1;
                self.back = (*node.as_ptr()).front;
                &mut (*node.as_ptr()).elem
            })
        } else {
            None
        }
    }
}

impl<'a, T> ExactSizeIterator for IterMut<'a, T> {
    fn len(&self) -> usize {
        self.len
    }
}

impl<T> IntoIterator for LinkedList<T> {
    type IntoIter = IntoIter<T>;
    type Item = T;

    fn into_iter(self) -> Self::IntoIter {
        IntoIter { list: self }
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;

    fn next(&mut self) -> Option<Self::Item> {
        self.list.pop_front()
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.list.len, Some(self.list.len))
    }
}

impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        self.list.pop_back()
    }
}

impl<T> ExactSizeIterator for IntoIter<T> {
    fn len(&self) -> usize {
        self.list.len
    }
}

impl<'a, T> CursorMut<'a, T> {
    pub fn index(&self) -> Option<usize> {
        self.index
    }

    pub fn move_next(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // We're on a real element, go to its next (back)
                self.cur = (*cur.as_ptr()).back;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() += 1;
                } else {
                    // We just walked to the ghost, no more index
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // We're at the ghost, and there is a real front, so move to it!
            self.cur = self.list.front;
            self.index = Some(0)
        } else {
            // We're at the ghost, but that's the only element... do nothing.
        }
    }

    pub fn move_prev(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // We're on a real element, go to its previous (front)
                self.cur = (*cur.as_ptr()).front;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() -= 1;
                } else {
                    // We just walked to the ghost, no more index
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // We're at the ghost, and there is a real back, so move to it!
            self.cur = self.list.back;
            self.index = Some(self.list.len - 1)
        } else {
            // We're at the ghost, but that's the only element... do nothing.
        }
    }

    pub fn current(&mut self) -> Option<&mut T> {
        unsafe { self.cur.map(|node| &mut (*node.as_ptr()).elem) }
    }

    pub fn peek_next(&mut self) -> Option<&mut T> {
        unsafe {
            let next = if let Some(cur) = self.cur {
                // Normal case, try to follow the cur node's back pointer
                (*cur.as_ptr()).back
            } else {
                // Ghost case, try to use the list's front pointer
                self.list.front
            };

            // Yield the element if the next node exists
            next.map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn peek_prev(&mut self) -> Option<&mut T> {
        unsafe {
            let prev = if let Some(cur) = self.cur {
                // Normal case, try to follow the cur node's front pointer
                (*cur.as_ptr()).front
            } else {
                // Ghost case, try to use the list's back pointer
                self.list.back
            };

            // Yield the element if the prev node exists
            prev.map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn split_before(&mut self) -> LinkedList<T> {
        // We have this:
        //
        //     list.front -> A <-> B <-> C <-> D <- list.back
        //                               ^
        //                              cur
        //
        //
        // And we want to produce this:
        //
        //     list.front -> C <-> D <- list.back
        //                   ^
        //                  cur
        //
        //
        //    return.front -> A <-> B <- return.back
        //
        if let Some(cur) = self.cur {
            // We are pointing at a real element, so the list is non-empty.
            unsafe {
                // Current state
                let old_len = self.list.len;
                let old_idx = self.index.unwrap();
                let prev = (*cur.as_ptr()).front;

                // What self will become
                let new_len = old_len - old_idx;
                let new_front = self.cur;
                let new_back = self.list.back;
                let new_idx = Some(0);

                // What the output will become
                let output_len = old_len - new_len;
                let output_front = self.list.front;
                let output_back = prev;

                // Break the links between cur and prev
                if let Some(prev) = prev {
                    (*cur.as_ptr()).front = None;
                    (*prev.as_ptr()).back = None;
                }

                // Produce the result:
                self.list.len = new_len;
                self.list.front = new_front;
                self.list.back = new_back;
                self.index = new_idx;

                LinkedList {
                    front: output_front,
                    back: output_back,
                    len: output_len,
                    _boo: PhantomData,
                }
            }
        } else {
            // We're at the ghost, just replace our list with an empty one.
            // No other state needs to be changed.
            std::mem::replace(self.list, LinkedList::new())
        }
    }

    pub fn split_after(&mut self) -> LinkedList<T> {
        // We have this:
        //
        //     list.front -> A <-> B <-> C <-> D <- list.back
        //                         ^
        //                        cur
        //
        //
        // And we want to produce this:
        //
        //     list.front -> A <-> B <- list.back
        //                         ^
        //                        cur
        //
        //
        //    return.front -> C <-> D <- return.back
        //
        if let Some(cur) = self.cur {
            // We are pointing at a real element, so the list is non-empty.
            unsafe {
                // Current state
                let old_len = self.list.len;
                let old_idx = self.index.unwrap();
                let next = (*cur.as_ptr()).back;

                // What self will become
                let new_len = old_idx + 1;
                let new_back = self.cur;
                let new_front = self.list.front;
                let new_idx = Some(old_idx);

                // What the output will become
                let output_len = old_len - new_len;
                let output_front = next;
                let output_back = self.list.back;

                // Break the links between cur and next
                if let Some(next) = next {
                    (*cur.as_ptr()).back = None;
                    (*next.as_ptr()).front = None;
                }

                // Produce the result:
                self.list.len = new_len;
                self.list.front = new_front;
                self.list.back = new_back;
                self.index = new_idx;

                LinkedList {
                    front: output_front,
                    back: output_back,
                    len: output_len,
                    _boo: PhantomData,
                }
            }
        } else {
            // We're at the ghost, just replace our list with an empty one.
            // No other state needs to be changed.
            std::mem::replace(self.list, LinkedList::new())
        }
    }

    pub fn splice_before(&mut self, mut input: LinkedList<T>) {
        // We have this:
        //
        // input.front -> 1 <-> 2 <- input.back
        //
        // list.front -> A <-> B <-> C <- list.back
        //                     ^
        //                    cur
        //
        //
        // Becoming this:
        //
        // list.front -> A <-> 1 <-> 2 <-> B <-> C <- list.back
        //                                 ^
        //                                cur
        //
        unsafe {
            // We can either `take` the input's pointers or `mem::forget`
            // it. Using `take` is more responsible in case we ever do custom
            // allocators or something that also needs to be cleaned up!
            if input.is_empty() {
                // Input is empty, do nothing.
            } else if let Some(cur) = self.cur {
                // Both lists are non-empty
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                if let Some(prev) = (*cur.as_ptr()).front {
                    // General Case, no boundaries, just internal fixups
                    (*prev.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(prev);
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);
                } else {
                    // No prev, we're appending to the front
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);
                    self.list.front = Some(in_front);
                }
                // Index moves forward by input length
                *self.index.as_mut().unwrap() += input.len;
            } else if let Some(back) = self.list.back {
                // We're on the ghost but non-empty, append to the back
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                (*back.as_ptr()).back = Some(in_front);
                (*in_front.as_ptr()).front = Some(back);
                self.list.back = Some(in_back);
            } else {
                // We're empty, become the input, remain on the ghost
                std::mem::swap(self.list, &mut input);
            }

            self.list.len += input.len;
            // Not necessary but Polite To Do
            input.len = 0;

            // Input dropped here
        }
    }

    pub fn splice_after(&mut self, mut input: LinkedList<T>) {
        // We have this:
        //
        // input.front -> 1 <-> 2 <- input.back
        //
        // list.front -> A <-> B <-> C <- list.back
        //                     ^
        //                    cur
        //
        //
        // Becoming this:
        //
        // list.front -> A <-> B <-> 1 <-> 2 <-> C <- list.back
        //                     ^
        //                    cur
        //
        unsafe {
            // We can either `take` the input's pointers or `mem::forget`
            // it. Using `take` is more responsible in case we ever do custom
            // allocators or something that also needs to be cleaned up!
            if input.is_empty() {
                // Input is empty, do nothing.
            } else if let Some(cur) = self.cur {
                // Both lists are non-empty
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                if let Some(next) = (*cur.as_ptr()).back {
                    // General Case, no boundaries, just internal fixups
                    (*next.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(next);
                    (*cur.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(cur);
                } else {
                    // No next, we're appending to the back
                    (*cur.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(cur);
                    self.list.back = Some(in_back);
                }
                // Index doesn't change
            } else if let Some(front) = self.list.front {
                // We're on the ghost but non-empty, append to the front
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                (*front.as_ptr()).front = Some(in_back);
                (*in_back.as_ptr()).back = Some(front);
                self.list.front = Some(in_front);
            } else {
                // We're empty, become the input, remain on the ghost
                std::mem::swap(self.list, &mut input);
            }

            self.list.len += input.len;
            // Not necessary but Polite To Do
            input.len = 0;

            // Input dropped here
        }
    }
}

unsafe impl<T: Send> Send for LinkedList<T> {}
unsafe impl<T: Sync> Sync for LinkedList<T> {}

unsafe impl<'a, T: Send> Send for Iter<'a, T> {}
unsafe impl<'a, T: Sync> Sync for Iter<'a, T> {}

unsafe impl<'a, T: Send> Send for IterMut<'a, T> {}
unsafe impl<'a, T: Sync> Sync for IterMut<'a, T> {}

#[allow(dead_code)]
fn assert_properties() {
    fn is_send<T: Send>() {}
    fn is_sync<T: Sync>() {}

    is_send::<LinkedList<i32>>();
    is_sync::<LinkedList<i32>>();

    is_send::<IntoIter<i32>>();
    is_sync::<IntoIter<i32>>();

    is_send::<Iter<i32>>();
    is_sync::<Iter<i32>>();

    is_send::<IterMut<i32>>();
    is_sync::<IterMut<i32>>();

    fn linked_list_covariant<'a, T>(x: LinkedList<&'static T>) -> LinkedList<&'a T> {
        x
    }
    fn iter_covariant<'i, 'a, T>(x: Iter<'i, &'static T>) -> Iter<'i, &'a T> {
        x
    }
    fn into_iter_covariant<'a, T>(x: IntoIter<&'static T>) -> IntoIter<&'a T> {
        x
    }

    /// ```compile_fail,E0308
    /// use linked_list::IterMut;
    ///
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
}

#[cfg(test)]
mod test {
    use super::LinkedList;

    fn generate_test() -> LinkedList<i32> {
        list_from(&[0, 1, 2, 3, 4, 5, 6])
    }

    fn list_from<T: Clone>(v: &[T]) -> LinkedList<T> {
        v.iter().map(|x| (*x).clone()).collect()
    }

    #[test]
    fn test_basic_front() {
        let mut list = LinkedList::new();

        // Try to break an empty list
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // Try to break a one item list
        list.push_front(10);
        assert_eq!(list.len(), 1);
        assert_eq!(list.pop_front(), Some(10));
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // Mess around
        list.push_front(10);
        assert_eq!(list.len(), 1);
        list.push_front(20);
        assert_eq!(list.len(), 2);
        list.push_front(30);
        assert_eq!(list.len(), 3);
        assert_eq!(list.pop_front(), Some(30));
        assert_eq!(list.len(), 2);
        list.push_front(40);
        assert_eq!(list.len(), 3);
        assert_eq!(list.pop_front(), Some(40));
        assert_eq!(list.len(), 2);
        assert_eq!(list.pop_front(), Some(20));
        assert_eq!(list.len(), 1);
        assert_eq!(list.pop_front(), Some(10));
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);
    }

    #[test]
    fn test_basic() {
        let mut m = LinkedList::new();
        assert_eq!(m.pop_front(), None);
        assert_eq!(m.pop_back(), None);
        assert_eq!(m.pop_front(), None);
        m.push_front(1);
        assert_eq!(m.pop_front(), Some(1));
        m.push_back(2);
        m.push_back(3);
        assert_eq!(m.len(), 2);
        assert_eq!(m.pop_front(), Some(2));
        assert_eq!(m.pop_front(), Some(3));
        assert_eq!(m.len(), 0);
        assert_eq!(m.pop_front(), None);
        m.push_back(1);
        m.push_back(3);
        m.push_back(5);
        m.push_back(7);
        assert_eq!(m.pop_front(), Some(1));

        let mut n = LinkedList::new();
        n.push_front(2);
        n.push_front(3);
        {
            assert_eq!(n.front().unwrap(), &3);
            let x = n.front_mut().unwrap();
            assert_eq!(*x, 3);
            *x = 0;
        }
        {
            assert_eq!(n.back().unwrap(), &2);
            let y = n.back_mut().unwrap();
            assert_eq!(*y, 2);
            *y = 1;
        }
        assert_eq!(n.pop_front(), Some(0));
        assert_eq!(n.pop_front(), Some(1));
    }

    #[test]
    fn test_iterator() {
        let m = generate_test();
        for (i, elt) in m.iter().enumerate() {
            assert_eq!(i as i32, *elt);
        }
        let mut n = LinkedList::new();
        assert_eq!(n.iter().next(), None);
        n.push_front(4);
        let mut it = n.iter();
        assert_eq!(it.size_hint(), (1, Some(1)));
        assert_eq!(it.next().unwrap(), &4);
        assert_eq!(it.size_hint(), (0, Some(0)));
        assert_eq!(it.next(), None);
    }

    #[test]
    fn test_iterator_double_end() {
        let mut n = LinkedList::new();
        assert_eq!(n.iter().next(), None);
        n.push_front(4);
        n.push_front(5);
        n.push_front(6);
        let mut it = n.iter();
        assert_eq!(it.size_hint(), (3, Some(3)));
        assert_eq!(it.next().unwrap(), &6);
        assert_eq!(it.size_hint(), (2, Some(2)));
        assert_eq!(it.next_back().unwrap(), &4);
        assert_eq!(it.size_hint(), (1, Some(1)));
        assert_eq!(it.next_back().unwrap(), &5);
        assert_eq!(it.next_back(), None);
        assert_eq!(it.next(), None);
    }

    #[test]
    fn test_rev_iter() {
        let m = generate_test();
        for (i, elt) in m.iter().rev().enumerate() {
            assert_eq!(6 - i as i32, *elt);
        }
        let mut n = LinkedList::new();
        assert_eq!(n.iter().rev().next(), None);
        n.push_front(4);
        let mut it = n.iter().rev();
        assert_eq!(it.size_hint(), (1, Some(1)));
        assert_eq!(it.next().unwrap(), &4);
        assert_eq!(it.size_hint(), (0, Some(0)));
        assert_eq!(it.next(), None);
    }

    #[test]
    fn test_mut_iter() {
        let mut m = generate_test();
        let mut len = m.len();
        for (i, elt) in m.iter_mut().enumerate() {
            assert_eq!(i as i32, *elt);
            len -= 1;
        }
        assert_eq!(len, 0);
        let mut n = LinkedList::new();
        assert!(n.iter_mut().next().is_none());
        n.push_front(4);
        n.push_back(5);
        let mut it = n.iter_mut();
        assert_eq!(it.size_hint(), (2, Some(2)));
        assert!(it.next().is_some());
        assert!(it.next().is_some());
        assert_eq!(it.size_hint(), (0, Some(0)));
        assert!(it.next().is_none());
    }

    #[test]
    fn test_iterator_mut_double_end() {
        let mut n = LinkedList::new();
        assert!(n.iter_mut().next_back().is_none());
        n.push_front(4);
        n.push_front(5);
        n.push_front(6);
        let mut it = n.iter_mut();
        assert_eq!(it.size_hint(), (3, Some(3)));
        assert_eq!(*it.next().unwrap(), 6);
        assert_eq!(it.size_hint(), (2, Some(2)));
        assert_eq!(*it.next_back().unwrap(), 4);
        assert_eq!(it.size_hint(), (1, Some(1)));
        assert_eq!(*it.next_back().unwrap(), 5);
        assert!(it.next_back().is_none());
        assert!(it.next().is_none());
    }

    #[test]
    fn test_eq() {
        let mut n: LinkedList<u8> = list_from(&[]);
        let mut m = list_from(&[]);
        assert!(n == m);
        n.push_front(1);
        assert!(n != m);
        m.push_back(1);
        assert!(n == m);

        let n = list_from(&[2, 3, 4]);
        let m = list_from(&[1, 2, 3]);
        assert!(n != m);
    }

    #[test]
    fn test_ord() {
        let n = list_from(&[]);
        let m = list_from(&[1, 2, 3]);
        assert!(n < m);
        assert!(m > n);
        assert!(n <= n);
        assert!(n >= n);
    }

    #[test]
    fn test_ord_nan() {
        let nan = 0.0f64 / 0.0;
        let n = list_from(&[nan]);
        let m = list_from(&[nan]);
        assert!(!(n < m));
        assert!(!(n > m));
        assert!(!(n <= m));
        assert!(!(n >= m));

        let n = list_from(&[nan]);
        let one = list_from(&[1.0f64]);
        assert!(!(n < one));
        assert!(!(n > one));
        assert!(!(n <= one));
        assert!(!(n >= one));

        let u = list_from(&[1.0f64, 2.0, nan]);
        let v = list_from(&[1.0f64, 2.0, 3.0]);
        assert!(!(u < v));
        assert!(!(u > v));
        assert!(!(u <= v));
        assert!(!(u >= v));

        let s = list_from(&[1.0f64, 2.0, 4.0, 2.0]);
        let t = list_from(&[1.0f64, 2.0, 3.0, 2.0]);
        assert!(!(s < t));
        assert!(s > one);
        assert!(!(s <= one));
        assert!(s >= one);
    }

    #[test]
    fn test_debug() {
        let list: LinkedList<i32> = (0..10).collect();
        assert_eq!(format!("{:?}", list), "[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]");

        let list: LinkedList<&str> = vec!["just", "one", "test", "more"]
            .iter()
            .copied()
            .collect();
        assert_eq!(format!("{:?}", list), r#"["just", "one", "test", "more"]"#);
    }

    #[test]
    fn test_hashmap() {
        // Check that HashMap works with this as a key

        let list1: LinkedList<i32> = (0..10).collect();
        let list2: LinkedList<i32> = (1..11).collect();
        let mut map = std::collections::HashMap::new();

        assert_eq!(map.insert(list1.clone(), "list1"), None);
        assert_eq!(map.insert(list2.clone(), "list2"), None);

        assert_eq!(map.len(), 2);

        assert_eq!(map.get(&list1), Some(&"list1"));
        assert_eq!(map.get(&list2), Some(&"list2"));

        assert_eq!(map.remove(&list1), Some("list1"));
        assert_eq!(map.remove(&list2), Some("list2"));

        assert!(map.is_empty());
    }

    #[test]
    fn test_cursor_move_peek() {
        let mut m: LinkedList<u32> = LinkedList::new();
        m.extend([1, 2, 3, 4, 5, 6]);
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        assert_eq!(cursor.current(), Some(&mut 1));
        assert_eq!(cursor.peek_next(), Some(&mut 2));
        assert_eq!(cursor.peek_prev(), None);
        assert_eq!(cursor.index(), Some(0));
        cursor.move_prev();
        assert_eq!(cursor.current(), None);
        assert_eq!(cursor.peek_next(), Some(&mut 1));
        assert_eq!(cursor.peek_prev(), Some(&mut 6));
        assert_eq!(cursor.index(), None);
        cursor.move_next();
        cursor.move_next();
        assert_eq!(cursor.current(), Some(&mut 2));
        assert_eq!(cursor.peek_next(), Some(&mut 3));
        assert_eq!(cursor.peek_prev(), Some(&mut 1));
        assert_eq!(cursor.index(), Some(1));

        let mut cursor = m.cursor_mut();
        cursor.move_prev();
        assert_eq!(cursor.current(), Some(&mut 6));
        assert_eq!(cursor.peek_next(), None);
        assert_eq!(cursor.peek_prev(), Some(&mut 5));
        assert_eq!(cursor.index(), Some(5));
        cursor.move_next();
        assert_eq!(cursor.current(), None);
        assert_eq!(cursor.peek_next(), Some(&mut 1));
        assert_eq!(cursor.peek_prev(), Some(&mut 6));
        assert_eq!(cursor.index(), None);
        cursor.move_prev();
        cursor.move_prev();
        assert_eq!(cursor.current(), Some(&mut 5));
        assert_eq!(cursor.peek_next(), Some(&mut 6));
        assert_eq!(cursor.peek_prev(), Some(&mut 4));
        assert_eq!(cursor.index(), Some(4));
    }

    #[test]
    fn test_cursor_mut_insert() {
        let mut m: LinkedList<u32> = LinkedList::new();
        m.extend([1, 2, 3, 4, 5, 6]);
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.splice_before(Some(7).into_iter().collect());
        cursor.splice_after(Some(8).into_iter().collect());
        // check_links(&m);
        assert_eq!(
            m.iter().cloned().collect::<Vec<_>>(),
            &[7, 1, 8, 2, 3, 4, 5, 6]
        );
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_prev();
        cursor.splice_before(Some(9).into_iter().collect());
        cursor.splice_after(Some(10).into_iter().collect());
        check_links(&m);
        assert_eq!(
            m.iter().cloned().collect::<Vec<_>>(),
            &[10, 7, 1, 8, 2, 3, 4, 5, 6, 9]
        );

        /* remove_current not impl'd
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_prev();
        assert_eq!(cursor.remove_current(), None);
        cursor.move_next();
        cursor.move_next();
        assert_eq!(cursor.remove_current(), Some(7));
        cursor.move_prev();
        cursor.move_prev();
        cursor.move_prev();
        assert_eq!(cursor.remove_current(), Some(9));
        cursor.move_next();
        assert_eq!(cursor.remove_current(), Some(10));
        check_links(&m);
        assert_eq!(m.iter().cloned().collect::<Vec<_>>(), &[1, 8, 2, 3, 4, 5, 6]);
        */

        let mut m: LinkedList<u32> = LinkedList::new();
        m.extend([1, 8, 2, 3, 4, 5, 6]);
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        let mut p: LinkedList<u32> = LinkedList::new();
        p.extend([100, 101, 102, 103]);
        let mut q: LinkedList<u32> = LinkedList::new();
        q.extend([200, 201, 202, 203]);
        cursor.splice_after(p);
        cursor.splice_before(q);
        check_links(&m);
        assert_eq!(
            m.iter().cloned().collect::<Vec<_>>(),
            &[200, 201, 202, 203, 1, 100, 101, 102, 103, 8, 2, 3, 4, 5, 6]
        );
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_prev();
        let tmp = cursor.split_before();
        assert_eq!(m.into_iter().collect::<Vec<_>>(), &[]);
        m = tmp;
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        let tmp = cursor.split_after();
        assert_eq!(
            tmp.into_iter().collect::<Vec<_>>(),
            &[102, 103, 8, 2, 3, 4, 5, 6]
        );
        check_links(&m);
        assert_eq!(
            m.iter().cloned().collect::<Vec<_>>(),
            &[200, 201, 202, 203, 1, 100, 101]
        );
    }

    fn check_links<T: Eq + std::fmt::Debug>(list: &LinkedList<T>) {
        let from_front: Vec<_> = list.iter().collect();
        let from_back: Vec<_> = list.iter().rev().collect();
        let re_reved: Vec<_> = from_back.into_iter().rev().collect();

        assert_eq!(from_front, re_reved);
    }
}
```
