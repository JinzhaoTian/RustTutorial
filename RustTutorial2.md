# 一. 迭代器与闭包
Rust 的设计灵感来源于很多现存的语言和技术。其中一个显著的影响就是**函数式编程**（functional programming）。函数式编程风格通常包含将函数作为参数值或其他函数的返回值、将函数赋值给变量以供之后执行等等。
## 1. 闭包
闭包（Closure）也叫Lambda表达式或匿名函数。

不像普通函数，闭包可以对参数和返回类型进行推断，大多数时候都不需要写出来。以下定义都是合法的：
```rust
|| 42;
|x| x + 1;
|x:i32| x + 1;
|x:i32| -> i32 { x + 1 };
```
在上面的例子中，如果是单行语句且没有标注返回类型的时候，花括号是可选的。

闭包可以像任何其他对象一样绑定到某个变量：
```rust
let f = |x| x + 1;
```
然后可以像调用函数一样调用闭包：
```rust
f(10);
```
也可以在定义的地方直接调用：
```rust
let r = (|x| x + 1)(2); // r == 3
```

闭包可以捕获外部的环境变量（自由变量）。

闭包捕获变量的方式分为三类：引用（&T）、可变引用（&mut T）和值（T）。

捕获变量时，闭包会根据上面列出的顺序（从约束最少到约束最多），优先按引用捕获，必要时才会使用后面的捕获方式：
```rust
let x = 10;
// 闭包按引用捕获变量x，因为println!只需要引用参数
let show_x = || println!("x = {}", x);
show_x();
```
外部变量的引用保存在show_x对象中，对外部变量的借用持续到show_x离开作用域为止。

下面是一个捕获可变引用的例子：
```rust
let mut count = 0;
// 闭包按可变引用捕获变量count 
// incr也必须是可变的，因为它持有可变引用，调用incr会改变闭包的状态
let mut incr = || { count += 1; println!("count = {}", count); };
incr();
```
下面的代码演示了闭包转移捕获变量所有权时的情况（捕获变量的值）：
```rust
// b是不可复制类型，因此按值捕获时所有权会转移
let b = String::from("Hello World");
let f = move || {
    println!("b = {}", b);
};
f();

println!("b = {}", b);  // 再访问b，会报错
```
由于`String`的所有者只有一个，所以，在捕获变量的值的时候 `b` 的所有权被转移进了闭包内。如果 `b` 是 `i32` 类型的话，仍然可以访问。

闭包另外有一些高级用法，有兴趣可以了解一下书。

## 2. 迭代器
迭代器模式允许你对一个项的序列进行某些处理。迭代器（iterator）负责遍历序列中的每一项和决定序列何时结束的逻辑。当使用迭代器时，我们无需重新实现这些逻辑。

在 Rust 中，迭代器是 惰性的（lazy），这意味着在调用方法使用迭代器之前它都不会有效果。
```rust
let v1 = vec![1, 2, 3];

let v1_iter = v1.iter();

for val in v1_iter {
    println!("Got: {}", val);
}
```
### 1). `Iterator` trait 和 `next` 方法
迭代器都实现了一个叫做 Iterator 的定义于标准库的 trait。这个 trait 的定义看起来像这样：
```rust
pub trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;

    // 此处省略了方法的默认实现
}
```
注意这里有一下我们还未讲到的新语法：`type Item` 和 `Self::Item`，他们定义了 trait 的 关联类型（associated type）。第十九章会深入讲解关联类型，不过现在只需知道这段代码表明实现 `Iterator` trait 要求同时定义一个 `Item` 类型，这个 `Item` 类型被用作 `next` 方法的返回值类型。换句话说，`Item` 类型将是迭代器返回元素的类型。

`next` 是 `Iterator` 实现者被要求定义的唯一方法。`next` 一次返回迭代器中的一个项，封装在 `Some` 中，当迭代器结束时，它返回 `None`。

可以直接调用迭代器的 `next` 方法:
```rust
#[test]
fn iterator_demonstration() {
    let v1 = vec![1, 2, 3];

    let mut v1_iter = v1.iter();  // 此处v1_iter需要是mut的

    assert_eq!(v1_iter.next(), Some(&1));
    assert_eq!(v1_iter.next(), Some(&2));
    assert_eq!(v1_iter.next(), Some(&3));
    assert_eq!(v1_iter.next(), None);
}
```
注意 `v1_iter` 需要是可变的：在迭代器上调用 `next` 方法改变了迭代器中用来记录序列位置的状态。换句话说，代码 消费（consume）了，或使用了迭代器。每一个 `next` 调用都会从迭代器中消费一个项。使用 `for` 循环时无需使 `v1_iter` 可变因为 `for` 循环会获取 `v1_iter` 的所有权并在后台使 `v1_iter` 可变。

另外需要注意到从 `next` 调用中得到的值是vector的不可变引用。`iter` 方法生成一个不可变引用的迭代器。如果我们需要一个获取 `v1` 所有权并返回拥有所有权的迭代器，则可以调用 `into_iter` 而不是 `iter`。类似的，如果我们希望迭代可变引用，则可以调用 `iter_mut` 而不是 `iter`。

### 2). 消费迭代器的方法：
调用 next 方法的方法被称为 消费适配器，因为调用他们会消耗迭代器。比如`sum`方法，这个方法获取迭代器的所有权并反复调用 next 来遍历迭代器，因而会消费迭代器。当其遍历每一个项时，它将每一个项加总到一个总和并在迭代完成时返回总和：
```rust
fn iterator_sum() {
    let v1 = vec![1, 2, 3];

    let v1_iter = v1.iter();

    let total: i32 = v1_iter.sum();   // 6
}
```

# 二. 智能指针
智能指针是Rust中一种特殊的数据结构。它与普通指针的本质区别在于普通指针是对值的借用，而智能指针通常拥有对数据的所有权。在Rust中，如果你想要在堆内存中定义一个对象，并不是像Java中那样直接new一个，也不是像C语言中那样需要手动malloc函数来分配内存空间。Rust中使用的是`Box::new`来对数据进行封箱，而`Box<T>`就是我们今天要介绍的智能指针之一。除了`Box<T>`之外，Rust标准库中提供的智能指针还有`Rc<T>`、`Ref<T>`、`RefCell<T>`等等。在详细介绍之前，我们还是先了解一下智能指针的基本概念。

智能指针通常使用结构体实现。智能指针区别于常规结构体的显著特性在于其实现了 `Deref` 和 `Drop` trait。`Deref` trait 允许智能指针结构体实例表现的像引用一样，这样就可以编写既用于引用、又用于智能指针的代码。`Drop` trait 允许我们自定义当智能指针离开作用域时运行的代码。

## 1. `Box<T>`
最简单直接的智能指针是 `box`，其类型是 `Box<T>`。 `box` 允许你将一个值放在堆上而不是栈上。留在栈上的则是指向堆数据的指针。
```rust
fn main() {
    let b = Box::new(5);
    println!("b = {}", b);
}
```
定义了变量 `b`，其值是一个指向被分配在堆上的值 `5` 的 `Box`。`b` 这样的 box 在 `main` 的末尾离开作用域时，它将被释放。这个释放过程作用于 box 本身（位于栈上）和它所指向的数据（位于堆上）。

## 2. `Rc<T>`
在前面的学习中，我们知道Rust中一个值在同一时间只能有一个变量拥有其所有权，但有时我们可能会需要多个变量拥有所有权，例如在图结构中，两个图可能对同一条边拥有所有权。

对于这样的情况，Rust为我们提供了智能指针Rc（reference counting）来解决共享所有权的问题。每当我们通过Rc共享一个所有权时，引用计数就会加一。当引用计数为0时，该值才会被析构。

## 3. `RefCell<T>`
前文中我们聊过变量的可变性和不可变性，主要是针对变量的。按照前面所讲的，对于结构体来说，我们也只能控制它的整个实例是否可变。实例的具体某个成员是否可变我们是控制不了的。但在实际开发中，这样的场景也是比较常见的。比如我们有一个User结构体：
```rust
struct User {
    id: i32,
    name: str,
    age: u8,
}
```
通常情况下，我们只能修改一个人的名称或者年龄，而不能修改用户的id。如果我们把User的实例设置成了可变状态，那就不能保证别人不会去修改id。

为了应对这种情况，Rust为我们提供了Cell<T>和RefCell<T>。它们本质上不属于智能指针，而是可以提供内部可变性的容器。内部可变性实际上是一种设计模式，它的内部是通过一些unsafe代码来实现的。

我们先来看一下Cell<T>的使用方法:
```rust
use std::cell::Cell;
struct Foo {
    x: u32,
    y: Cell<u32>,
}

fn main() {
    let foo = Foo { x: 1, y: Cell::new(3)};
    assert_eq!(1, foo.x);
    assert_eq!(3, foo.y.get());
    foo.y.set(5);
    assert_eq!(5, foo.y.get());
}
```
我们可以使用Cell的set/get方法来设置/获取起内部的值。这有点像我们在Java实体类中的setter/getter方法。这里有一点需要注意：Cell<T>中包裹的T必须要实现Copy才能够使用get方法，如果没有实现Copy，则需要使用Cell提供的get_mut方法来返回可变借用，而set方法在任何情况下都可以使用。由此可见Cell并没有违反借用规则。

对于没有实现Copy的类型，使用Cell<T>还是比较不方便的，还好Rust还提供了`RefCell<T>`。话不多说，我们直接来看代码:
```rust
use std::cell::RefCell;
fn main() {
    let x = RefCell::new(vec![1, 2, 3]);
    println!("{:?}", x.borrow());
    x.borrow_mut().push(5);
    println!("{:?}", x.borrow());
}
```
从上面这段代码中我们可以观察到`RefCell<T>`的borrow_mut和borrow方法对应了`Cell<T>`中的set和get方法。

`RefCell<T>`和`Cell<T>`还有一点区别是：`Cell<T>`没有运行时开销（不过也不要用它包裹大的数据结构），而`RefCell<T>`是有运行时开销的，这是因为使用`RefCell<T>`时需要维护一个借用检查器，如果违反借用规则，则会引起线程恐慌。

# 三. 无畏并发
安全且高效的处理并发编程是 Rust 的另一个主要目标。
## 1. 线程
Rust 标准库只提供了 1:1 线程模型实现。但是可以通过`crate`包来做m:n的线程模型。
### 1). 调用线程
为了创建一个新线程，需要调用 thread::spawn 函数并传递一个闭包，并在其中包含希望在新线程运行的代码。
```rust
use std::thread;
use std::time::Duration;

fn main() {
    thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {} from the spawned thread!", i);
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
        thread::sleep(Duration::from_millis(1));
    }
}
```
`thread::sleep` 调用强制线程停止执行一小段时间，这会允许其他不同的线程运行。

在这里子线程是打印不完的：主线程打印完之后，就直接结束程序了

### 2). 使用 join 等待所有线程结束:
```rust
use std::thread;
use std::time::Duration;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {} from the spawned thread!", i);
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
        thread::sleep(Duration::from_millis(1));
    }

    handle.join().unwrap();
}
```
可以通过将 `thread::spawn` 的返回值储存在变量中来修复新建线程部分没有执行或者完全没有执行的问题。`thread::spawn` 的返回值类型是 `JoinHandle`。`JoinHandle` 是一个拥有所有权的值，当对其调用 `join` 方法时，它会等待其线程结束。通过调用 handle 的 join 会阻塞**当前线程**直到 handle 所代表的线程结束。

## 2. 消息传递
Rust 中一个实现消息传递并发的主要工具是 通道（channel）。
```rust
use std::thread;
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {   // 子线程发送
        let val = String::from("hi");
        tx.send(val).unwrap();
    });

    let received = rx.recv().unwrap();  // 主线程接收
    println!("Got: {}", received);
}
```
首先使用标准库中的包`mpsc`( 多个生产者，单个消费者（multiple producer, single consumer）的缩写)，Rust 标准库实现通道的方式意味着一个通道可以有多个产生值的 发送（sending）端，但只能有一个消费这些值的 接收（receiving）端。

多个发送端可以通过`clone`来实现：
```rust
let tx1 = mpsc::Sender::clone(&tx);
```

`mpsc::channel` 函数返回一个元组：第一个元素是发送端，而第二个元素是接收端。

使用 `thread::spawn` 来创建一个新线程并使用 `move` 将 `tx` 移动到闭包中这样新建线程就拥有 `tx` 了。

通道的发送端有一个 send 方法用来获取需要放入通道的值。`send` 方法返回一个 `Result<T, E>` 类型，为了防止出错调用 `unwrap` 产生 `panic`。

通道的接收端有两个有用的方法：`recv` 和 `try_recv`。这里，我们使用了 `recv`，它是 receive 的缩写。这个方法会阻塞主线程执行直到从通道中接收一个值。一旦发送了一个值，`recv` 会在一个 `Result<T, E>` 中返回它。当通道发送端关闭，`recv` 会返回一个错误表明不会再有新的值到来了。

`try_recv` 不会阻塞，相反它立刻返回一个 `Result<T, E>`：Ok 值包含可用的信息，而 Err 值代表此时没有任何消息。

### ** 通道与转移所有权
`send` 函数获取其参数的所有权并移动这个值归接收者所有。

## 3. 共享状态

Rust使用互斥器来进行共享内存操作，相当于《操作系统》里说过的PV操作。Rust里面有`Mutex<T>`API调用锁
```rust
use std::sync::Mutex;

fn main() {
    let m = Mutex::new(5);   // 首先给内存上锁

    {
        let mut num = m.lock().unwrap();  // 获得锁
        *num = 6;
    }                                     // 离开域自动上锁

    println!("m = {:?}", m);
}
```
