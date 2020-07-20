# 第二部分

## 一. 错误处理

Rust 对可靠性的执着也延伸到了错误处理。错误对于软件来说是不可避免的，所以 Rust 有很多特性来处理出现错误的情况。在很多情况下，Rust 要求你承认出错的可能性，并在编译代码之前就采取行动。这些要求使得程序更为健壮，它们确保了你会在将代码部署到生产环境之前就发现错误并正确地处理它们！

### 1. 不可恢复错误（unrecoverable）

突然有一天，代码出问题了，而你对此束手无策。对于这种情况，Rust 有 `panic!`宏。当执行这个宏时，程序会打印出一个错误信息，展开并清理栈数据，然后接着退出。

我们可以直接调用 `panic!` ：

```rust
fn main() {
    panic!("crash and burn");
}
```

在程序出错的时候也会自动调用 `panic!` ：

```rust
fn main() {
    let v = vec![1, 2, 3];

    v[99];  // 越界错误
}
```

我们可以设置 `RUST_BACKTRACE` 环境变量来得到一个 `backtrace`。`backtrace` 是一个执行到目前位置所有被调用的函数的列表。Rust 的 `backtrace` 跟其他语言中的一样：阅读 `backtrace` 的关键是从头开始读直到发现你编写的文件。这就是问题的发源地。这一行往上是你的代码所调用的代码；往下则是调用你的代码的代码。这些行可能包含核心 Rust 代码，标准库代码或用到的 crate 代码。将 `RUST_BACKTRACE` 环境变量设置为任何不是 0 的值来获取 `backtrace`。

```rust
$ RUST_BACKTRACE=1 cargo run
```

### 2. 可恢复错误（recoverable）

首先介绍`Result`枚举，它定义有如下两个成员，`Ok` 和 `Err`：

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

`T` 和 `E` 是泛型类型参数，`T` 代表成功时返回的 `Ok` 成员中的数据的类型，而 `E` 代表失败时返回的 `Err` 成员中的错误的类型。

rust有一些返回 `Result` 的函数，比如：

```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let f = File::open("hello.txt");

    let f = match f {
        Ok(file) => file,
        Err(error) => match error.kind() {
            ErrorKind::NotFound => match File::create("hello.txt") {
                Ok(fc) => fc,
                Err(e) => panic!("Problem creating the file: {:?}", e),
            },
            other_error => panic!("Problem opening the file: {:?}", other_error),
        },
    };
}
```

还有一种简单的错误处理方式`unwrap`：如果 `Result` 值是成员 `Ok`，`unwrap` 会返回 `Ok` 中的值。如果 `Result` 是成员 `Err`，`unwrap` 会为我们调用 `panic!`。

```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt").unwrap();
}
```

与`unwrap`类似的还有`expect`，它还允许我们选择 `panic!` 的错误信息：

```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt").expect("Failed to open hello.txt");
}
```

## 二. 泛型，特性与生存时间

### 1. 泛型

泛型在类型理论中叫做参数多态（parametric polymorphism），它意味着它们是对于给定参数（parametric）能够有多种形式（poly是多，morph是形态）的函数或类型。有点像C++里面的模板。

```rust
struct Point<T> {
    x: T,
    y: T,
}

let int_origin = Point { x: 0, y: 0 };
let float_origin = Point { x: 0.0, y: 0.0 };
```

为了参数化要定义的函数的签名中的类型，我们需要像给函数的值参数起名那样为这类型参数起一个名字。任何标识符都可以作为类型参数名。不过选择 `T` 是因为 Rust 的习惯是让变量名尽量短，通常就只有一个字母，同时 Rust 类型命名规范是骆驼命名法（CamelCase）。`T` 作为 “type” 的缩写是大部分 Rust 程序员的首选。

当需要在函数体中使用一个参数时，必须在函数签名中声明这个参数以便编译器能知道函数体中这个名称的意义。同理，当在函数签名中使用一个类型参数时，必须在使用它之前就声明它。为了定义泛型版本的 largest 函数，类型参数声明位于函数名称与参数列表中间的尖括号 `<>` 中，像这样：

```rust
fn largest<T>(list: &[T]) -> T {
```

这可以理解为：函数 largest 有泛型类型 T。它有一个参数 list，它的类型是一个 T 值的 slice。largest 函数将会返回一个与 T 相同类型的值。

### 2. Trait: 定义共享的行为

一个类型的行为由其可供调用的方法构成。如果可以对不同类型调用相同的方法的话，这些类型就可以共享相同的行为了。trait 定义是一种将方法签名组合起来的方法，目的是定义一个实现某些目的所必需的行为的集合。

trait有点像C++中的虚函数，但是其中有些不同。下面举例：

```rust
struct Circle {
    x: f64,
    y: f64,
    radius: f64,
}

impl Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * (self.radius * self.radius)
    }
}
```

trait 很有用是因为他们允许一个类型对它的行为提供特定的承诺。就是"我"要实现什么。

```rust
struct Circle {
    x: f64,
    y: f64,
    radius: f64,
}

trait HasArea {
    fn area(&self) -> f64;
}

impl HasArea for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * (self.radius * self.radius)
    }
}
```

可以确定任何实现HasArea将会拥有一个.area\(\)方法。

```rust
trait HasArea {
    fn area(&self) -> f64;
}

struct Circle {
    x: f64,
    y: f64,
    radius: f64,
}

impl HasArea for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * (self.radius * self.radius)
    }
}

struct Square {
    x: f64,
    y: f64,
    side: f64,
}

impl HasArea for Square {
    fn area(&self) -> f64 {
        self.side * self.side
    }
}

fn print_area<T: HasArea>(shape: T) {
    println!("This shape has an area of {}", shape.area());
}

fn main() {
    let c = Circle {
        x: 0.0f64,
        y: 0.0f64,
        radius: 1.0f64,
    };

    let s = Square {
        x: 0.0f64,
        y: 0.0f64,
        side: 1.0f64,
    };

    print_area(c);
    print_area(s);
}
```

trait被impl实现的方法可以重载。

### 3. 生命周期与引用有效性

Rust 中的每一个**引用**都有其生命周期（lifetime），也就是引用保持有效的作用域。大部分时候生命周期是隐含并可以推断的，正如大部分时候类型也是可以推断的一样。类似于当因为有多种可能类型的时候必须注明类型，也会出现引用的生命周期以一些不同方式相关联的情况，所以 Rust 需要我们使用**泛型生命周期参数**来注明他们的关系，这样就能确保运行时实际使用的引用绝对是有效的。

#### 1\). 生命周期避免了悬垂引用

生命周期的主要目标是避免悬垂引用，它会导致程序引用了非预期引用的数据。比如，下面的程序是会出错的：

```rust
{
    let r;

    {
        let x = 5;
        r = &x;
    }

    println!("r: {}", r);
}
```

外部作用域声明了一个没有初值的变量 `r`，而内部作用域声明了一个初值为 5 的变量`x`。在内部作用域中，我们尝试将 `r` 的值设置为一个 `x` 的引用。接着在内部作用域结束后，尝试打印出 `r` 的值。这段代码不能编译因为 `r` 引用的值在尝试使用之前就离开了作用域。

#### 2\). 借用检查器

Rust 编译器有一个**借用检查器**（borrow checker），它比较作用域来确保所有的借用都是有效的。下面是相同的例子不过带有变量生命周期的注释：

```rust
{
    let r;                // ---------+-- 'a
                          //          |
    {                     //          |
        let x = 5;        // -+-- 'b  |
        r = &x;           //  |       |
    }                     // -+       |
                          //          |
    println!("r: {}", r); //          |
}                         // ---------+
```

这里将 `r` 的生命周期标记为 `'a` 并将 `x` 的生命周期标记为 `'b`。如你所见，内部的 `'b` 块要比外部的生命周期 `'a` 小得多。在编译时，Rust 比较这两个生命周期的大小，并发现 `r` 拥有生命周期 `'a`，不过它引用了一个拥有生命周期 `'b` 的对象。程序被拒绝编译，因为生命周期 `'b` 比生命周期 `'a` 要小：被引用的对象比它的引用者存在的时间更短。

#### 3\). 函数中的泛型生命周期

用一个例子演示：编写一个返回两个字符串 slice 中较长者的函数。这个函数获取两个字符串 slice 并返回一个字符串 slice。一旦我们实现了 `longest` 函数，下面的代码应该会打印出 `The longest string is abcd`：

```rust
fn main() {
    let string1 = String::from("abcd");
    let string2 = "xyz";

    let result = longest(string1.as_str(), string2);
    println!("The longest string is {}", result);
}
```

如果像下面这样实现 `longest` 函数，它并不能编译：

```rust
fn longest(x: &str, y: &str) -> &str {  // 在需要返回参数相关的引用时, Rust要求我们为这些引用显式的指定生命周期标记
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

当我们定义这个函数的时候，并不知道传递给函数的具体值，所以也不知道到底是 `if` 还是 `else` 会被执行。我们也不知道传入的引用的具体生命周期，所以也就不能像前面那样通过观察作用域来确定返回的引用是否总是有效。借用检查器自身同样也无法确定，因为它不知道 `x` 和 `y` 的生命周期是如何与返回值的生命周期相关联的。为了修复这个错误，我们将增加泛型生命周期参数来定义引用间的关系**以便借用检查器可以进行分析**。

#### 4\). 生命周期注解语法

生命周期注解并不改变任何引用的生命周期的长短。与当函数签名中指定了泛型类型参数后就可以接受任何类型一样，当指定了泛型生命周期后函数也能接受任何生命周期的引用。生命周期注解描述了多个引用生命周期相互的关系，而不影响其生命周期。

生命周期注解有着一个不太常见的语法：生命周期参数名称必须以撇号（'）开头，其名称通常全是小写，类似于泛型其名称非常短。'a 是大多数人默认使用的名称。生命周期参数注解位于引用的 & 之后，并有一个空格来将引用类型与生命周期注解分隔开。

```rust
&i32        // 引用
&'a i32     // 带有显式生命周期的引用
&'a mut i32 // 带有显式生命周期的可变引用
```

如果想让上面的程序运行，就可以加上注解：

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```


## 三. 迭代器与闭包

Rust 的设计灵感来源于很多现存的语言和技术。其中一个显著的影响就是**函数式编程**（functional programming）。函数式编程风格通常包含将函数作为参数值或其他函数的返回值、将函数赋值给变量以供之后执行等等。

### 1. 闭包

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

外部变量的引用保存在show\_x对象中，对外部变量的借用持续到show\_x离开作用域为止。

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

### 2. 迭代器

迭代器模式允许你对一个项的序列进行某些处理。迭代器（iterator）负责遍历序列中的每一项和决定序列何时结束的逻辑。当使用迭代器时，我们无需重新实现这些逻辑。

在 Rust 中，迭代器是 惰性的（lazy），这意味着在调用方法使用迭代器之前它都不会有效果。

```rust
let v1 = vec![1, 2, 3];

let v1_iter = v1.iter();

for val in v1_iter {
    println!("Got: {}", val);
}
```

#### 1\). `Iterator` trait 和 `next` 方法

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

#### 2\). 消费迭代器的方法：

调用 next 方法的方法被称为 消费适配器，因为调用他们会消耗迭代器。比如`sum`方法，这个方法获取迭代器的所有权并反复调用 next 来遍历迭代器，因而会消费迭代器。当其遍历每一个项时，它将每一个项加总到一个总和并在迭代完成时返回总和：

```rust
fn iterator_sum() {
    let v1 = vec![1, 2, 3];

    let v1_iter = v1.iter();

    let total: i32 = v1_iter.sum();   // 6
}
```

## 四. 智能指针

智能指针是Rust中一种特殊的数据结构。它与普通指针的本质区别在于普通指针是对值的借用，而智能指针通常拥有对数据的所有权。在Rust中，如果你想要在堆内存中定义一个对象，并不是像Java中那样直接new一个，也不是像C语言中那样需要手动malloc函数来分配内存空间。Rust中使用的是`Box::new`来对数据进行封箱，而`Box<T>`就是我们今天要介绍的智能指针之一。除了`Box<T>`之外，Rust标准库中提供的智能指针还有`Rc<T>`、`Ref<T>`、`RefCell<T>`等等。在详细介绍之前，我们还是先了解一下智能指针的基本概念。

智能指针通常使用结构体实现。智能指针区别于常规结构体的显著特性在于其实现了 `Deref` 和 `Drop` trait。`Deref` trait 允许智能指针结构体实例表现的像引用一样，这样就可以编写既用于引用、又用于智能指针的代码。`Drop` trait 允许我们自定义当智能指针离开作用域时运行的代码。

### 1. `Box<T>`

最简单直接的智能指针是 `box`，其类型是 `Box<T>`。 `box` 允许你将一个值放在堆上而不是栈上。留在栈上的则是指向堆数据的指针。

```rust
fn main() {
    let b = Box::new(5);
    println!("b = {}", b);
}
```

定义了变量 `b`，其值是一个指向被分配在堆上的值 `5` 的 `Box`。`b` 这样的 box 在 `main` 的末尾离开作用域时，它将被释放。这个释放过程作用于 box 本身（位于栈上）和它所指向的数据（位于堆上）。

### 2. `Rc<T>`

在前面的学习中，我们知道Rust中一个值在同一时间只能有一个变量拥有其所有权，但有时我们可能会需要多个变量拥有所有权，例如在图结构中，两个图可能对同一条边拥有所有权。

对于这样的情况，Rust为我们提供了智能指针Rc（reference counting）来解决共享所有权的问题。每当我们通过Rc共享一个所有权时，引用计数就会加一。当引用计数为0时，该值才会被析构。

### 3. `RefCell<T>`

前文中我们聊过变量的可变性和不可变性，主要是针对变量的。按照前面所讲的，对于结构体来说，我们也只能控制它的整个实例是否可变。实例的具体某个成员是否可变我们是控制不了的。但在实际开发中，这样的场景也是比较常见的。比如我们有一个User结构体：

```rust
struct User {
    id: i32,
    name: str,
    age: u8,
}
```

通常情况下，我们只能修改一个人的名称或者年龄，而不能修改用户的id。如果我们把User的实例设置成了可变状态，那就不能保证别人不会去修改id。

为了应对这种情况，Rust为我们提供了Cell和RefCell。它们本质上不属于智能指针，而是可以提供内部可变性的容器。内部可变性实际上是一种设计模式，它的内部是通过一些unsafe代码来实现的。

我们先来看一下Cell的使用方法:

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

我们可以使用Cell的set/get方法来设置/获取起内部的值。这有点像我们在Java实体类中的setter/getter方法。这里有一点需要注意：Cell中包裹的T必须要实现Copy才能够使用get方法，如果没有实现Copy，则需要使用Cell提供的get\_mut方法来返回可变借用，而set方法在任何情况下都可以使用。由此可见Cell并没有违反借用规则。

对于没有实现Copy的类型，使用Cell还是比较不方便的，还好Rust还提供了`RefCell<T>`。话不多说，我们直接来看代码:

```rust
use std::cell::RefCell;
fn main() {
    let x = RefCell::new(vec![1, 2, 3]);
    println!("{:?}", x.borrow());
    x.borrow_mut().push(5);
    println!("{:?}", x.borrow());
}
```

从上面这段代码中我们可以观察到`RefCell<T>`的borrow\_mut和borrow方法对应了`Cell<T>`中的set和get方法。

`RefCell<T>`和`Cell<T>`还有一点区别是：`Cell<T>`没有运行时开销（不过也不要用它包裹大的数据结构），而`RefCell<T>`是有运行时开销的，这是因为使用`RefCell<T>`时需要维护一个借用检查器，如果违反借用规则，则会引起线程恐慌。

## 五. 无畏并发

安全且高效的处理并发编程是 Rust 的另一个主要目标。

### 1. 线程

Rust 标准库只提供了 1:1 线程模型实现。但是可以通过`crate`包来做m:n的线程模型。

#### 1\). 调用线程

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

#### 2\). 使用 join 等待所有线程结束:

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

### 2. 消息传递

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

首先使用标准库中的包`mpsc`\( 多个生产者，单个消费者（multiple producer, single consumer）的缩写\)，Rust 标准库实现通道的方式意味着一个通道可以有多个产生值的 发送（sending）端，但只能有一个消费这些值的 接收（receiving）端。

多个发送端可以通过`clone`来实现：

```rust
let tx1 = mpsc::Sender::clone(&tx);
```

`mpsc::channel` 函数返回一个元组：第一个元素是发送端，而第二个元素是接收端。

使用 `thread::spawn` 来创建一个新线程并使用 `move` 将 `tx` 移动到闭包中这样新建线程就拥有 `tx` 了。

通道的发送端有一个 send 方法用来获取需要放入通道的值。`send` 方法返回一个 `Result<T, E>` 类型，为了防止出错调用 `unwrap` 产生 `panic`。

通道的接收端有两个有用的方法：`recv` 和 `try_recv`。这里，我们使用了 `recv`，它是 receive 的缩写。这个方法会阻塞主线程执行直到从通道中接收一个值。一旦发送了一个值，`recv` 会在一个 `Result<T, E>` 中返回它。当通道发送端关闭，`recv` 会返回一个错误表明不会再有新的值到来了。

`try_recv` 不会阻塞，相反它立刻返回一个 `Result<T, E>`：Ok 值包含可用的信息，而 Err 值代表此时没有任何消息。

#### \*\* 通道与转移所有权

`send` 函数获取其参数的所有权并移动这个值归接收者所有。

### 3. 共享状态

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

