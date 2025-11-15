
## 泛型

**泛型是具体类型与其他属性的抽象替代**

通过泛型，我们可以对一些重复性的函数进行重复性消除，将泛型放置在函数签名中通常用于指定参数和返回值类型的地方

通过泛型，我们的函数，结构体，枚举，方法可以接受各种各样类型，比如可以同时接受i32类型和i64类型的实例的创建

**单态化**：Rust在编译时会将泛型代码转换为特定代码，既为泛型函数填入使用过的具体类型，因此不会过度降低性能

## Trait（全部是AI内容，书我看着真的吃力）

好，咱们从一个简单例子出发，通俗易懂地聊聊 Rust 中的 `trait` 是什么。

### 什么是 `trait`？

`trait` 就像是一个标准或者约定，定义了一组方法，任何类型只要实现了这个 `trait`，就承诺可以提供这些方法的具体实现。你可以把 `trait` 想象成一个“功能接口”或“操作指南”。

举个现实生活的例子吧：  
假设你是一个游戏开发者。游戏中有各种角色，像：猎人、魔法师、战士等等。无论哪种角色，他们都有各自的攻击方式，对吧？  
所以你可以有一个“攻击者的规范”叫做 `Attacker`，这样规定：
- “任何能攻击的单位都必须实现一个方法——比如 `attack` 方法。”

然后，猎人可以定义自己的 `attack` 为射箭，魔法师可以使用火球术，而战士使用剑砍。这就是 Rust 中 `trait` 的核心思想。

---

### 用代码实现模拟这个例子：

这里用代码实现一下，看看 `trait` 是怎样起作用的：

```rust
// 定义一个 Trait，描述一个可攻击的行为
trait Attacker {
    fn attack(&self);
}

// 一个猎人角色
struct Hunter;

// 为猎人角色实现 Attacker Trait
impl Attacker for Hunter {
    fn attack(&self) {
        println!("猎人射了一支箭！");
    }
}

// 一个魔法师角色
struct Mage;

// 为魔法师角色实现 Attacker Trait
impl Attacker for Mage {
    fn attack(&self) {
        println!("魔法师释放了一个火球！");
    }
}

// 一个战士角色
struct Warrior;

// 为战士角色实现 Attacker Trait
impl Attacker for Warrior {
    fn attack(&self) {
        println!("战士挥动了他的剑！");
    }
}

// 测试它们
fn main() {
    let hunter = Hunter;
    let mage = Mage;
    let warrior = Warrior;

    // 调用它们的 attack 方法
    hunter.attack();
    mage.attack();
    warrior.attack();
}
```

运行这段代码，你会看到：
```
猎人射了一支箭！
魔法师释放了一个火球！
战士挥动了他的剑！
```

---

### 换个生活类比：
假设你是一个公司老板，你招了一堆人，每个人都能写不同种类的文章。你作为老板（程序员），只在乎：“是不是符合写文章的标准（Trait）？”  
- 如果可以写技术文章，就要有“写文章”的能力，用具体代码表示就是实现了“写文章”的 `trait`，例如方法 `write`。
- 这时，不管你找的是技术大佬、网红，还是普通小编，只要他们会实现“写文章”，你就可以统一对他们下达写文章的指令。

---

### 总结
- `trait` 是一组方法的约定。
- 任何类型只要实现了这些约定，就意味着它“支持这个 `trait`”并且可以使用这个能力。
- 通过 `trait`，Rust 可以实现“泛型”的功能，也就是写出能适配多种类型的代码，而不需要关心这些类型的具体实现。

这是个新手友好的介绍，慢慢上手后你会发现 `trait` 是 Rust 非常强大也非常重要的一个特性！

好的，我们继续深入学习 `trait`，这次重点聊聊：

1. **`trait` 的默认实现**  
2. **通过 `trait` 定义接受不同类型的参数的函数（泛型的用法）**  
3. **`trait` 约束的使用和作用**  

我们会用简单的例子加通俗类比来展开。

---

### 1. `trait` 的默认实现

#### 什么是默认实现？
`trait` 不仅可以定义一组方法，还可以给这些方法提供一个默认实现。换句话说，如果某个类型实现了这个 `trait` 而没有提供自己的实现，就会使用默认的实现。

#### 为什么/什么时候使用默认实现？
- 当某些方法的大部分类型都会有相同的实现时，可以提供一个默认实现，减少重复代码。
- 如果类型需要覆盖一部分默认实现，可以再重写（override）。

#### 举例：
假设你开发一个动物园管理系统，定义一个 `Animal` 的 `trait`，所有动物都会有一个 `speak` 方法。默认情况下，动物不会说话，你可以为这个方法提供默认实现。

```rust
// 定义一个 Trait
trait Animal {
    // 定义一个具有默认实现的方法
    fn speak(&self) {
        println!("动物不知道怎么说话！");
    }
}

// 猫实现了 Animal trait，但不提供 speak 的实现
struct Cat;

impl Animal for Cat {}

// 狗也实现了 Animal trait，但它会提供自己的 speak 实现
struct Dog;

impl Animal for Dog {
    fn speak(&self) {
        println!("汪汪汪！");
    }
}

fn main() {
    let cat = Cat;
    let dog = Dog;

    cat.speak(); // 默认实现："动物不知道怎么说话！"
    dog.speak(); // 自己的实现："汪汪汪！"
}
```

#### 输出结果：
```
动物不知道怎么说话！
汪汪汪！
```

**总结：** 默认实现适合有通用逻辑的情况，子类型可以选择复用或者自定义。

---

### 2. 通过 `trait` 定义接收不同类型的参数的函数

在 Rust 中，利用 `trait`，我们可以为函数定义一种“接受特定能力的类型”。只要某个类型实现了某个 `trait`，这个函数就能接受它作为参数。

#### 举例：
假设你继续开发动物园管理系统，定义了一个 `Animal` `trait`，任何实现了 `Animal` 的类型都可以被传递给展示函数，它会调用这些类型的 `speak` 方法。

```rust
// 定义一个 Trait
trait Animal {
    fn speak(&self);
}

// 为具体的类型实现 Trait
struct Cat;

impl Animal for Cat {
    fn speak(&self) {
        println!("喵喵喵！");
    }
}

struct Dog;

impl Animal for Dog {
    fn speak(&self) {
        println!("汪汪汪！");
    }
}

// 定义一个接受所有实现了 Animal Trait 的函数
fn animal_speak<T: Animal>(animal: T) {
    animal.speak();
}

fn main() {
    let cat = Cat;
    let dog = Dog;

    animal_speak(cat); // "喵喵喵！"
    animal_speak(dog); // "汪汪汪！"
}
```

#### 这里的关键点：
- `T: Animal` 表示泛型参数 `T` 必须实现 `Animal` `trait`。
- 这使函数具备多态能力：无论是 `Cat`、`Dog`，还是将来新增的类型（只要实现 `Animal`），都可以传入。

**总结：** 使用 `trait` 可以让函数变得更灵活、多态。

---

### 3. `trait` 约束的用法

#### 什么是 `trait` 约束？
`trait` 约束让我们可以明确要求泛型参数具备哪些功能（即实现了哪些 `trait`）。它起到一种“能力筛选”的作用。

#### 为什么使用 `trait` 约束？
- 泛型类型默认情况下是“万能的”，无法使用任何具体的方法。
- 通过 `trait` 约束，可以限制泛型类型必须实现某些方法，使得代码更加安全、精确。

#### 举例：
现在，你有一个函数需要打印参数信息，你希望这个函数只接受那些实现了 `std::fmt::Display`（格式化打印能力）的类型。

```rust
// 定义一个打印函数，要求泛型 T 必须实现 Display trait
fn display_info<T: std::fmt::Display>(item: T) {
    println!("打印信息：{}", item);
}

fn main() {
    let number = 42;
    let text = "Hello, Rust!";
    
    display_info(number); // "打印信息：42"
    display_info(text); // "打印信息：Hello, Rust!"

    // 这个会报错，因为 Vec<i32> 没有实现 Display trait
    // let vec = vec![1, 2, 3];
    // display_info(vec);
}
```

#### 如何组合多个 `trait` 约束？
如果某个泛型参数需要满足多个 `trait`，你可以用 `+` 将它们组合起来。

比如，你需要实现一个“会打印的动物”功能：

```rust
use std::fmt::Display;

// 定义 Animal Trait
trait Animal {
    fn speak(&self);
}

// 同时要求实现 Animal 和 Display 的泛型
fn describe_animal<T: Animal + Display>(animal: T) {
    println!("{}", animal);
    animal.speak();
}
//泛型（Generic）`T` 是泛型参数，表示这个函数可以接受不同类型的输入，只要这些类型满足特定的条件。泛型让函数可以适配多种类型，由一个函数为多种数据类型服务，提升代码的复用性

// 定义一个类型
struct Dog;

impl Animal for Dog {
    fn speak(&self) {
        println!("汪汪汪！");
    }
}

impl Display for Dog {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "这是一只狗")
    }
}

fn main() {
    let dog = Dog;
    describe_animal(dog);
}
```

#### 输出结果：
```
这是一只狗
汪汪汪！
```

#### 总结：
- `trait` 约束让泛型函数更灵活，但不失类型安全。
- 使用 `+` 可以组合多个 `trait` 约束，让某类型具备多种行为。
- 这里也可以使用where从句来简化trait约束，实际上只是更加美观，增强可读性
---

### 总结：
- `trait` 的默认实现可以提供通用逻辑，类型可以选择性重写。
- `trait` 让函数支持多态，通过泛型实现对不同类型的统一操作。
- `trait` 约束控制泛型参数的行为，让代码更加安全、灵活。

## 