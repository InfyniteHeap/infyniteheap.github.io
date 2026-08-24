---
title: "LRU和LFU笔记"
description: "讲解实现LRU和LFU的步骤，顺便讲解实现链式哈希表的步骤。"
slug: "lru-and-lfu-note"
date: "2025-08-09T07:30:16+08:00"
image: nahida.jpg
categories:
  - 计算机软件
tags:
  - 数据结构与算法
  - LRU
  - LFU
math:
toc:
---

近日，笔者在 LeetCode 上刷到了[146. LRU 缓存](https://leetcode.cn/problems/lru-cache/description/)和[460. LFU 缓存](https://leetcode.cn/problems/lfu-cache/description/)这两道题。虽然这两种数据结构工作原理不难理解，但是实现它们却并非易事。因此，笔者撰写本篇文章，旨在带领读者自下至上逐步实现这两种数据结构。

至于 LRU 和 LFU 的概念与工作原理，网上都有，这里便不再单独讲解了。

## 说明

LRU 和 LFU 的核心是链式哈希表。该数据结构有如下特点：

1. 保证内部元素的**插入顺序**。
2. 增删查改操作的时间复杂度均为 **O(1)**，且**不会破坏**原有元素插入顺序。

部分编程语言已经内置了这种数据结构，允许开发者直接使用：

- Java：[`LinkedHashMap`](https://docs.oracle.com/en/java/javase/26/docs/api/java.base/java/util/LinkedHashMap.html)。
- Python：自 3.7 版本起，Python 内置的[`dict`](https://docs.python.org/3/library/stdtypes.html#mapping-types-dict)便已符合上述特点。当然，考虑到我们有将数据移动到双向链表头/尾的需求，我们一般会使用[`collections.OrderedDict`](https://docs.python.org/3/library/collections.html#collections.OrderedDict)，因为它包含诸如`move_to_end`等这样方便的高级方法。
- JavaScript/TypeScript：[`Map`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map)。

以 Python 为例，LRU 实现如下：

```python
from collections import OrderedDict


class LRUCache:
    # 使用__slots__属性以提高访问类内属性的速度
    __slots__: list[str] = ["cache", "cap"]

    def __init__(self, capacity: int) -> None:
        # 显式标注类型可提升代码可读性，以及开发时的code lint体验
        self.cache: OrderedDict[int, int] = OrderedDict()
        self.cap: int = capacity

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1

        self.cache.move_to_end(key, last=False)
        return self.cache[key]

    def put(self, key: int, value: int) -> None:
        if self.cap == 0:
            return

        self.cache[key] = value
        self.cache.move_to_end(key, last=False)

        if len(self.cache) > self.cap:
            _ = self.cache.popitem()
```

LFU 实现如下：

```python
from collections import OrderedDict


class LFUCache:
    __slots__: list[str] = [
        "key_to_value",
        "key_to_freq",
        "freq_to_keys",
        "min_freq",
        "cap",
    ]

    def __init__(self, capacity: int) -> None:
        self.key_to_value: dict[int, int] = {}
        self.key_to_freq: dict[int, int] = {}
        self.freq_to_keys: dict[int, OrderedDict[int, None]] = {}
        self.min_freq: int = 0
        self.cap: int = capacity

    def increase_freq(self, key: int) -> None:
        freq = self.key_to_freq[key]
        self.key_to_freq[key] += 1

        del self.freq_to_keys[freq][key]
        if len(self.freq_to_keys[freq]) == 0:
            del self.freq_to_keys[freq]
            if freq == self.min_freq:
                self.min_freq += 1

        if freq + 1 not in self.freq_to_keys:
            self.freq_to_keys[freq + 1] = OrderedDict()
        self.freq_to_keys[freq + 1][key] = None

    def remove_min_freq_key(self) -> None:
        key, _ = self.freq_to_keys[self.min_freq].popitem(last=False)

        if len(self.freq_to_keys[self.min_freq]) == 0:
            del self.freq_to_keys[self.min_freq]

        del self.key_to_value[key]
        del self.key_to_freq[key]

    def get(self, key: int) -> int:
        if key not in self.key_to_value:
            return -1

        self.increase_freq(key)
        return self.key_to_value[key]

    def put(self, key: int, value: int) -> None:
        if self.cap == 0:
            return

        if key in self.key_to_value:
            self.key_to_value[key] = value
            self.increase_freq(key)

            return

        self.key_to_value[key] = value
        self.key_to_freq[key] = 1
        if 1 not in self.freq_to_keys:
            self.freq_to_keys[1] = OrderedDict()
        self.freq_to_keys[1][key] = None

        if len(self.key_to_value) > self.cap:
            self.remove_min_freq_key()

        self.min_freq = 1
```

部分编程语言或许没有内置链式哈希表，但允许开发者利用语言内置的双向链表和哈希表手动实现一个[^1]：

- C++：[`std::list`](https://en.cppreference.com/w/cpp/container/list.html)和[`std::unordered_map`](https://en.cppreference.com/w/cpp/container/unordered_map.html)。其中，哈希表的值是指向与键相关的双向链表节点的迭代器（本质上就是指针）。
- Go：[`list`](https://pkg.go.dev/container/list)和[`map`](https://pkg.go.dev/maps)。其中，哈希表的值是指向与键相关的双向链表节点的指针，其类型为`*list.Element`。
- .NET[^2]：[`LinkedList`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.linkedlist-1)和[`Dictionary`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.dictionary-2)。其中，哈希表的值是指向与键相关的双向链表节点的指针，其类型为[`LinkedListNode`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.linkedlistnode-1)。

除上述语言外，其余语言除了要手动实现链式哈希表，还要在实现它之前先手动实现一个双向链表！综合考虑，本文将给出 Rust 代码实现。除了 Rust（很不幸）不符合上述两种情况[^3]之外，还有以下这些原因：

1. 笔者最熟悉的语言是 Rust。
2. Rust 对类型的要求非常严格，且不允许隐式类型转换。
3. 由于 Rust 严格限制裸指针的使用[^4]，因此使用 Rust 编写链式数据结构要比其他语言困难。譬如，要表示指向某对象的指针，其他语言要么使用`*Type`或`Type*`，要么把类规定为引用类型（通常这种情况发生在 GC 语言中，因为这类语言往往按引用传递类对象）。而 Rust 则一般使用`Option<Rc<RefCell<Node<E>>>>`来表示。访问对象时，其他语言可以直接使用`ptr->field`（C/C++）或`ptr.field`，而 Rust 则必须用`ptr.as_ref().unwrap().borrow().field`。其中，使用`as_ref()`是为了防止`unwrap`将`ptr`的所有权转移走，以方便后续继续使用`ptr`。

接下来，我们开始使用 Rust 逐步实现 LRU 和 LFU！

## 实现双向链表

根据上述所言，我们需要先实现双向链表。我们将实现步骤拆为两步：

1. 实现双向链表节点。
2. 实现双向链表本身。

### 实现双向链表节点

首先，我们先来实现双向链表节点：

```rust
use std::cell::RefCell;
use std::rc::Rc;

#[derive(PartialEq, Eq)]
struct Node<E> {
    prev: Option<Rc<RefCell<Node<E>>>>,
    next: Option<Rc<RefCell<Node<E>>>>,
    element: E,
}

impl<E> Node<E>
where
    E: PartialEq + Eq,
{
    fn new(element: E) -> Self {
        Self {
            prev: None,
            next: None,
            element,
        }
    }
}
```

我们为节点定义三个字段：用于指向前一个节点的`prev`、用于指向后一个节点的`next`，以及用于存储数据的`element`。其中，`element`为任意类型的数据，因此我们将其类型设为泛型类型。

### 实现双向链表

有了前面实现的双向链表节点，我们就可以实现双向链表了。

首先，我们先实现其数据结构：

```rust
struct DoubleLinkedList<E> {
    head: Option<Rc<RefCell<Node<E>>>>,
    tail: Option<Rc<RefCell<Node<E>>>>,
    len: usize,
}

impl<E> DoubleLinkedList<E>
where
    E: Default + PartialEq + Eq,
{
    fn new() -> Self {
        let head = Some(Rc::new(RefCell::new(Node::new(E::default()))));
        let tail = Some(Rc::new(RefCell::new(Node::new(E::default()))));

        head.as_ref().unwrap().borrow_mut().next = tail.clone();
        tail.as_ref().unwrap().borrow_mut().prev = head.clone();

        Self { head, tail, len: 0 }
    }
}
```

我们为双向链表定义三个字段：`head`、`tail`和`len`。其中，`head`和`tail`表示虚拟空节点，用于方便处理移除头/尾节点的情况；`len`表示双向链表长度，即双向链表所包含的节点数量。

接下来，我们为双向链表添加一些方法：

```rust
fn push_front(&mut self, node: Option<Rc<RefCell<Node<E>>>>) {
    if node.is_none() {
        return;
    }

    node.as_ref().unwrap().borrow_mut().prev = self.head.clone();
    node.as_ref().unwrap().borrow_mut().next = self.head.as_ref().unwrap().borrow().next.clone();

    self.head
        .as_ref()
        .unwrap()
        .borrow()
        .next
        .as_ref()
        .unwrap()
        .borrow_mut()
        .prev = node.clone();
    self.head.as_ref().unwrap().borrow_mut().next = node.clone();

    self.len += 1;
}

fn push_back(&mut self, node: Option<Rc<RefCell<Node<E>>>>) {
    if node.is_none() {
        return;
    }

    node.as_ref().unwrap().borrow_mut().prev = self.tail.as_ref().unwrap().borrow().prev.clone();
    node.as_ref().unwrap().borrow_mut().next = self.tail.clone();

    self.tail
        .as_ref()
        .unwrap()
        .borrow()
        .prev
        .as_ref()
        .unwrap()
        .borrow_mut()
        .next = node.clone();
    self.tail.as_ref().unwrap().borrow_mut().prev = node.clone();

    self.len += 1;
}

fn pop(&mut self, node: &Option<Rc<RefCell<Node<E>>>>) {
    if node.is_none() {
        return;
    }

    node.as_ref()
        .unwrap()
        .borrow()
        .prev
        .as_ref()
        .unwrap()
        .borrow_mut()
        .next = node.as_ref().unwrap().borrow().next.clone();
    node.as_ref()
        .unwrap()
        .borrow()
        .next
        .as_ref()
        .unwrap()
        .borrow_mut()
        .prev = node.as_ref().unwrap().borrow().prev.clone();

    self.len -= 1;
}

fn pop_front(&mut self) -> Option<Rc<RefCell<Node<E>>>> {
    let front = self.head.as_ref().unwrap().borrow().next.clone();

    if front != self.tail {
        self.pop(&front);

        front
    } else {
        None
    }
}

fn pop_back(&mut self) -> Option<Rc<RefCell<Node<E>>>> {
    let back = self.tail.as_ref().unwrap().borrow().prev.clone();

    if back != self.head {
        self.pop(&back);

        back
    } else {
        None
    }
}

fn len(&self) -> usize {
    self.len
}
```

我们为双向链表实现了这几个方法：`push_front`、`push_back`、`pop`、`pop_front`、`pop_back`和`len`。虽然部分方法在实现链式哈希表时可能不会用到，但为了让这个双向链表更通用一些，笔者并未删除未用到的“冗余”方法。读者自己实现时可以酌情删除部分用不到的方法。

读到这里，或许有读者会疑惑，这几个方法含`unwrap`极高，操作双向链表时会不会有可能发生 panic？其实在这里直接使用`unwrap`是不会出现问题的，原因如下：

1. 初始化双向链表时，`head`和`tail`也会初始化为一个确切值。根据`push_*`方法中的逻辑，在修改`head.next`或`tail.prev`时，由于`head`、`head.next`、`tail`和`tail.prev`已经有值，因此`unwrap`所接收的值一定是`Option::Some(T)`。
2. 前三个传入节点指针的方法皆以针对传入的指针做空值校验，这也避免了因传入一个`None`而导致触发 panic 的情况。(其实，我们一般不会推入/弹出一个空节点指针，因此参数可以写成`Rc<RefCell<Node<E>>>`，这样连空值校验也不需要了。)

就算如此，在正式项目中，我们仍需要避免直接使用`unwrap`，而应改用其他更优雅的方式来处理此类情况。当然，本文的重点在于讲解实现思路，因此此类问题在这里就暂时忽略了。

## 实现链式哈希表

有了前面的双向链表，我们就可以着手实现链式哈希表了。

我们同样先实现其数据结构：

```rust
use std::collections::HashMap;
use std::hash::Hash;

struct LinkedHashMap<K, V> {
    dll: DoubleLinkedList<(K, V)>,
    hm: HashMap<K, Option<Rc<RefCell<Node<(K, V)>>>>>,
    len: usize,
}

impl<K, V> LinkedHashMap<K, V>
where
    K: Hash + Clone + Default + PartialEq + Eq,
    V: Clone + Default + PartialEq + Eq,
{
    fn new() -> Self {
        Self {
            dll: DoubleLinkedList::new(),
            hm: HashMap::new(),
            len: 0,
        }
    }
}
```

可以看到，链式哈希表包含`dll`和`hm`这两个字段。其中，`dll`表示双向链表对象，`hm`表示哈希表对象。为了实现时间复杂度为 O(1) 的增删查改操作，我们使用哈希表，为键和相关双向链表节点之间建立映射。这样，我们就能通过一个键来快速对双向链表元素进行操作了。

接下来，为链式哈希表实现这些方法：

```rust
fn insert_front(&mut self, key: K, value: V) {
    let node = Some(Rc::new(RefCell::new(Node::new((key.clone(), value)))));

    self.dll.push_front(node.clone());
    self.hm.insert(key, node);

    self.len += 1;
}

fn insert_back(&mut self, key: K, value: V) {
    let node = Some(Rc::new(RefCell::new(Node::new((key.clone(), value)))));

    self.dll.push_back(node.clone());
    self.hm.insert(key, node);

    self.len += 1;
}

fn move_to_front(&mut self, key: K) {
    let node = self.hm.get(&key).unwrap();

    self.dll.pop(node);
    self.dll.push_front(node.clone());
}

fn move_to_back(&mut self, key: K) {
    let node = self.hm.get(&key).unwrap();

    self.dll.pop(node);
    self.dll.push_back(node.clone());
}

fn get(&self, key: K) -> Option<V> {
    self.hm
        .get(&key)
        .map(|node| node.as_ref().unwrap().borrow().element.1.clone())
}

fn get_front(&self) -> (K, V) {
    self.dll
        .head
        .as_ref()
        .unwrap()
        .borrow()
        .next
        .as_ref()
        .unwrap()
        .borrow()
        .element
        .clone()
}

fn get_back(&self) -> (K, V) {
    self.dll
        .tail
        .as_ref()
        .unwrap()
        .borrow()
        .prev
        .as_ref()
        .unwrap()
        .borrow()
        .element
        .clone()
}

fn contains_key(&self, key: K) -> bool {
    self.hm.contains_key(&key)
}

fn remove(&mut self, key: K) -> (K, V) {
    let deleted_node = self.hm.get(&key).unwrap().clone();

    self.dll.pop(&deleted_node);
    self.hm.remove(&key);

    self.len -= 1;

    deleted_node.unwrap().borrow().element.clone()
}

fn len(&self) -> usize {
    assert_eq!(self.len, self.dll.len());
    assert_eq!(self.len, self.hm.len());

    self.len
}
```

使用方法普通哈希表相似，直接用即可。

## 实现 LRU

根据题目要求，我们需要完善该框架：

```rust
struct LRUCache {

}


/**
 * `&self` means the method takes an immutable reference.
 * If you need a mutable reference, change it to `&mut self` instead.
 */
impl LRUCache {

    fn new(capacity: i32) -> Self {

    }

    fn get(&self, key: i32) -> i32 {

    }

    fn put(&self, key: i32, value: i32) {

    }
}

/**
 * Your LRUCache object will be instantiated and called as such:
 * let obj = LRUCache::new(capacity);
 * let ret_1: i32 = obj.get(key);
 * obj.put(key, value);
 */
```

有了前面的链式哈希表，我们就可以直接填充它了：

```rust
struct LRUCache {
    cache: LinkedHashMap<i32, i32>,
    cap: usize,
}

impl LRUCache {
    fn new(capacity: i32) -> Self {
        Self {
            cache: LinkedHashMap::new(),
            cap: capacity as usize,
        }
    }

    fn get(&mut self, key: i32) -> i32 {
        if !self.cache.contains_key(key) {
            return -1;
        }

        self.cache.move_to_front(key);
        self.cache.get(key).unwrap()
    }

    fn put(&mut self, key: i32, value: i32) {
        if self.cache.contains_key(key) {
            self.cache.remove(key);
            self.cache.insert_front(key, value);

            return;
        }

        self.cache.insert_front(key, value);

        if self.cache.len() > self.cap {
            let back = self.cache.get_back();
            self.cache.remove(back.0);
        }
    }
}
```

这里我们规定，双向链表首端代表最新数据，尾端代表最旧数据。当然，读者也可以反过来，只需要把`get`和`put`函数中有关`front`和`back`的方法换成与其效果相反的那个就可以了。

这里说一句题外话：或许读者将本文代码提交到 LeetCode 后会发现，这种实现所消耗的运行时间有些长了。确实，笔者在实现相关数据结构时，并未考虑性能优化问题。以双向链表节点实现为例，如果我们需要提升性能，那么我们就需要把双向链表指针实现改为`Option<NonNull<Node<E>>>`（这也是 Rust 标准库所采用的实现），这样就不会产生`Rc`和`RefCell`的运行时性能开销了。当然，正如前文所述，本文的重点在于讲解实现思路。因此，在确保自己的实现没有 bug 后，读者可以自行尝试优化实现，提升性能。

## 实现 LFU

实现 LFU 比实现 LRU 略微复杂。但有了前面的链式哈希表，实现 LFU 相对就没有那么麻烦了。

根据题目要求，我们需要完善该框架：

```rust
struct LFUCache {

}


/**
 * `&self` means the method takes an immutable reference.
 * If you need a mutable reference, change it to `&mut self` instead.
 */
impl LFUCache {

    fn new(capacity: i32) -> Self {

    }

    fn get(&self, key: i32) -> i32 {

    }

    fn put(&self, key: i32, value: i32) {

    }
}

/**
 * Your LFUCache object will be instantiated and called as such:
 * let obj = LFUCache::new(capacity);
 * let ret_1: i32 = obj.get(key);
 * obj.put(key, value);
 */
```

这里，我们选择使用三哈希表的方式来完成此题。首先，我们先设计好基本的数据结构：

```rust
struct LFUCache {
    key_to_value: HashMap<i32, i32>,
    key_to_freq: HashMap<i32, i32>,
    freq_to_keys: HashMap<i32, LinkedHashMap<i32, ()>>,
    min_freq: i32,
    cap: usize,
}

impl LFUCache {
    fn new(capacity: i32) -> Self {
        Self {
            key_to_value: HashMap::new(),
            key_to_freq: HashMap::new(),
            freq_to_keys: HashMap::new(),
            min_freq: 0,
            cap: capacity as usize,
        }
    }
}
```

其中，`key_to_value`管理普通的键值映射，`key_to_freq`管理键与相关的频率的映射，`freq_to_keys`管理频率与键的映射。由于一个频率很可能对应多个键，而且在删除频率最低的键时，若存在多个键，则需删除最久未使用的那个。因此，这里需要使用链式哈希集合来管理这些键。又因为链式哈希集合就是无值链式哈希表，因此我们可以直接使用链式哈希表来构建该数据结构。

在开始实现`get`和`put`方法前，我们首先需要实现两个辅助方法：`increase_freq`和`remove_min_freq_key`：

```rust
fn increase_freq(&mut self, key: i32) {
    // 如果没有这个键，则不进行任何操作
    if !self.key_to_freq.contains_key(&key) {
        return;
    }

    // 根据键取出与其对应的频率
    let freq = *self.key_to_freq.get(&key).unwrap();
    // 为键更新频率
    self.key_to_freq.insert(key, freq + 1);

    // 根据频率取出与其对应的链式哈希集合，并删除与其对应的元素
    self.freq_to_keys.get_mut(&freq).unwrap().remove(key);
    // 如果删除后该集合为空，则删除该频率——集合映射
    if self.freq_to_keys.get(&freq).unwrap().len() == 0 {
        self.freq_to_keys.remove(&freq);

        // 如果删除的映射正好是表示最低频率的映射，那么增大最低频率
        if freq == self.min_freq {
            self.min_freq += 1;
        }
    }

    // 向新频率——集合映射添加元素
    self.freq_to_keys
        .entry(freq + 1)
        .or_insert(LinkedHashMap::new())
        .insert_front(key, ());
}

fn remove_min_freq_key(&mut self) {
    // 取出最低频率所对应的键
    let keys = self.freq_to_keys.get_mut(&self.min_freq).unwrap();

    // 删除集合中最久未使用的键
    let node = keys.dll.pop_back();
    let key = node.unwrap().borrow().element.0;
    keys.hm.remove(&key);
    keys.len -= 1;
    // 如果删除后该集合为空，则删除该频率——集合映射
    if keys.len() == 0 {
        self.freq_to_keys.remove(&self.min_freq);
    }

    self.key_to_value.remove(&key);
    self.key_to_freq.remove(&key);
}
```

接下来便可以实现`get`和`put`方法了：

```rust
fn get(&mut self, key: i32) -> i32 {
    if !self.key_to_value.contains_key(&key) {
        return -1;
    }

    self.increase_freq(key);
    self.key_to_value[&key]
}

fn put(&mut self, key: i32, value: i32) {
    // 如果容量为0，则不进行任何操作
    if self.cap == 0 {
        return;
    }

    // 如果已有键存在，则更新键值映射，且更新键的频率
    if self.key_to_value.contains_key(&key) {
        *self.key_to_value.get_mut(&key).unwrap() = value;
        self.increase_freq(key);

        return;
    }

    // 如果容量已满，则移除最久未使用的键
    if self.key_to_value.len() >= self.cap {
        self.remove_min_freq_key();
    }

    // 在三个哈希表中插入新项
    self.key_to_value.insert(key, value);
    self.key_to_freq.insert(key, 1);
    self.freq_to_keys
        .entry(1)
        .or_insert(LinkedHashMap::new())
        .insert_front(key, ());

    // 由于插入的新项默认频率是1，因此整个缓存中的最低频率便（重新）设置为1
    self.min_freq = 1;
}
```

至此，LFU 实现完毕。

## 总结

不难看出，虽然总体上 LRU 和 LFU 的实现比较困难，但只要把大问题分解成几个小问题并逐步解决，那么难度便会骤然下降了。读者可以尝试自己多实现几次，也可以尝试用其他语言再手动实现一次（无论语言是否有提供内置链式哈希表）。

那么，这篇文章就到这里了！希望读者读完本文后可以加深对双向链表、哈希表、LRU 和 LFU 的理解！

[^1]: 要想利用语言内置的双向链表和哈希表手动实现链式哈希表，一个最重要的条件是，我们要能很方便地获取并存储指向双向链表节点的指针，以保证增删查改的时间复杂度均为 O(1)。

[^2]: 虽然 .NET 已内置[`OrderedDictionary`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.ordereddictionary-2)，但由于其底层数据结构为数组和哈希表，因此，其`Remove`方法时间复杂度为 O(N)，不符合链式哈希表的第二个特点。

[^3]: crates.io 上有一个社区实现的[`IndexMap`](https://crates.io/crates/indexmap)，它可以保证插入元素的顺序。但由于其底层数据结构也为数组和哈希表（和 .NET 的`OrderedDictionary`一样），因此，其`shift_remove`方法时间复杂度为 O(N)，而`swap_remove`方法会破坏元素插入顺序，均不符合链式哈希表的第二个特点。

[^4]: 虽然 Rust 也可以直接使用裸指针，即`*const Type`和`*mut Type`，但 Rust 不鼓励使用它们。除非需要与外部函数打交道，或需要极致性能优化，否则使用`Option<Rc<RefCell<Node<E>>>>`足矣。在保证性能与裸指针不会相差较大的同时，还能最大限度地保障内存安全。
