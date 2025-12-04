# 🧬 第十章：泛型、Trait 与 生命周期

这是 Rust 的“抽象三剑客”。掌握它们，你才能写出复用性极高、类型安全且零成本抽象的代码。

## 1. 泛型 (Generics)：类型的替身

**核心思想**：泛型允许我们编写可以处理多种具体数据类型的代码，而无需为每种类型重复编写逻辑。通过使用抽象的**类型参数 (Type Parameter)**（通常命名为 `T`），我们可以定义函数、结构体或枚举的行为，而不受限于具体的类型。

### 🌰 代码实例：寻找最大值

假设我们要写一个函数，找出数组中最大的那个数。

**不使用泛型（重复造轮子）**：
我们需要为 `i32` 写一个，为 `char` 写一个，逻辑完全一样，只是类型签名不同。
```rust
fn largest_i32(list: &[i32]) -> i32 {
    let mut largest = list[0];
    for &item in list {
        if item > largest { largest = item; }
    }
    largest
}

fn largest_char(list: &[char]) -> char {
    let mut largest = list[0];
    for &item in list {
        if item > largest { largest = item; }
    }
    largest
}
```

**使用泛型（一次编写，到处运行）**：
我们定义一个函数，接受类型 `T`。
*注意：这里用到了 Trait Bound，因为不是所有类型都能比较大小（需要 `PartialOrd`），也不是所有类型都能移动（需要 `Copy`）。*

```rust
// <T> 声明我们要用一个泛型参数 T
// list: &[T] 表示这是一个由 T 类型元素组成的切片
// T: PartialOrd + Copy 是约束，要求 T 必须能比较大小且能进行 Copy
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T {
    let mut largest = list[0];

    for &item in list {
        // 因为 T 实现了 PartialOrd，所以可以使用 > 运算符
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];
    let result = largest(&number_list);
    println!("The largest number is {}", result);

    let char_list = vec!['y', 'm', 'a', 'q'];
    let result = largest(&char_list);
    println!("The largest char is {}", result);
}
```

### 🎭 单态化 (Monomorphization)：零成本抽象的秘密

使用泛型**不会造成任何运行时的性能损耗**。这是因为 Rust 编译器在编译期执行了**单态化**。

1.  **扫描**：编译器会扫描你的代码，查看你到底用哪些具体类型调用了泛型函数（比如上面的 `i32` 和 `char`）。
2.  **生成**：编译器会自动生成针对这些具体类型的专用函数版本（例如生成 `largest_for_i32` 和 `largest_for_char`）。
3.  **替换**：将所有泛型调用替换为对具体函数的调用。

**结果**：编译出来的机器码中**没有泛型**，全是具体类型的高效指令。这被称为“静态分发 (Static Dispatch)”。代价是生成的二进制文件体积会略微增大。

---

## 2. Trait：行为的契约

Trait 是 Rust 实现多态（Polymorphism）的唯一手段。它对应其他语言的 Interface（Java）或 Abstract Class（C++），但更灵活。

### 📜 核心概念
1.  **定义行为**：`trait Summary { fn summarize(&self) -> String; }`。
    *   这是一份合同：谁想当 `Summary`，谁就得会 `summarize`。
2.  **实现行为**：`impl Summary for NewsArticle { ... }`。
    *   这是签字画押：`NewsArticle` 承诺它会 `summarize`。
3.  **默认实现**：可以给 Trait 方法写个默认的 body。
    *   *好处*：实现者可以偷懒，不写这个方法就用默认的。

### 🔗 Trait Bound (Trait 约束)
这是泛型和 Trait 结合的威力所在。如果不加约束，泛型 `T` 是什么都干不了的（比如不能加减乘除，不能打印）。

```rust
// 写法 1：impl Trait 语法糖（简单直观）
fn notify(item: &impl Summary) { ... }

// 写法 2：Trait Bound 完整写法（适合复杂情况，强制多个参数同一类型）
fn notify<T: Summary>(item: &T) { ... }

// 写法 3：多重约束 (+)
// 要求 item 既能 Summary 又能 Display
fn notify(item: &(impl Summary + Display)) { ... }

// 写法 4：Where 从句（最整洁，推荐用于复杂签名）
fn some_function<T, U>(t: &T, u: &U) -> i32
where
    T: Display + Clone,
    U: Clone + Debug,
{ ... }
```

> **大师点拨**：Trait Bound 的意思是告诉编译器：“我不在乎 `T` 到底是啥，只要它能 `Display` 并且能 `Clone`，我就能处理它。”

---

## 3. 生命周期 (Lifetimes)：引用的有效期证明

这是 Rust 最独特也最令人头秃的概念。
**一句话真相**：生命周期标注**不会改变**任何东西的实际存活时间。它只是用来**帮编译器进行检查**，确保程序中没有任何悬垂引用（Dangling References）。

### 💀 为什么需要它？
编译器有一个“借用检查器”，它必须确保**所有的引用都是有效的**。
大多数时候，编译器能自动推导（生命周期省略规则）。但在某些复杂情况下，编译器看不懂了，需要你手工标注。

**经典场景**：函数返回两个引用中的一个。
```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() { x } else { y }
}
// ❌ 报错：missing lifetime specifier
```
**编译器困惑了**：编译器无法在编译期知道 `if` 会走哪条路。它不知道返回的引用是指向 `x` 还是 `y`。如果 `x` 和 `y` 的存活时间不一样，编译器就无法保证返回值的安全性。

### 🔖 标注语法 `'a`
我们需要给编译器一个“承诺”，明确输入和输出引用的关系。

```rust
// 读作：x、y 和返回值，至少都得活得像 'a 这么长
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

**解释**：
1.  泛型生命周期 `'a`。
2.  **约束**：输入引用 `x` 和 `y` 的生命周期必须覆盖 `'a`。
3.  **推导**：实际调用时，`'a` 会被取为 `x` 和 `y` 中**寿命较短**的那一个。
4.  **结果**：返回值的引用，其有效期也被限制为 `'a`（较短的那个）。这样编译器就能保证：只要在 `'a` 范围内使用返回值，就绝对安全。

### 🚫 结构体中的生命周期
如果结构体里存了引用，那结构体本身必须被标注。
```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}
```
**含义**：`ImportantExcerpt` 的实例不能比它里面的 `part` 引用活得更久。如果 `part` 指向的数据被销毁了，这个结构体实例也必须失效。

### 🌟 特殊生命周期：`'static`
*   **含义**：这个引用可以活得跟整个程序一样久。
*   **例子**：所有的字符串字面量 `let s: &'static str = "hello";` 都是静态生命周期，因为它们被硬编码在二进制数据段中，程序运行期间一直存在。

---

## 🎓 总结

这三者构成了 Rust 的抽象基石：
1.  **泛型**：让代码适配不同**数据类型**（Data Types），通过单态化实现零成本。
2.  **Trait**：让代码适配不同**行为**（Behaviors），是实现多态的契约。
3.  **生命周期**：让编译器能验证**引用**（References）的安全性，防止悬垂指针。
