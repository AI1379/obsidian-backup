---
tags:
  - 面试
  - 简历
  - CUDA
  - CPP
---

# C++ 与 CUDA 八股提纲

这份文档分两层：

1. `C++` 用 " 你本来就比较熟 " 的标准来准备，会明显比防守版更深
2. `CUDA` 仍然以工程使用和性能直觉为主，但会补到足够应对追问

---

## 一、C++ 主线

如果面试官看到你简历上写熟悉 `C++`，通常默认你至少应该扛住这几条主线：

1. 对象生命周期和资源管理
2. 值类别、拷贝语义、移动语义
3. 模板和泛型
4. STL 容器与复杂度
5. 多态和对象模型
6. 并发、原子、内存模型

---

## 二、C++ 基础必答点

### 1. 指针、引用、`const`

- 指针可以为空，可以改指向
- 引用本质上是别名，通常不能为空，也不能重新绑定
- `const T*` 是 " 指向常量的指针 "
- `T* const` 是 " 常量指针 "
- `const T&` 常用于只读传参，避免不必要拷贝

> **思考题：引用底层是怎么实现的？**
>
> 绝大多数编译器在实现引用时，本质上是把它当作一个自动解引用的常量指针（`T* const`）。编译期生成同样的地址运算代码，但语法层面让你不需要写 `*` 和 `&`。这就是为什么：
> - 引用必须初始化（指针可以不初始化）
> - 引用不能重新绑定（`T* const` 也不能改指向）
> - 对引用的操作等同于对原对象的操作
>
> 但要注意：标准并没有规定引用必须是指针实现的——它只是规定了 " 别名 " 语义。编译器可以用其他方式实现（比如直接优化掉），所以面试时不要说 " 引用就是指针 "。

> **思考题：为什么有 `const T&` 可以绑定到临时对象并延长其生命周期？**
>
> 这是 C++ 标准中一个特殊规则：当一个 const 左值引用（或右值引用）绑定到一个临时对象（纯右值）时，该临时对象的生命周期会被延长到该引用的生命周期。这在函数传参中非常有用：
> ```cpp
> void foo(const std::string& s);
> foo("hello");  // "hello" 隐式构造为临时 string，生命周期延长到 foo 返回
> ```
> 这个规则使传 const 引用既能避免拷贝，又能接受字面量。但注意：只有直接绑定才延长生命周期，间接绑定（如函数返回的引用）不会。

### 2. `new/delete` vs `malloc/free`

- `new` 分配内存并调用构造函数
- `delete` 调用析构函数并释放内存
- `malloc/free` 只负责原始内存，不处理构造析构
- 不能混用，因为分配释放路径和对象语义不匹配

> **思考题：`new` 失败了会怎样？和 `malloc` 有什么区别？**
>
> - `malloc` 失败返回 `NULL`（C 风格）
> - 默认 `new` 失败抛出 `std::bad_alloc` 异常（C++ 风格）
> - `new (std::nothrow)` 版本失败时返回 `nullptr`，类似 malloc
> - 在嵌入式或实时环境中，有时会显式用 nothrow 版本来避免异常开销
>
> 面试里能说出 " 标准 new 抛异常，nothrow new 返回空指针 " 基本就够了。进一步可以提到：C++17 之后 `operator new` 的失败处理和 `std::get_new_handler` 的关系。

### 3. RAII

RAII 是 C++ 面试里最核心的思想之一：

- 资源获取和对象生命周期绑定
- 作用域结束自动释放
- 资源不限于内存，还包括文件、锁、socket、GPU handle

典型例子：

- `unique_ptr`
- `lock_guard`
- 文件句柄包装类

> **思考题：RAII 和 GC 各有什么优劣？为什么 C++ 选择了 RAII 而非 GC？**
>
> RAII 的优势：
> - 释放时机确定（离开作用域），可预测性强，适合实时系统
> - 资源范围不限于内存：锁、文件、socket、GPU 资源都能统一管理
> - 零运行时开销（编译期确定构造/析构位置）
>
> RAII 的劣势：
> - 循环引用需要 `weak_ptr` 打破，不如 GC 自动处理
> - 需要程序员显式管理所有权（unique_ptr / shared_ptr）
>
> GC 的优势：
> - 自动处理循环引用
> - 程序员不需要关心释放时机
>
> GC 的劣势：
> - 释放时机不确定（stop-the-world 等）
> - 只能管理内存，不能管理其他资源（锁、文件等）
> - 运行时开销
>
> C++ 选择 RAII 的根本原因：C++ 设计哲学是 " 不为不用的东西付费 "，以及 " 资源 " 不仅包括内存。GC 无法解决锁释放、文件关闭等问题，最终还是需要类似 RAII 的机制。

### 4. Rule of Three / Five / Zero

- 如果类手动管理资源，至少要考虑析构、拷贝构造、拷贝赋值
- 现代 C++ 还要考虑移动构造、移动赋值，所以有 Rule of Five
- 如果资源交给标准库成员管理，尽量遵循 Rule of Zero

面试官真正想看的是：你知道什么时候应该少写，什么时候必须认真写。

> **思考题：为什么有了析构函数就最好显式处理拷贝构造和拷贝赋值（Rule of Three）？**
>
> 核心原因：如果你写了自定义析构函数，多半意味着你在管理某种资源（堆内存、文件句柄等）。编译器自动生成的拷贝构造和拷贝赋值只做 " 浅拷贝 "（逐成员拷贝），这会导致：
> - 两个对象持有同一个资源的指针/handle
> - 第一个对象析构时释放资源
> - 第二个对象析构时 double-free / use-after-free
>
> 所以如果析构函数非平凡，几乎一定需要自定义拷贝操作来 " 深拷贝 " 或禁止拷贝。Rule of Five 进一步加上了移动操作——因为现代 C++ 中，如果你管理资源，移动语义往往能极大提升性能。

---

## 三、值类别、移动语义、完美转发

### 1. 左值、右值、将亡值

先抓住直觉：

- 左值通常有稳定身份，可取地址
- 纯右值通常是临时值
- 将亡值可以理解为 " 即将被搬走资源的对象 "

如果被追问，不要只背定义，要能说出它们和重载解析、移动语义的关系。

> **思考题：`std::move` 后的对象还能用吗？处于什么状态？**
>
> `std::move` 后的对象处于 " 有效但未指定 "（valid but unspecified）的状态。这意味着：
> - 你仍然可以调用不带前提条件的成员函数（如赋值、析构）
> - 但你不知道它的值是多少（可能为空，可能不是）
> - 你不应该假设它恢复了默认值
>
> 标准库的实践是：被移动后的对象处于 " 未指定 " 但必须是 " 有效 " 的——保证可以安全析构或重新赋值。例如 `std::string` 被移动后可能为空，也可能不改变。唯一可以做的安全操作是：给它赋值新值，或者让它析构。这就是为什么移动操作通常要把源对象的指针成员置 `nullptr`。

### 2. `std::move`

- `std::move` 不负责真的移动
- 它只是把表达式显式转换成右值引用
- 真正是否发生 " 搬资源 " 取决于目标类型有没有合适的移动操作

> **思考题：如果我对一个 `const` 对象调用 `std::move` 会发生什么？**
>
> ```cpp
> const std::string s = "hello";
> std::string s2 = std::move(s);  // 实际上调用的是拷贝构造！
> ```
>
> 因为 `std::move(s)` 返回 `const std::string&&`，它不能绑定到 `std::string&&`（移动构造的参数），但可以绑定到 `const std::string&`（拷贝构造的参数）。所以实际上发生的是拷贝而非移动。这说明了 `move` 不强制移动——最终走哪个重载完全由重载决议决定。

### 3. 移动构造 / 移动赋值

核心目标：

- 避免深拷贝
- 转移资源所有权

典型场景：

- 函数返回大对象
- 容器扩容搬迁元素
- `push_back(std::move(x))`

> **思考题：移动构造函数通常要加 `noexcept`，为什么？**
>
> 以 `std::vector` 为例：当 vector 需要扩容时，需要把旧内存中的元素搬迁到新内存。如果元素类型有 `noexcept` 的移动构造，vector 会选择移动——这只需拷贝指针、把源置空，不会失败。但如果移动构造没标 `noexcept`，vector 会退而求其次用拷贝构造——因为拷贝的过程可以保证 " 即使出问题，原对象仍然完好 "；而移动会在中途改掉源对象，如果移动到一半抛异常，vector 就处于 " 部分被移动、部分还在原位 " 的不可恢复状态。
>
> 所以 `noexcept` 不仅是文档承诺，还直接影响运行时性能——标了 `noexcept` 的移动构造能在容器扩容时被优先使用。

### 4. 万能引用和完美转发

- 形如模板参数推导场景里的 `T&&` 常被叫做 forwarding reference
- `std::forward<T>(x)` 的目的是保留实参原本的值类别
- 这一块常和 `emplace_back`、工厂函数、包装函数一起问

你至少要会这句：

> `move` 是无条件把对象转成右值，`forward` 是有条件地保留原始值类别。

> **思考题：为什么 `std::forward` 需要显式传模板参数 `T`，而 `std::move` 不需要？**
>
> `move` 的行为是确定的：无条件转成右值。所以它只需要推导实参类型。
> `forward` 的行为依赖于 `T`：如果 `T` 是 `int&`（左值引用），forward 应该返回左值引用；如果 `T` 是 `int`（非引用），forward 应该返回右值引用。这个条件判断需要知道原始模板参数 `T` 是引用还是非引用——而 `T` 只存在于模板上下文中，只能通过显式传参告诉 forward。所以 `std::forward` 必须通过 `forward<T>(x)` 的 `<T>` 来接收这个信息。

---

## 四、类、对象模型、多态

### 1. 构造、析构、拷贝、赋值

要分清这些场景：

- 拷贝构造：用已有对象初始化新对象
- 拷贝赋值：用已有对象给已存在对象赋值
- 移动构造 / 移动赋值：转移资源

### 2. 虚函数和运行时多态

- 编译期多态：重载、模板、CRTP
- 运行期多态：虚函数

高频追问：

- 为什么析构函数常写成虚函数
- 为什么不建议在构造 / 析构里做虚调用

> **思考题：为什么析构函数在基类中要写成 virtual？如果有一个继承体系的对象被基类指针删除会怎样？**
>
> ```cpp
> Base* p = new Derived();
> delete p;  // 如果 ~Base() 不是虚函数，只调用 ~Base()，Derived 的资源泄漏
> ```
> 当使用基类指针删除派生类对象时，如果析构函数不是虚函数，编译器只会根据指针的静态类型（Base）来决定调用哪个析构函数。Derived 的析构函数不会被调用，其管理的资源（如 Derived 中的 string、vector 等成员）不会被正确释放。将基类析构函数声明为虚函数后，调用会经过 vtable 动态分派到正确的析构函数（先 ~Derived()，再 ~Base()）。

> **思考题：为什么构造函数里调用虚函数不会发生多态？**
>
> 在构造 Derived 对象时，构造顺序是从基类到派生类。当基类构造函数执行时，Derived 的部分还没有被构造——此时 Derived 的 vptr 还没有设置好，对象的动态类型是 Base。如果此时调用虚函数，只会调用 Base 的版本，不会分派到 Derived。这是标准规定：在构造和析构期间，虚函数的动态类型就是当前正在构造/析构的那个类的类型。

### 3. 对象模型你可以答到哪一层

如果对方开始往深里问，比较稳的说法是：

- 含虚函数的对象里，很多实现会放一个 `vptr`
- `vptr` 指向虚函数表
- 通过基类指针调用虚函数时，会在运行时分派到对应实现

这里要注意：

- `vptr/vtable` 属于常见实现方式，不是标准硬性规定
- 你可以说 " 主流实现通常如此 "，别说成标准明确规定

> **思考题：多重继承下 vptr 怎么布局？菱形继承会有什么问题？**
>
> 多重继承时，一个对象可能有多个 `vptr`——每个基类子对象各有一个。当你做 `Derived* -> Base2*` 的转换时，指针本身的值可能会变（编译器加偏移量），指向对应的基类子对象及其 vptr。这就是为什么 `static_cast` 和 `reinterpret_cast` 在涉及多继承时行为不同。
>
> 菱形继承（A 被 B 和 C 继承，然后 D 继承 B 和 C）的问题：D 中有两份 A 的数据成员，造成二义性和空间浪费。虚继承 `virtual public A` 通过引入一个 `vbptr`（虚基类指针）间接访问共享的 A，解决重复问题，但带来了额外开销（间接访问、构造顺序复杂化）。

### 4. 继承和内存布局

适合更深入一点回答：

- 单继承相对直观
- 多继承会让对象布局和指针调整更复杂
- 空基类优化 `EBO` 是常见考点
- 对齐和 padding 会影响对象大小

你至少应该知道：

- `sizeof` 不只是成员大小相加
- 对齐会插 padding
- 虚函数、多继承、空基类都可能影响布局

> **思考题：空基类优化（EBO）是什么？为什么需要它？**
>
> C++ 标准规定每个对象必须有唯一的地址，所以 sizeof(空类) >= 1。但如果一个空类作为基类被继承，且派生类的第一个成员不是同类型的空基类，编译器可以不为空基类分配独立空间（即空基类子对象的大小为 0）。这避免了空间浪费。`std::unique_ptr` 就利用 EBO：如果删除器是空类（如 `std::default_delete`），unique_ptr 的大小就等于裸指针的大小，不增加额外开销。

---

## 五、模板、类型推导、现代 C++

### 1. 模板到底在解决什么

- 模板是编译期泛型
- 目的不是 " 炫技 "，而是复用逻辑并保留类型信息

### 2. 模板实例化

- 模板不是写出来就变成代码
- 只有在被用到时才会实例化
- 这就是为什么很多模板实现放头文件里（Q: 有没有可以放在源文件里面但是暴露给外部使用的情况？）

> **思考题：模板能不能放在源文件里但暴露给外部使用？**
>
> **可以，但有限制。** 常规的 " 头文件声明 + 源文件定义 " 模式对模板通常不够用，因为模板在编译调用处时需要看到完整定义才能实例化。
>
> 但以下情况可以把模板实现在源文件：
>
> 1. **显式实例化（Explicit Instantiation）**：如果你提前知道模板会被哪些类型使用，可以在源文件中做显式实例化。头文件放声明，源文件放定义 + `template class MyTemplate<int>;`。外部的 `MyTemplate<int>` 能找到实例化后的符号，但如果是 `MyTemplate<double>` 且你没显式实例化它，链接器会报错。
>
> 2. **头文件 include 源文件**：技术上可行但是反模式，不推荐。
>
> 3. **模块（C++20 Modules）**：模块改变了编译模型，模板定义可以放在模块实现单元中，只要在接口单元中正确声明即可。这是真正解决模板必须放头文件问题的方案，但需要编译器和构建系统支持。
>
> 工程结论：除非你能穷举所有模板参数组合（如内部库），否则模板仍然适合放头文件。

### 3. 函数模板、类模板、偏特化、全特化

至少要分清：

- 类模板可以偏特化
- 函数模板不能偏特化，但可以重载

> **思考题：为什么函数模板不能偏特化？**
>
> 这是 C++ 标准委员会的设计选择，主要原因是函数模板本身已经支持重载——偏特化的大部分使用场景都可以用重载来替代。而且如果同时允许偏特化和重载，函数模板的重载决议规则会变得非常复杂（类模板没有重载，只有特化，所以偏特化语义清晰）。实践中：如果你觉得需要一个 " 偏特化的函数模板 "，写一个重载的函数模板，或者把实现委托给一个可偏特化的类模板即可。

### 4. `auto`、`decltype`、`decltype(auto)`

- `auto` 用来做类型推导
- `decltype(expr)` 按表达式规则推导类型
- `decltype(auto)` 更适合保留返回值类型特征，尤其是引用性

> **思考题：`auto` 和 `decltype(auto)` 的核心区别是什么？什么时候必须用 `decltype(auto)`？**
>
> `auto` 推导规则会剥除引用和顶层 const（类似模板参数推导）：
> ```cpp
> int& foo();
> auto x = foo();      // x 是 int，引用性被剥除了
> decltype(auto) y = foo();  // y 是 int&，保留引用性
> ```
>
> `decltype(auto)` 推导规则等同于 `decltype(expr)`。必须用 `decltype(auto)` 的场景：
> - 函数返回值需要保留引用性（比如包装函数需要原封不动返回被包装函数的返回值）
> - 需要保留表达式精确类型（包括值类别）的变量声明
>
> 典型例子是 `std::forward` 的返回值和通用包装函数：
> ```cpp
> template <typename F, typename... Args>
> decltype(auto) wrapper(F&& f, Args&&... args) {
>     return std::forward<F>(f)(std::forward<Args>(args)...);
> }
> // 用 auto 会剥掉引用，可能导致不必要的拷贝或悬垂引用
> ```

### 5. SFINAE、`enable_if`、concepts

如果对方按现代 C++ 深度问，这几个很常见：

- SFINAE 的核心直觉是：模板替换失败不算硬错误，可以退而匹配别的候选
- `enable_if` 是经典约束手段
- `concepts` 是更现代、更可读的模板约束方式

如果你不是天天写库，可以把口径放在：

- 我理解它们解决 " 模板约束和重载选择 " 的问题
- 日常工程里我更多停留在使用层，不会把自己包装成模板元编程专家

> **思考题：SFINAE 的全称和核心机制是什么？举一个实际的工程应用场景？**
>
> SFINAE = Substitution Failure Is Not An Error（替换失败不算错误）
>
> 核心机制：编译器在尝试匹配模板重载时，如果某个候选的模板参数替换失败了（比如某个类型没有某个成员），编译器不会报错，而是静默丢弃这个候选，继续尝试其他重载。
>
> 工程应用场景：
> 1. 编译期检查类型是否有某个方法（如 `.begin()` / `.end()`），从而区分容器和其他类型
> 2. 检查类型是否可安全 `memcpy`（`std::is_trivially_copyable` 内部使用了类似机制）
> 3. C++17 之前的 " 概念式 " 约束类型参数（现在更多被 concepts 替代）
>
> ```cpp
> // C++17 之前的经典 enable_if 用法：仅对整数类型启用
> template <typename T,
>           typename = std::enable_if_t<std::is_integral_v<T>>>
> void process(T val);
> ```

### 6. `constexpr`

- 早期更多是 " 编译期常量表达式 "
- 现代 C++ 下它的能力越来越强，可以做更多编译期计算
- 面试里常问 `const` 和 `constexpr` 区别

一个稳妥回答是：

- `const` 更强调只读
- `constexpr` 更强调在满足条件时可参与编译期求值

> **思考题：C++14/17/20 对 constexpr 分别放宽了什么限制？**
>
> - **C++11**：constexpr 函数只能包含一条 return 语句
> - **C++14**：允许 constexpr 函数中有多条语句、局部变量、循环、条件分支。这让编译期计算变得实用
> - **C++17**：`constexpr` 可以用于 lambda（lambda 默认 constexpr）；`if constexpr` 引入编译期条件分支
> - **C++20**：`constexpr` 可以用于虚函数、`dynamic_cast`、`try-catch`、`std::vector` 和 `std::string` 的构造和析构（在 constexpr 上下文中）；`constinit` 和 `consteval` 引入

---

## 六、STL 和容器常见深问

### 1. `vector`

高频点：

- 连续内存
- 随机访问快
- 尾插摊还 `O(1)`
- 扩容会重新分配并搬迁元素
- 扩容可能导致迭代器、引用、指针失效

追问常见方向：

- 为什么缓存友好
- `reserve` 和 `resize` 区别
- `push_back` 和 `emplace_back` 区别

> **思考题：`push_back` 和 `emplace_back` 的本质区别是什么？什么场景下 `emplace_back` 能提升性能？**
>
> `push_back` 接收的是已经构造好的对象（或能隐式转换的东西），传给 vector 时可能发生一次拷贝或移动。
> `emplace_back` 接收的是构造参数，直接在 vector 内部构造对象——避免了临时对象的创建和后续的拷贝/移动。
>
> ```cpp
> vector<pair<string, int>> v;
> // push_back: 先构造临时 pair，再移动入 vector（或拷贝）
> v.push_back({"hello", 42});
> // emplace_back: 直接在 vector 内存中构造 pair
> v.emplace_back("hello", 42);
> ```
>
> 性能差异最明显的场景：对象不可移动（只能拷贝）、构造参数类型和值类型不同导致隐式转换、对象较大且拷贝昂贵。但如果参数已经是同类型的右值，并且有 noexcept 移动构造，两者性能几乎一致。

> **思考题：`vector<bool>` 有什么特殊之处？为什么它经常被建议避免使用？**
>
> `vector<bool>` 被标准特化成了一个 "packed bits" 结构——每个 bool 只占 1 bit，而不是 1 byte。这导致了：
> - `v[0]` 返回的不是 `bool&`，而是一个代理对象（`vector<bool>::reference`）
> - `auto b = v[0];` 中 `b` 的类型不是 `bool`，是代理类型
> - 不能用 `&v[0]` 获取 bool 的指针
> - 与泛型代码不兼容（对 `vector<T>` 的某些假设在 `T=bool` 时失效）
>
> 替代方案：用 `vector<char>` 或 `deque<bool>`（`deque<bool>` 没有这个特化），或 `boost::container::vector<bool>`。

### 2. `map` vs `unordered_map`

- `map` 通常基于红黑树，有序，`O(log n)`
- `unordered_map` 通常基于哈希表，平均 `O(1)`
- `unordered_map` 会有 rehash，迭代器失效规则也要知道个大概

> **思考题：什么场景下 `map` 反而比 `unordered_map` 更快？**
>
> 1. **数据量非常小**：红黑树的开销可能小于哈希计算 + 碰撞处理
> 2. **需要有序遍历**：`map` 天然有序，`unordered_map` 需要额外排序
> 3. **哈希函数很差**：大量碰撞时 `unordered_map` 退化为 `O(n)`，而 `map` 稳定 `O(log n)`
> 4. **频繁按范围查询**（lower_bound / upper_bound）：`map` 支持这些操作，`unordered_map` 不支持
> 5. **内存受限**：`unordered_map` 的桶数组和链表节点有额外内存开销
>
> 一个实用判断：如果你只需要 " 存进去、查出来 "，用 `unordered_map`；如果你需要有序性、范围查询，或者数据量不确定但需要稳定的最坏情况性能，用 `map`。

### 3. `string`

你至少知道：

- `std::string` 是拥有型字符串容器
- 小字符串优化 `SSO` 是常见实现技巧，但也是实现相关，不是标准强制

> **思考题：SSO 是怎么工作的？为什么现代 C++ 的 `string` 都倾向于实现 SSO？**
>
> SSO（Small String Optimization）：在 `std::string` 对象内部预留一小块内联缓冲区（通常 15-23 个字符，在 64 位系统上恰好利用 `string` 对象本身的存储空间）。短字符串直接存在对象内部，不分配堆内存；长字符串才用堆分配。
>
> 现代实现倾向 SSO 的原因：
> - 大多数程序中的字符串都很短（日志、键、标签等）
> - 避免堆分配意味着更好的缓存局部性和更少的分配/释放开销
> - 配合移动语义：移动一个 SSO 字符串只需 memcpy 对象本身
>
> COW（Copy-on-Write）曾被旧版 libstdc++ 使用，但它和标准要求的某些操作语义有冲突（如 `operator[]` 的 non-const 版本），加上多线程下原子引用计数开销大，现代实现基本都转向了 SSO。

### 4. 迭代器失效

这是很能区分 " 会用 " 和 " 真熟 " 的点。

至少要知道：

- `vector` 扩容常导致整体失效
- `erase` 常使删除点及其后的迭代器失效
- 链表类容器通常在局部删除上更稳定，但缓存局部性差

> **思考题：遍历 vector 时 erase 元素，怎么写才是正确的？**
>
> ```cpp
> // 错误：erase 后 it 失效
> for (auto it = v.begin(); it != v.end(); ++it) {
>     if (*it % 2 == 0) v.erase(it);  // it 失效！
> }
>
> // 正确：利用 erase 返回下一个有效迭代器
> for (auto it = v.begin(); it != v.end(); ) {
>     if (*it % 2 == 0)
>         it = v.erase(it);  // erase 返回被删元素之后的有效迭代器
>     else
>         ++it;
> }
> ```
>
> C++20 引入了 `std::erase_if(v, pred)`，封装了这个常见模式。

---

## 七、异常安全和工程质量

### 1. 异常安全分级

比较常见的说法：

- 不抛异常保证（nothrow guarantee）
- 基本保证（basic guarantee）：操作失败不会泄漏资源，对象仍可析构
- 强保证（strong guarantee）：操作要么成功完成，要么失败但对象状态像没发生过一样

### 2. copy-and-swap

这个技巧经常和强异常安全、赋值运算符一起问：

- 先拷贝出临时对象
- 再交换资源
- 让失败尽量发生在修改原对象之前

> **思考题：copy-and-swap 是如何实现强异常安全的？有什么代价？**
>
> 原理：
> 1. 拷贝构造一个临时对象（如果这一步抛异常，原对象完好无损）
> 2. 用 `swap` 交换临时对象和当前对象的内部数据（swap 通常不抛异常）
> 3. 临时对象离开作用域时析构（带着原来的数据）
>
> 代价：
> - 每次赋值操作都会做一次完整拷贝，即使可以使用移动优化
> - 所以有时会额外提供移动赋值运算符来提高效率
> - copy-and-swap 更适合作为 " 统一的赋值运算符 " 实现方式，接受参数按值传递，然后用 swap

### 3. `noexcept`

- 它影响异常语义，也影响某些库优化机会
- 例如容器在搬迁元素时，常更愿意选 `noexcept` move

> **思考题：什么时候应该标记 `noexcept`？什么时候不应该？**
>
> 应该标 `noexcept` 的情况：
> - 移动构造/移动赋值（让容器在扩容时优先用移动）
> - `swap` 函数
> - 析构函数（析构函数默认是 noexcept(true)，尽量不要破坏这一点）
> - 简单的 getter/setter（如 `int size() const noexcept`）
>
> 不应该标 `noexcept` 的情况：
> - 函数内部调用了可能抛异常的函数（如果你标了 noexcept 但实际抛了，程序会直接 `std::terminate`）
> - 函数的功能将来可能需要扩展，且扩展版可能需要抛异常
> - 接口设计还不稳定时，一旦标了 noexcept 就变成了 ABI 承诺，后续去掉 noexcept 可能破坏调用方

---

## 八、C++ 并发、原子和内存模型

如果你说你熟悉 C++，这块非常值得准备。并发是区分 " 用过 C++" 和 " 理解 C++" 的关键分水岭。

### 1. `thread`、`mutex`、条件变量

#### 1.1 `std::thread` 基础

- `thread` 构造时启动线程，传入可调用对象和参数
- `join()` 阻塞等待线程结束，`detach()` 分离线程让它在后台运行
- 线程对象析构前必须调用 `join()` 或 `detach()`，否则 `std::terminate`
- 参数按值传递到线程（即使是引用类型也会拷贝），要传引用用 `std::ref`

> **思考题：为什么线程函数参数默认按值传递？这会导致什么陷阱？**
>
> ```cpp
> void update(std::string& s) { s = "done"; }
> std::string s = "hello";
> std::thread t(update, s);  // s 被拷贝！即使 update 接受 string&
> ```
>
> 线程是异步的——被传入的参数可能在调用方的栈帧已销毁后才被使用。默认按值拷贝避免了悬垂引用。如果确实要传引用，用 `std::ref(s)`，它生成 `std::reference_wrapper`，线程会从中解出真实引用。但此时程序员自己负责确保被引用的对象在线程执行期间存活。
>
> `detach()` 的另一个陷阱：分离线程可能访问已销毁的局部变量。不要 detach 使用了局部变量引用的线程。

#### 1.2 Mutex 家族

| mutex 类型 | 特点 | 适用场景 |
|---|---|---|
| `std::mutex` | 基本互斥锁，不可递归 | 大多数场景 |
| `std::recursive_mutex` | 同一线程可重复加锁 | 递归调用、多个函数各自加锁时 |
| `std::timed_mutex` | 支持 `try_lock_for/until` | 需要超时的场景 |
| `std::shared_mutex` (C++17) | 共享读、独占写 | 读多写少的场景 |
| `std::recursive_shared_mutex` | 共享 + 递归 | | |

> **思考题：为什么应该尽量避免 `recursive_mutex`？**
>
> `recursive_mutex` 虽然方便，但它掩盖了设计问题：
> - 它说明你的锁粒度或所有权边界不够清晰——你在已持有锁的情况下又进入了需要同一锁的代码
> - 需要匹配次数相同的 unlock，容易出错
> - 可能意味着你的类不变量在持有锁的间隙被破坏
>
> 更好的做法是重构代码，明确哪些函数是 " 调用时已持锁 " 的私有版本，哪些是 " 对外公开、内部加锁 " 的公有版本。如果确实需要递归锁，至少确保你清楚为什么。

#### 1.3 RAII 锁管理器

- `std::lock_guard`：最简单的 RAII 锁，构造加锁，析构解锁，不可手动解锁
- `std::unique_lock`：可延迟加锁、提前解锁、可移动，适合配合条件变量
- `std::scoped_lock` (C++17)：可同时锁定多个 mutex，避免死锁
- `std::shared_lock` (C++17)：配合 `shared_mutex` 的读锁

```cpp
// scoped_lock 避免死锁：同时锁两个 mutex
std::mutex m1, m2;
{
    std::scoped_lock lock(m1, m2);  // 内部用 std::lock 算法避免死锁
    // 同时持有 m1 和 m2
}
```

> **思考题：`lock_guard`、`unique_lock`、`scoped_lock` 分别在什么场景使用？**
>
> - `lock_guard`：简单的临界区保护，不需要条件变量。代码最短、开销最小
> - `unique_lock`：需要配合 `condition_variable`（cv 要求 unique_lock）、需要手动提前 unlock、需要延迟加锁
> - `scoped_lock`：需要同时锁多个 mutex，或者 C++17 之后单纯想写最安全的代码（scoped_lock 是 lock_guard 的 C++17 替代升级版——对单个锁两者等价，但 scoped_lock 也能处理多个锁）

#### 1.4 条件变量

```cpp
std::mutex m;
std::condition_variable cv;
std::queue<int> q;

// 消费者
void consumer() {
    while (true) {
        std::unique_lock<std::mutex> lock(m);
        cv.wait(lock, [] { return !q.empty(); });  // 带谓词，自动处理虚假唤醒
        int val = q.front(); q.pop();
        lock.unlock();  // 尽早释放锁
        process(val);
    }
}

// 生产者
void producer(int val) {
    {
        std::lock_guard<std::mutex> lock(m);
        q.push(val);
    }
    cv.notify_one();  // 通知一个等待者
}
```

> **思考题：条件变量为什么要用 `unique_lock` 而不能用 `lock_guard`？`
>
> `condition_variable::wait()` 内部需要做三件事：
> 1. 释放锁（让生产者能获得锁）
> 2. 进入等待状态（阻塞）
> 3. 被唤醒后重新获取锁（才能安全检查条件）
>
> `lock_guard` 不支持手动 unlock/lock，无法完成步骤 1 和 3。`unique_lock` 支持 `lock()` / `unlock()` / `try_lock()`，所以条件变量内部可以用它来灵活控制锁状态。

> **思考题：什么是虚假唤醒（spurious wakeup）？怎么处理？**
>
> 虚假唤醒是指：线程在 `wait()` 中被唤醒，但没有收到对应的 `notify`。这是操作系统线程调度层面的现象（POSIX 标准允许），可能由信号、内核调度等原因引起，并非程序错误。
>
> 处理方式：**始终用带谓词的 wait**：
> ```cpp
> cv.wait(lock, [] { return ready; });  // 等价于 while (!ready) cv.wait(lock);
> ```
> 谓词在唤醒后（重新获得锁后）被检查，如果条件仍不满足，自动重新等待。手动用 `while` 循环也是正确的，但带谓词的 `wait` 更简洁。

#### 1.5 其他同步原语（C++20）

- `std::latch`：一次性计数器，所有线程到达后一起继续
- `std::barrier`：可复用的同步点，所有线程到达后执行回调再继续
- `std::counting_semaphore` / `std::binary_semaphore`：轻量级信号量
- `std::jthread`：自动 join 的线程，支持中断（stop_token）

> **思考题：`barrier` 和 `latch` 的区别是什么？**
>
> - `latch` 是一次性的：计数器从 N 倒数到 0，所有等待的线程被释放。不能再重置。适合 " 等所有初始化任务完成 " 这种一次性场景。
> - `barrier` 是循环使用的：N 个线程到达屏障后自动重置，可以用于循环迭代中的同步。`barrier` 还可以在到达时执行一个完成回调。
>
> 典型应用：并行迭代 `for (int i = 0; i < 10; ++i) { parallel_work(); barrier.arrive_and_wait(); }`

### 2. `std::atomic`

#### 2.1 基本操作

- `load()` / `store()`：读写
- `exchange()`：原子地读出旧值并写入新值
- `fetch_add` / `fetch_sub`：原子加减，返回旧值
- `compare_exchange_weak` / `compare_exchange_strong`：CAS 操作

```cpp
std::atomic<int> counter{0};

// 用法 1：简单计数（默认 memory_order_seq_cst）
counter.fetch_add(1);

// 用法 2：CAS 实现无锁更新
int expected = counter.load();
while (!counter.compare_exchange_weak(expected, expected + 1)) {
    // expected 被自动更新为当前值，循环重试
}
```

> **思考题：`compare_exchange_weak` 和 `compare_exchange_strong` 有什么区别？什么时候用哪个？**
>
> - `compare_exchange_strong`：只有真正不匹配时才失败。语义简单，但某些平台（ARM）需要更多指令来实现
> - `compare_exchange_weak`：即使在比较结果相等的情况下也可能 " 虚假失败 "（spurious failure），但性能更好（在 x86 上两者通常一样，但在 ARM 上用 LL/SC 指令时 weak 更高效）
>
> **使用原则**：如果在 CAS 失败后需要用循环重试，用 `weak` 版本——虚假失败只是多循环一次，不影响正确性。如果后续操作代价很大、不能容忍虚假失败，或者你的代码里失败分支不是循环重试，用 `strong`。

#### 2.2 `atomic` 与 `mutex` 的边界

- `atomic` 适合：单个变量的原子读/写/更新（计数器、标志位、状态机）
- `mutex` 适合：保护多变量或复杂数据结构的不变关系
- " 用 atomic 代替 mutex" 只有在临界区仅是单个原子操作时成立
- `atomic` 无法替代保护 " 多个变量必须同时一致更新 " 的场景

> **思考题：`atomic<int> x = 0; x = x + 1;` 这行代码是原子的吗？**
>
> **不是。** `x = x + 1` 包含三个操作：load x -> 加 1 -> store x。它等价于：
> ```cpp
> int tmp = x.load();  // 原子
> tmp = tmp + 1;       // 非原子（不涉及 x）
> x.store(tmp);        // 原子
> ```
> 整个表达式不是原子的——另一个线程可能在这三步之间修改 x。如果需要原子加法，用 `x.fetch_add(1)`。

### 3. 内存序（Memory Order）

这是并发面试中最硬核的部分。以下是每个 memory_order 的精确理解：

#### 3.1 `memory_order_relaxed`

- 只保证操作的**原子性**（读/写不会被切碎），不保证任何顺序
- 编译器可能重排，CPU 也可能重排
- 适用场景：单调递增计数器，无同步需求的统计

```cpp
// 只关心最终总数，不关心和别的变量之间的顺序
std::atomic<int> total_hits{0};
void record_hit() { total_hits.fetch_add(1, std::memory_order_relaxed); }
```

> **思考题：既然 relaxed 不保证顺序，为什么还需要它？**
>
> 在只需要原子性、不需要跨变量同步的场景（如全局计数器、统计值），relaxed 比 seq_cst 快得多。特别是在弱排序架构（ARM、PowerPC）上，seq_cst 需要额外的内存屏障指令（如 `dmb`），而 relaxed 不需要。在高频操作中（如每个 packet 更新一次计数器），使用 relaxed 能显著降低开销。

#### 3.2 `memory_order_release` 和 `memory_order_acquire`

这是理解并发同步的核心：

- **release**（发布）：当前线程之前的所有内存写操作，对之后做 acquire 的线程可见。用于 " 发布 " 数据
- **acquire**（获取）：当前线程之后的所有内存操作不会被重排到 acquire 之前。用于 " 获取 " 数据
- release + acquire 配对形成**同步关系**（synchronizes-with）

```cpp
// 线程 A：
data = 42;                                // （1）写入数据
ready.store(true, memory_order_release);  // （2）发布 flag

// 线程 B：
while (!ready.load(memory_order_acquire));  // （3）获取 flag（自旋等待）
assert(data == 42);                         // （4）一定能看到 data = 42
```

关键理解：当线程 B 看到 ready == true 时，线程 A 在 release 之前的所有写操作（包括 data = 42）对线程 B 都可见。

> **思考题：如果把上面的 release/acquire 都换成 relaxed，assert 还会成立吗？**
>
> 不保证成立。`relaxed` 不建立跨变量的顺序关系：
> - CPU 可能重排：线程 A 中，ready 的写可能在 data 的写之前对其他核心可见
> - 编译器也可能重排：编译后的指令顺序不保证和源代码一致
> - 线程 B 即使看到 ready == true，也可能看到旧的 data 值（缓存未同步）
>
> 这就是为什么 acquire/release 不是可选的——在没有它们的情况下，多线程的正确性没有保证。

#### 3.3 `memory_order_acq_rel`

- 兼具 acquire 和 release 语义
- 常用于 RMW（read-modify-write）操作，如 `fetch_add`、`compare_exchange`
- 典型场景：构建锁的基础原语、引用计数

```cpp
// 引用计数：fetch_add 的修改对后续 release 可见，fetch_sub 在释放前可见之前的修改
ref_count.fetch_add(1, memory_order_relaxed);   // 增加引用，只需要原子性
ref_count.fetch_sub(1, memory_order_acq_rel);   // 释放前需要 acquire 来看到所有修改
```

#### 3.4 `memory_order_seq_cst`

- 默认的 memory_order，最严格
- 所有 `seq_cst` 操作之间存在全局统一的顺序（single total order）
- 代价最高，但在 x86 上由于硬件提供强内存模型，`seq_cst` 和 `acq_rel` 的指令开销通常一样

> **思考题：seq_cst 和 acquire-release 的根本区别是什么？**
>
> acquire-release 只在线程间建立**成对的**同步关系（pairwise synchronizes-with）。而 seq_cst 额外保证**所有** seq_cst 操作在所有线程看来有同一个全局顺序（single total order）。
>
> 一个体现区别的场景（独立写入序列号，简称 IRIW）：
> ```cpp
> // 线程 A: x.store(1, mo);  线程 B: y.store(1, mo);
> // 线程 C: r1 = x.load(mo); r2 = y.load(mo);
> // 线程 D: r3 = y.load(mo); r4 = x.load(mo);
> ```
> 用 `mo = acquire/release`：C 可能看到 (r1=1, r2=0) 同时 D 看到 (r3=1, r4=0)——即 " 写传播到不同线程的顺序不一致 "。用 `mo = seq_cst`：这种情况不可能发生。
>
> 实践中，如果你只在**两个线程之间**做生产者 - 消费者同步，acquire/release 足够且更快。如果你有多个线程需要看到**一致的事件顺序**，需要 seq_cst。

### 4. `happens-before`

这是理解并发正确性的核心词。

- 如果 A happens-before B，那么 B 一定观察到 A 的效果
- 它是**传递**的：A → B，B → C 意味着 A → C
- 建立 happens-before 的主要途径：
  1. **程序顺序**（sequenced-before）：同一线程内 A 在 B 之前执行
  2. **同步**（synchronizes-with）：
     - mutex unlock → 后续 lock
     - release store → acquire load（看到值）
     - `notify_one` → `wait` 被唤醒
     - `thread::join` 后，被等待线程的所有操作 happens-before join 返回之后

```cpp
// 线程 A                  线程 B
data = 42;                // (A1)
lock();                   // (A2)
q.push(data);             // (A3)
unlock();                 // (A4)
                          lock();                // (B1) 获取到 A4 释放的锁
                          auto d = q.front();    // (B2) 能看到 A3 的结果
                          unlock();              // (B3)
```

happens-before 链：A1→A2→A3→A4→B1→B2→B3。所以 B3 能看到 A1 的结果。

> **思考题：所有并发正确性最终都能归结于 happens-before 关系吗？**
>
> 是的。C++ 内存模型为多线程程序定义了 " 数据竞赛 "：两个操作访问同一内存、至少有一个是写、且两者之间没有 happens-before 关系 → 这是数据竞赛，未定义行为。反过来说，要证明程序的正确性，本质上就是证明 " 对某个共享变量的所有写访问，都和读访问之间有确定的 happens-before 关系 "。理解了 happens-before，就理解了 C++ 并发正确性的基石。

### 5. 伪共享（False Sharing）

- 不同线程虽然改的是不同变量
- 但如果这些变量落在同一 cache line（通常 64 字节），上层看似无竞争，底层仍会产生缓存抖动

```cpp
// 问题代码：
struct BadCounter {
    alignas(64) int counter_a;  // 如果不用 alignas，a 和 b 可能同一 cache line
    int counter_b;  // 两个线程分别改 a 和 b，但实际上互相无效对方的 cache line
};

// 修复方案：
struct alignas(64) PaddedCounter {
    int counter_a;
    char padding[60];  // 确保下一个 counter 在另一 cache line
};
// 或直接用硬件提示（C++17）：
struct GoodCounter {
    alignas(std::hardware_destructive_interference_size) int counter_a;
    alignas(std::hardware_destructive_interference_size) int counter_b;
};
```

> **思考题：为什么伪共享会影响性能？在多核 CPU 上 cache coherence 协议是怎么导致这个问题的？**
>
> CPU 的缓存一致性协议（如 MESI）以 cache line 为粒度进行一致性维护。
> 当核心 1 写 `counter_a` 时，它需要独占（Modified/Exclusive）拥有包含 `counter_a` 的整条 cache line。如果核心 2 的缓存中也持有这行（因为 `counter_b` 也在上面），核心 2 的拷贝会被置为 Invalid。
> 当核心 2 写 `counter_b` 时，它需要重新获取该行的独占权，导致核心 1 的拷贝失效。
> 两个核心交替地把 cache line 抢来抢去，即使它们访问的是完全不同的变量。这叫做 " 缓存行乒乓 "（cache line ping-pong），每次跨核心的缓存行传输有几十到几百个周期的延迟。在高竞争场景下，伪共享可能让程序慢 10 倍以上。
>
> 解决方案的核心思想：把不同线程高频写入的变量放到不同的 cache line 上（通过 padding 或 `alignas(hardware_destructive_interference_size)`）。

### 6. `std::async`、`std::future`、`std::promise`

- `std::async`：启动异步任务，返回 `future`
- `std::future`：获取异步任务的结果，`get()` 会阻塞等待
- `std::promise`：手动设置结果，通过 `get_future()` 获得关联的 future
- `std::packaged_task`：包装可调用对象，可以获取它的 future

```cpp
// async + future
auto future = std::async(std::launch::async, [] {
    return expensive_computation();
});
// 做其他事情...
auto result = future.get();  // 阻塞直到完成

// promise + future：更灵活的手动控制
std::promise<int> p;
std::future<int> f = p.get_future();
std::thread worker([&p] {
    p.set_value(42);  // 设置结果
});
int result = f.get();  // 阻塞等待
worker.join();
```

> **思考题：`std::async` 的 `std::launch::async` 和 `std::launch::deferred` 有什么区别？**
>
> - `std::launch::async`：保证在新线程中异步执行
> - `std::launch::deferred`：推迟执行——直到 `future.get()` 或 `wait()` 被调用时，在当前线程中同步执行（惰性求值）
> - 默认（两者都传）：由实现决定是异步还是延迟（通常就是两者都指定的语义，但实际选取哪个是实现定义的）
>
> 要注意：如果 deferred 任务从未被 get/wait，它根本不会执行——析构时会丢弃任务。

### 7. 无锁编程基础

- 核心思想：用 `atomic` + CAS 实现数据结构，避免 mutex 的开销
- 优势：无死锁、通常延迟更低
- 挑战：实现正确极其困难，ABA 问题、内存回收（hazard pointer / RCU）

> **思考题：什么是 ABA 问题？如何解决？**
>
> ABA 问题：线程 A 读到共享变量值为 A，准备 CAS 把 A 改成 C。但在 A 做 CAS 之前，线程 B 把 A 改成 B，又改回 A。线程 A 的 CAS 仍然成功（值还是 A），但它不知道中间发生过变化——如果这个值代表指针，原来的对象可能已被释放，新 A 是另一个对象，CAS 成功却指向了垃圾内存。
>
> 解决思路：
> 1. **带版本号的指针**（tagged pointer）：在高位放版本号，每次修改递增。CAS 同时比较指针和版本号
> 2. **hazard pointer**：读线程标记自己正在使用某指针，写线程延迟回收直到没有人标记
> 3. **RCU**（Read-Copy-Update）：读者无锁，写者做拷贝 - 修改 - 替换，旧版本延迟释放
> 4. 直接使用 `std::atomic<std::shared_ptr<T>>`（C++20），标准库帮你处理了并发安全和内存回收

### 8. C++17 并行算法

```cpp
#include <execution>

std::vector<int> v = {3, 1, 4, 1, 5, 9, 2, 6, 5, 3};

// 串行（默认）
std::sort(std::execution::seq, v.begin(), v.end());

// 并行
std::sort(std::execution::par, v.begin(), v.end());

// 并行 + 向量化（SIMD）
std::sort(std::execution::par_unseq, v.begin(), v.end());
```

注意：并行算法的迭代器要求较严格（至少是前向迭代器），异常处理和线程安全也需要关注。

### 9. 实验代码

以下每个实验都是一个独立可编译运行的 `.cpp` 文件。编译建议：

```bash
# 基础编译
g++ -std=c++17 -pthread -O2 exp_xxx.cpp -o exp_xxx

# 建议同时用 ThreadSanitizer 检测数据竞争
g++ -std=c++17 -pthread -O2 -fsanitize=thread exp_xxx.cpp -o exp_xxx_tsan

# 部分实验需要 C++20 (latch/barrier)
g++ -std=c++20 -pthread -O2 exp_xxx.cpp -o exp_xxx
```

---

#### 实验 1：Mutex 家族与 RAII 锁管理器

演示 `lock_guard`、`unique_lock`、`scoped_lock`、`shared_mutex` 的使用场景。

```cpp
// exp_mutex_family.cpp
// 编译: g++ -std=c++17 -pthread -O2 exp_mutex_family.cpp -o exp_mutex_family
#include <iostream>
#include <thread>
#include <mutex>
#include <shared_mutex>
#include <vector>
#include <chrono>

// ========== 场景 1：lock_guard — 最简单临界区保护 ==========
void demo_lock_guard() {
    std::mutex m;
    int counter = 0;
    constexpr int N = 100000;

    auto worker = [&] {
        for (int i = 0; i < N; ++i) {
            std::lock_guard<std::mutex> lg(m);  // 构造加锁，析构解锁
            ++counter;
        }
    };

    std::thread t1(worker), t2(worker);
    t1.join(); t2.join();
    std::cout << "[lock_guard] counter = " << counter
              << " (expected " << 2 * N << ")\n";
}

// ========== 场景 2：unique_lock — 配合条件变量 / 手动解锁 ==========
void demo_unique_lock() {
    std::mutex m;
    bool ready = false;
    int data = 0;

    std::thread producer([&] {
        {
            std::unique_lock<std::mutex> ul(m);
            data = 42;
            ready = true;
            // unique_lock 可以在此处手动 unlock，也可以让析构处理
        }
        // 锁已释放，消费者可以获取
    });

    std::thread consumer([&] {
        // 轮询等待 ready（生产环境请用 condition_variable，见实验 2）
        while (true) {
            std::unique_lock<std::mutex> ul(m);
            if (ready) {
                std::cout << "[unique_lock] data = " << data << " (should be 42)\n";
                break;
            }
            ul.unlock();  // 手动释放让 producer 能获取锁
            std::this_thread::yield();
        }
    });

    producer.join(); consumer.join();
}

// ========== 场景 3：scoped_lock — 同时锁多个 mutex，避免死锁 ==========
void demo_scoped_lock() {
    std::mutex m1, m2;
    int a = 0, b = 0;

    // 如果分别 lock(m1); lock(m2); 可能死锁
    // scoped_lock 内部用 std::lock 算法，保证无死锁
    auto transfer_1_to_2 = [&] {
        for (int i = 0; i < 10000; ++i) {
            std::scoped_lock lock(m1, m2);  // C++17: 同时锁两个
            ++a; --b;
        }
    };
    auto transfer_2_to_1 = [&] {
        for (int i = 0; i < 10000; ++i) {
            std::scoped_lock lock(m1, m2);
            --a; ++b;
        }
    };

    std::thread t1(transfer_1_to_2), t2(transfer_2_to_1);
    t1.join(); t2.join();
    std::cout << "[scoped_lock] a=" << a << " b=" << b
              << " (sum should be 0)\n";
}

// ========== 场景 4：shared_mutex — 读多写少 ==========
void demo_shared_mutex() {
    std::shared_mutex sm;
    int data = 0;
    std::atomic<int> read_count{0};

    // 写线程：独占锁
    auto writer = [&] {
        for (int i = 0; i < 10; ++i) {
            std::unique_lock<std::shared_mutex> ul(sm);  // 独占写锁
            ++data;
            std::this_thread::sleep_for(std::chrono::milliseconds(5));
        }
    };

    // 读线程：共享锁
    auto reader = [&](int id) {
        for (int i = 0; i < 50; ++i) {
            std::shared_lock<std::shared_mutex> sl(sm);  // 共享读锁
            volatile int _ = data;  // 模拟读
            read_count.fetch_add(1, std::memory_order_relaxed);
            std::this_thread::sleep_for(std::chrono::milliseconds(1));
        }
    };

    std::thread tw(writer);
    std::vector<std::thread> readers;
    for (int i = 0; i < 4; ++i)
        readers.emplace_back(reader, i);

    tw.join();
    for (auto& t : readers) t.join();
    std::cout << "[shared_mutex] final data=" << data
              << " total reads=" << read_count.load() << "\n";
}

int main() {
    demo_lock_guard();********
    demo_unique_lock();
    demo_scoped_lock();
    demo_shared_mutex();
    return 0;
}
```

**预期输出理解**：每个场景的结果都应和 `expected` 一致。如果不用锁（注释掉 lock_guard 那行），counter 大概率不会是 2N——这就是数据竞争。

---

#### 实验 2：条件变量 — 生产者 / 消费者

演示 `condition_variable`、虚假唤醒处理、`notify_one` vs `notify_all`。

```cpp
// exp_condition_variable.cpp
// 编译: g++ -std=c++17 -pthread -O2 exp_condition_variable.cpp -o exp_condition_variable
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>
#include <chrono>
#include <vector>

int main() {
    std::mutex m;
    std::condition_variable cv;
    std::queue<int> q;
    bool done = false;  // 生产者结束标志

    constexpr int TOTAL = 20;

    // ========== 生产者：每 50ms 生产一个 ==========
    std::thread producer([&] {
        for (int i = 1; i <= TOTAL; ++i) {
            std::this_thread::sleep_for(std::chrono::milliseconds(50));
            {
                std::lock_guard<std::mutex> lg(m);
                q.push(i);
                std::cout << "[producer] produced " << i << "\n";
            }
            cv.notify_one();  // 通知一个等待的消费者
        }
        {
            std::lock_guard<std::mutex> lg(m);
            done = true;
        }
        cv.notify_all();  // 通知所有消费者（done 是全局性的）
    });

    // ========== 消费者：等待并处理 ==========
    auto consumer = [&](int id) {
        while (true) {
            std::unique_lock<std::mutex> ul(m);

            // ★ 带谓词的 wait：自动处理虚假唤醒
            // 等价于 while (!pred()) { cv.wait(ul); }
            cv.wait(ul, [&] { return !q.empty() || done; });

            if (!q.empty()) {
                int val = q.front(); q.pop();
                ul.unlock();  // 尽早释放锁，让其他线程可以获取
                std::cout << "  [consumer " << id << "] consumed " << val << "\n";
            } else if (done) {
                break;  // done && queue empty → 退出
            }
        }
    };

    std::vector<std::thread> consumers;
    for (int i = 0; i < 3; ++i)
        consumers.emplace_back(consumer, i);

    producer.join();
    for (auto& t : consumers) t.join();
    std::cout << "All done.\n";
    return 0;
}
```

**动手实验**：
1. 把 `cv.wait(ul, [&]{...})` 改成 `cv.wait(ul)` ——会发生什么？（生产者结束后消费者可能永远阻塞）
2. 把 `cv.notify_one()` 改成注释掉——消费者是否还能取到数据？（不能，会一直等待）
3. 把 `producer` 的 `lg` 局部作用域 `{}` 去掉——锁持有太久，消费者等锁时数据已经在队列里了

---

#### 实验 3：atomic — `x = x + 1` 不是原子的！

这个实验最直观地展示 "atomic 操作" 和 "看起来像一行但不是原子的操作" 的区别。

```cpp
// exp_atomic_basic.cpp
// 编译: g++ -std=c++17 -pthread -O2 exp_atomic_basic.cpp -o exp_atomic_basic
// ThreadSanitizer: g++ -std=c++17 -pthread -O2 -fsanitize=thread exp_atomic_basic.cpp -o exp_atomic_tsan
#include <iostream>
#include <thread>
#include <atomic>
#include <vector>
#include <chrono>

int main() {
    constexpr int N_THREADS = 8;
    constexpr int N_INCREMENTS = 100000;

    // ====== 版本 1：非原子 int，用 x = x + 1 ======
    {
        int non_atomic = 0;
        auto worker = [&] {
            for (int i = 0; i < N_INCREMENTS; ++i) {
                // ★ 这一行包含 3 步：load -> add -> store，不是原子的
                non_atomic = non_atomic + 1;
            }
        };

        std::vector<std::thread> threads;
        for (int i = 0; i < N_THREADS; ++i)
            threads.emplace_back(worker);
        for (auto& t : threads) t.join();

        std::cout << "[non-atomic] counter = " << non_atomic
                  << " (expected " << N_THREADS * N_INCREMENTS << ")"
                  << " lost: " << (N_THREADS * N_INCREMENTS - non_atomic) << "\n";
    }

    // ====== 版本 2：atomic int，用 fetch_add（原子） ======
    {
        std::atomic<int> atomic_counter{0};
        auto worker = [&] {
            for (int i = 0; i < N_INCREMENTS; ++i) {
                atomic_counter.fetch_add(1, std::memory_order_relaxed);
            }
        };

        std::vector<std::thread> threads;
        for (int i = 0; i < N_THREADS; ++i)
            threads.emplace_back(worker);
        for (auto& t : threads) t.join();

        std::cout << "[atomic fetch_add] counter = " << atomic_counter.load()
                  << " (expected " << N_THREADS * N_INCREMENTS << ")\n";
    }

    // ====== 版本 3：atomic int，但用 x = x + 1（还是非原子的整体操作！） ======
    {
        std::atomic<int> a{0};
        auto worker = [&] {
            for (int i = 0; i < N_INCREMENTS; ++i) {
                // ★ 即使变量是 atomic<int>，x = x + 1 也包含 load→add→store 三步
                // 三步之间另一个线程可能插进来修改
                a = a + 1;  // 等价于 a.store(a.load() + 1); ——不是原子的整体
            }
        };

        std::vector<std::thread> threads;
        for (int i = 0; i < N_THREADS; ++i)
            threads.emplace_back(worker);
        for (auto& t : threads) t.join();

        std::cout << "[atomic but x=x+1] counter = " << a.load()
                  << " (expected " << N_THREADS * N_INCREMENTS << ")"
                  << " lost: " << (N_THREADS * N_INCREMENTS - a.load())
                  << " — even atomic<int> doesn't make x=x+1 atomic!\n";
    }

    // ====== 版本 4：CAS 循环实现无锁加法（等价于 fetch_add） ======
    {
        std::atomic<int> cas_counter{0};
        auto worker = [&] {
            for (int i = 0; i < N_INCREMENTS; ++i) {
                int expected = cas_counter.load(std::memory_order_relaxed);
                // ★ compare_exchange_weak 在 x86 上可能虚假失败，但循环保证正确
                while (!cas_counter.compare_exchange_weak(
                    expected, expected + 1,
                    std::memory_order_relaxed,
                    std::memory_order_relaxed))
                {
                    // expected 被自动更新为当前值，继续重试
                }
            }
        };

        std::vector<std::thread> threads;
        for (int i = 0; i < N_THREADS; ++i)
            threads.emplace_back(worker);
        for (auto& t : threads) t.join();

        std::cout << "[CAS loop]     counter = " << cas_counter.load()
                  << " (expected " << N_THREADS * N_INCREMENTS << ")\n";
    }

    return 0;
}
```

**预期输出**（典型）：
```
[non-atomic] counter = 234567 (expected 800000) lost: 565433
[atomic fetch_add] counter = 800000 (expected 800000)
[atomic but x=x+1] counter = 312456 (expected 800000) lost: 487544
[CAS loop]     counter = 800000 (expected 800000)
```

关键理解：**`atomic` 保证单个操作的原子性（load/store/fetch_add），但 `x = x + 1` 是两个原子操作夹一个非原子加法，整体不是原子的。**

---

#### 实验 4：内存序 — relaxed 的 "不可靠" 标志位

这是理解内存序最重要的实验。我们用 `relaxed` 和 `release/acquire` 对比同一个场景。

```cpp
// exp_memory_order_flag.cpp
// 编译: g++ -std=c++17 -pthread -O2 exp_memory_order_flag.cpp -o exp_memory_order_flag
// ★ 在 x86 上编译器重排主要体现在 O2 下。ARM 上硬件也会重排。
// ★ 如果你有 ARM 机器（树莓派/苹果 M 系列），跑这个实验会更明显。
#include <iostream>
#include <thread>
#include <atomic>
#include <cassert>

// 共享数据
int data = 0;                              // 非原子数据
std::atomic<bool> ready{false};           // 标志位

constexpr int ITERATIONS = 1000000;

// ====== 版本 1：relaxed — 无同步保证 ======
void test_relaxed() {
    int violations = 0;
    for (int iter = 0; iter < ITERATIONS; ++iter) {
        data = 0;
        ready.store(false, std::memory_order_relaxed);

        std::thread writer([&] {
            data = 42;                                        // ① 写数据
            ready.store(true, std::memory_order_relaxed);     // ② relaxed 标记
            // 用 relaxed 时，编译器/CPU 可以重排 ① 和 ②！
            // 其他线程可能先看到 ready=true，再看到 data=42
        });

        std::thread reader([&] {
            if (ready.load(std::memory_order_relaxed)) {      // ③ relaxed 读
                if (data != 42) {                             // ④
                    ++violations;  // ★ 重排发生了！
                }
            }
        });

        writer.join(); reader.join();
    }
    std::cout << "[relaxed] violations: " << violations
              << " / " << ITERATIONS << " iterations\n";
    if (violations > 0)
        std::cout << "  -> 编译器或 CPU 重排了 data=42 和 ready=true 的顺序！\n";
}

// ====== 版本 2：release/acquire — 正确的同步 ======
void test_release_acquire() {
    int violations = 0;
    for (int iter = 0; iter < ITERATIONS; ++iter) {
        data = 0;
        ready.store(false, std::memory_order_relaxed);

        std::thread writer([&] {
            data = 42;                                         // ① 写数据
            ready.store(true, std::memory_order_release);      // ② release 发布
            // release 保证：② 之前的所有写（包括 data=42）在 ② 对 acquire 可见时也可见
        });

        std::thread reader([&] {
            if (ready.load(std::memory_order_acquire)) {       // ③ acquire 获取
                // acquire 保证：③ 之后的代码一定能看到 release 之前的写
                if (data != 42) {
                    ++violations;  // ★ 如果有 violation，说明同步失败
                }
            }
        });

        writer.join(); reader.join();
    }
    std::cout << "[release/acquire] violations: " << violations
              << " / " << ITERATIONS << " iterations\n";
}

int main() {
    test_relaxed();
    test_release_acquire();
    return 0;
}
```

**预期行为**：
- `relaxed`：在 x86 上由于硬件 TSO（Total Store Order）模型，store-store 重排很少见，但 **编译器重排** 在高优化等级下可能发生。在 ARM 上 violations 会更明显。
- `release/acquire`：violations 必须是 0——这是标准的同步语义保证。

> **CPU 架构影响**：x86 是强内存模型（TSO），很多 relaxed 能 " 碰巧正常工作 "。ARM/PowerPC 是弱内存模型，relaxed 的违规会更频繁出现。这也是为什么写跨平台代码必须用正确的 memory_order——在 x86 上 " 测试通过 " 不代表在 ARM 上也正确。

---

#### 实验 5：内存序 — seq_cst 的全局顺序 vs acquire/release 的局部同步

IRIW（Independent Readers, Independent Writers）模式展示了 seq_cst 和 acquire/release 的关键区别：seq_cst 保证所有线程看到一致的全局操作顺序。

```cpp
// exp_memory_order_iriw.cpp
// 编译: g++ -std=c++17 -pthread -O2 exp_memory_order_iriw.cpp -o exp_memory_order_iriw
// ★ 这个实验在 x86 上可能两者都不违规（x86 store-store 有序，load 也有较强保证）
// ★ 在 ARM 上差异更明显
#include <iostream>
#include <thread>
#include <atomic>
#include <vector>

template<typename MO>
void test_iriw(const char* name, MO mo_write, MO mo_read) {
    // IRIW: Independent Readers, Independent Writers
    // Thread 1: x = 1        Thread 3: r1 = x; r2 = y;
    // Thread 2: y = 1        Thread 4: r3 = y; r4 = x;
    // 问题：能否出现 r1=1,r2=0 且 r3=1,r4=0？(两个读者看到的写顺序不一致)

    std::atomic<int> x{0}, y{0};
    constexpr int ITERATIONS = 100000;
    int violation_count = 0;

    for (int iter = 0; iter < ITERATIONS; ++iter) {
        x.store(0); y.store(0);
        std::atomic<bool> ready1{false}, ready2{false};
        int r1_val = 0, r2_val = 0, r3_val = 0, r4_val = 0;

        std::thread t1([&] { x.store(1, mo_write); });
        std::thread t2([&] { y.store(1, mo_write); });
        std::thread t3([&] {
            r1_val = x.load(mo_read);
            r2_val = y.load(mo_read);
        });
        std::thread t4([&] {
            r3_val = y.load(mo_read);
            r4_val = x.load(mo_read);
        });

        t1.join(); t2.join(); t3.join(); t4.join();

        if (r1_val == 1 && r2_val == 0 && r3_val == 1 && r4_val == 0) {
            ++violation_count;
        }
    }

    std::cout << "[" << name << "] IRIW violations: " << violation_count
              << " / " << ITERATIONS << "\n";

    if (violation_count > 0) {
        std::cout << "  -> 两个读者看到了不同的写入顺序！"
                  << " T3 认为 x 先于 y 写入，T4 认为 y 先于 x 写入。\n";
        std::cout << "  -> 这说明没有全局统一的顺序。\n";
    } else {
        std::cout << "  -> 所有读者看到了一致的写入顺序（全局顺序）或碰巧没抓到。\n";
    }
}

int main() {
    std::cout << "=== IRIW Test: relaxed ===\n";
    test_iriw("relaxed", std::memory_order_relaxed, std::memory_order_relaxed);

    std::cout << "\n=== IRIW Test: acquire/release ===\n";
    test_iriw("acq_rel", std::memory_order_release, std::memory_order_acquire);

    std::cout << "\n=== IRIW Test: seq_cst ===\n";
    test_iriw("seq_cst", std::memory_order_seq_cst, std::memory_order_seq_cst);

    std::cout << "\n结论：\n";
    std::cout << "- seq_cst 保证所有 seq_cst 操作有统一的全局顺序\n";
    std::cout << "- acquire/release 只在线程间建立 PAIRWISE 的同步\n";
    std::cout << "- 在 x86 上由于强内存模型，acquire/release 也经常 \"碰巧\" 一致\n";
    std::cout << "- 在 ARM 等弱内存模型上，acquire/release 的 violation 会出现\n";

    return 0;
}
```

---

#### 实验 6：伪共享（False Sharing）— 量化性能影响

这个实验让你亲眼看到伪共享对性能的巨大影响。

```cpp
// exp_false_sharing.cpp
// 编译: g++ -std=c++17 -pthread -O2 exp_false_sharing.cpp -o exp_false_sharing
#include <iostream>
#include <thread>
#include <vector>
#include <chrono>
#include <atomic>

constexpr int N_INCREMENTS = 100000000;  // 1 亿次

// ====== 结构 1：无 padding — 两个计数器可能在同一 cache line ======
struct NoPadding {
    alignas(64) std::atomic<int64_t> a{0};  // alignas(64) 这里只让 a 对齐到 cache line 开头
    std::atomic<int64_t> b{0};   // b 紧挨着 a，大概率在同一 cache line
};

// ====== 结构 2：有 padding — 每个计数器独占一条 cache line ======
struct WithPadding {
    alignas(64) std::atomic<int64_t> a{0};
    char padding[64];  // 把 b 推到下一条 cache line
    alignas(64) std::atomic<int64_t> b{0};
};

// 或用 C++17 标准方式（如果编译器支持）
// alignas(std::hardware_destructive_interference_size) std::atomic<int64_t> a{0};
// alignas(std::hardware_destructive_interference_size) std::atomic<int64_t> b{0};

template<typename T>
double benchmark(T& counters) {
    auto worker_a = [&] {
        for (int i = 0; i < N_INCREMENTS; ++i)
            counters.a.fetch_add(1, std::memory_order_relaxed);
    };
    auto worker_b = [&] {
        for (int i = 0; i < N_INCREMENTS; ++i)
            counters.b.fetch_add(1, std::memory_order_relaxed);
    };

    auto start = std::chrono::high_resolution_clock::now();
    std::thread t1(worker_a), t2(worker_b);
    t1.join(); t2.join();
    auto end = std::chrono::high_resolution_clock::now();

    return std::chrono::duration<double, std::milli>(end - start).count();
}

int main() {
    std::cout << "Cache line size (typical): 64 bytes\n";
    std::cout << "Each increment: " << N_INCREMENTS << " fetch_add calls\n\n";

    // 多次测量取平均
    constexpr int RUNS = 5;

    {
        double total = 0;
        for (int i = 0; i < RUNS; ++i) {
            NoPadding c;
            total += benchmark(c);
        }
        std::cout << "[No padding]  avg: " << total / RUNS << " ms\n";
    }

    {
        double total = 0;
        for (int i = 0; i < RUNS; ++i) {
            WithPadding c;
            total += benchmark(c);
        }
        std::cout << "[With padding] avg: " << total / RUNS << " ms\n";
    }

    std::cout << "\n";
    std::cout << "★ 如果 No padding 明显更慢，就是伪共享在作祟：\n";
    std::cout << "  两个线程分别写 a 和 b，但它们在同一 cache line 上，\n";
    std::cout << "  cache coherence 协议导致 cache line 在两个核之间来回 \"乒乓\"。\n";
    std::cout << "  With padding 让 a 和 b 在不同 cache line，避免了伪共享。\n";

    return 0;
}
```

**预期结果**：在大多数多核 CPU 上，`NoPadding` 耗时可能是 `WithPadding` 的 **3-10 倍**。

---

#### 实验 7：无锁栈 — CAS 实战 + ABA 问题演示

用 CAS 实现一个最简单的无锁栈，然后演示 ABA 问题的本质。

```cpp
// exp_lockfree_stack.cpp
// 编译: g++ -std=c++17 -pthread -O2 exp_lockfree_stack.cpp -o exp_lockfree_stack
#include <iostream>
#include <thread>
#include <atomic>
#include <vector>
#include <chrono>

// ====== 最简单的无锁栈（有 ABA 风险的版本） ======
template<typename T>
class LockFreeStack {
    struct Node {
        T data;
        Node* next;
        Node(T d) : data(d), next(nullptr) {}
    };

    std::atomic<Node*> head_{nullptr};

public:
    void push(T val) {
        Node* new_node = new Node(val);
        // CAS 循环：尝试把 new_node 设为 head
        do {
            new_node->next = head_.load(std::memory_order_relaxed);
        } while (!head_.compare_exchange_weak(
            new_node->next, new_node,
            std::memory_order_release,
            std::memory_order_relaxed));
    }

    bool pop(T& result) {
        Node* old_head;
        do {
            old_head = head_.load(std::memory_order_acquire);
            if (!old_head) return false;  // 栈空
        } while (!head_.compare_exchange_weak(
            old_head, old_head->next,
            std::memory_order_acquire,
            std::memory_order_relaxed));

        result = old_head->data;
        // ★ ABA 风险：如果此时另一个线程 pop 了 old_head，又 push 了回来，
        //   且 old_head 指向的 Node 被释放后新 Node 恰好分配到同一地址，
        //   这里的 delete 会导致 use-after-free 或 double-free。
        // 生产环境需要 hazard pointer / epoch-based reclamation / shared_ptr<atomic>
        // delete old_head;  // 故意注释掉——正确的内存回收需要额外机制
        return true;
    }

    ~LockFreeStack() {
        // 简化：仅用于演示，不完整回收
        Node* n = head_.load();
        while (n) {
            Node* next = n->next;
            delete n;
            n = next;
        }
    }
};

int main() {
    constexpr int N_THREADS = 4;
    constexpr int N_PER_THREAD = 25000;
    constexpr int TOTAL = N_THREADS * N_PER_THREAD;

    LockFreeStack<int> stack;

    // ====== 多线程 push ======
    std::vector<std::thread> producers;
    for (int i = 0; i < N_THREADS; ++i) {
        producers.emplace_back([&stack, i] {
            for (int j = 0; j < N_PER_THREAD; ++j) {
                stack.push(i * N_PER_THREAD + j);
            }
        });
    }
    for (auto& t : producers) t.join();

    std::cout << "All pushes done.\n";

    // ====== 多线程 pop ======
    std::atomic<int> pop_count{0};
    std::mutex cout_mutex;

    std::vector<std::thread> consumers;
    for (int i = 0; i < N_THREADS; ++i) {
        consumers.emplace_back([&] {
            int val;
            while (pop_count.fetch_add(1) < TOTAL) {
                if (stack.pop(val)) {
                    // pop 成功
                } else {
                    break;  // 栈空
                }
            }
        });
    }
    for (auto& t : consumers) t.join();

    std::cout << "All pops done. (elements expected: " << TOTAL << ")\n";
    std::cout << "\n注意：这个栈在 push/pop 在并发下正确工作，\n";
    std::cout << "但 delete 已注释掉——因为存在 ABA 问题！\n";
    std::cout << "解决方案：hazard pointer / epoch-based RCU / atomic<shared_ptr>（C++20）\n";

    return 0;
}
```

---

#### 实验 8：happens-before 侦探游戏

通过 mutex、atomic、thread join 建立 happens-before 关系，然后用断言验证。

```cpp
// exp_happens_before.cpp
// 编译: g++ -std=c++17 -pthread -O2 exp_happens_before.cpp -o exp_happens_before
#include <iostream>
#include <thread>
#include <mutex>
#include <atomic>
#include <cassert>

std::mutex cout_mutex;
#define LOG(msg) do { \
    std::lock_guard<std::mutex> _lg(cout_mutex); \
    std::cout << msg << std::endl; \
} while(0)

int main() {
    // ====== happens-before 链 1：mutex lock/unlock ======
    {
        std::mutex m;
        int shared = 0;

        std::thread t1([&] {
            shared = 100;              // (A)
            {
                std::lock_guard lg(m);  // (B) lock
                // 临界区
            }                           // (C) unlock → happens-before → (D)
        });

        t1.join();  // t1 的所有操作 happens-before join 返回

        std::thread t2([&] {
            std::lock_guard lg(m);      // (D) lock → 能看到 (C) 之前的所有写
            // shared 可能是 100（如果 t1 的锁在 t2 之前释放）
            // 也可能是 0（如果 t2 先获得锁）
            LOG("shared via mutex: " + std::to_string(shared));
        });

        t2.join();
    }

    // ====== happens-before 链 2：release/acquire ======
    {
        std::atomic<bool> flag{false};
        int data = 0;

        std::thread t1([&] {
            data = 42;                                         // (E)
            flag.store(true, std::memory_order_release);       // (F) release
            // (E) happens-before (F) [sequenced-before, 同线程]
        });

        std::thread t2([&] {
            while (!flag.load(std::memory_order_acquire));     // (G) acquire
            // (F) synchronizes-with (G) [因为 release/acquire 配对]
            // 所以 (E) happens-before (G) [通过传递]
            // 所以 (G) 之后一定能看到 (E) 的结果
            assert(data == 42);  // 绝对不会触发
            LOG("data via release/acquire: " + std::to_string(data));
        });

        t1.join(); t2.join();
    }

    // ====== happens-before 链 3：thread join ======
    {
        int result = 0;
        std::thread t([&] {
            result = 999;  // (H)
        });
        t.join();           // (I) — (H) happens-before (I)
        // join 返回后，result 一定等于 999
        assert(result == 999);  // 绝对不会触发
        LOG("result via join: " + std::to_string(result));
    }

    LOG("\n所有断言都通过了——happens-before 关系保证了这些操作的可见性。");
    LOG("如果去掉同步机制（不用 mutex、不用 release/acquire、不 join），");
    LOG("这些断言就可能失败——数据竞赛、cppreference 称之为未定义行为。");

    return 0;
}
```

---

#### 实验 9：async / future / promise / latch / barrier

演示 C++11/20 的异步基础设施。

```cpp
// exp_async_future.cpp
// 编译: g++ -std=c++20 -pthread -O2 exp_async_future.cpp -o exp_async_future
// （latch/barrier 需要 C++20）
#include <iostream>
#include <future>
#include <thread>
#include <latch>     // C++20
#include <barrier>   // C++20
#include <vector>
#include <chrono>

// ====== async + future ======
void demo_async_future() {
    std::cout << "=== async + future ===\n";

    // launch::async: 保证在新线程中执行
    auto future = std::async(std::launch::async, [] {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        return 42;
    });

    std::cout << "Waiting for result...\n";
    int result = future.get();  // 阻塞直到完成
    std::cout << "Result: " << result << "\n\n";
}

// ====== promise + future ======
void demo_promise_future() {
    std::cout << "=== promise + future ===\n";

    std::promise<std::string> p;
    std::future<std::string> f = p.get_future();

    std::thread worker([&p] {
        std::this_thread::sleep_for(std::chrono::milliseconds(50));
        p.set_value("hello from worker");  // 设置结果到 promise
        // 也可以 p.set_exception(...) 传递异常
    });

    std::cout << "Received: " << f.get() << "\n";
    worker.join();
    std::cout << "\n";
}

// ====== shared_future — 一个 future 多次获取 ======
void demo_shared_future() {
    std::cout << "=== shared_future ===\n";

    std::promise<int> p;
    std::shared_future<int> sf = p.get_future().share();  // future → shared_future

    auto reader = [sf](int id) {
        int val = sf.get();  // 多个线程都可以 get()
        std::cout << "  Thread " << id << " got: " << val << "\n";
    };

    std::thread t1(reader, 1), t2(reader, 2), t3(reader, 3);
    std::this_thread::sleep_for(std::chrono::milliseconds(10));
    p.set_value(100);  // 所有等待的 shared_future 都被通知
    t1.join(); t2.join(); t3.join();
    std::cout << "\n";
}

// ====== latch：一次性同步点 ======
void demo_latch() {
    std::cout << "=== latch (C++20) ===\n";

    constexpr int N_WORKERS = 5;
    std::latch start_gate{1};       // 等 1 个信号后开始
    std::latch done_gate{N_WORKERS}; // 等 N_WORKERS 个 worker 都完成

    auto worker = [&](int id) {
        start_gate.wait();  // 等待起跑信号
        std::cout << "  Worker " << id << " started\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(id * 20));
        std::cout << "  Worker " << id << " done\n";
        done_gate.count_down();  // 报告完成
    };

    std::vector<std::thread> workers;
    for (int i = 0; i < N_WORKERS; ++i)
        workers.emplace_back(worker, i);

    std::cout << "Ready... Go!\n";
    start_gate.count_down();  // 所有 worker 同时起跑

    done_gate.wait();  // 等所有 worker 完成
    std::cout << "All workers done!\n";

    for (auto& t : workers) t.join();
    std::cout << "\n";
}

// ====== barrier：可复用的同步点 ======
void demo_barrier() {
    std::cout << "=== barrier (C++20) ===\n";

    constexpr int N_THREADS = 4;
    constexpr int N_ROUNDS = 3;

    // barrier 在每次所有线程到达后执行回调，然后重置
    std::barrier sync_point(N_THREADS, [round = 0]() mutable {
        std::cout << "  [All threads reached barrier for round " << round++ << "]\n";
    });

    auto worker = [&](int id) {
        for (int round = 0; round < N_ROUNDS; ++round) {
            // 模拟每轮做不同量的工作
            std::this_thread::sleep_for(std::chrono::milliseconds((id + 1) * 30));
            std::cout << "  Thread " << id << " reached barrier (round " << round << ")\n";
            sync_point.arrive_and_wait();  // 到达并等待所有线程
        }
    };

    std::vector<std::thread> threads;
    for (int i = 0; i < N_THREADS; ++i)
        threads.emplace_back(worker, i);
    for (auto& t : threads) t.join();

    std::cout << "All rounds complete.\n";
}

int main() {
    demo_async_future();
    demo_promise_future();
    demo_shared_future();
    demo_latch();
    demo_barrier();
    return 0;
}
```

---

#### 实验 10：综合 — 自己动手测

把前面的知识点串起来，用 `#define` 切换不同的配置做对比实验。

```cpp
// exp_bench_all.cpp
// 编译: g++ -std=c++17 -pthread -O2 exp_bench_all.cpp -o exp_bench_all
// 这个实验让你一键对比：无锁 vs mutex vs atomic (不同 memory order)
#include <iostream>
#include <thread>
#include <mutex>
#include <atomic>
#include <vector>
#include <chrono>
#include <string>

constexpr int N_THREADS = 4;
constexpr int64_t N_OPS = 10000000;  // 每线程 1000 万次

// ====== 方案 1：mutex ======
double bench_mutex() {
    std::mutex m;
    int64_t counter = 0;

    auto worker = [&] {
        for (int64_t i = 0; i < N_OPS; ++i) {
            std::lock_guard<std::mutex> lg(m);
            ++counter;  // 被保护的普通变量
        }
    };

    auto start = std::chrono::high_resolution_clock::now();
    std::vector<std::thread> threads;
    for (int i = 0; i < N_THREADS; ++i)
        threads.emplace_back(worker);
    for (auto& t : threads) t.join();
    auto end = std::chrono::high_resolution_clock::now();

    std::cout << "  counter = " << counter
              << " (expected " << N_THREADS * N_OPS << ")\n";
    return std::chrono::duration<double, std::milli>(end - start).count();
}

// ====== 方案 2：atomic (seq_cst, 默认) ======
double bench_atomic_seq_cst() {
    std::atomic<int64_t> counter{0};

    auto worker = [&] {
        for (int64_t i = 0; i < N_OPS; ++i) {
            counter.fetch_add(1, std::memory_order_seq_cst);
        }
    };

    auto start = std::chrono::high_resolution_clock::now();
    std::vector<std::thread> threads;
    for (int i = 0; i < N_THREADS; ++i)
        threads.emplace_back(worker);
    for (auto& t : threads) t.join();
    auto end = std::chrono::high_resolution_clock::now();

    std::cout << "  counter = " << counter.load()
              << " (expected " << N_THREADS * N_OPS << ")\n";
    return std::chrono::duration<double, std::milli>(end - start).count();
}

// ====== 方案 3：atomic (relaxed) ======
double bench_atomic_relaxed() {
    std::atomic<int64_t> counter{0};

    auto worker = [&] {
        for (int64_t i = 0; i < N_OPS; ++i) {
            counter.fetch_add(1, std::memory_order_relaxed);
        }
    };

    auto start = std::chrono::high_resolution_clock::now();
    std::vector<std::thread> threads;
    for (int i = 0; i < N_THREADS; ++i)
        threads.emplace_back(worker);
    for (auto& t : threads) t.join();
    auto end = std::chrono::high_resolution_clock::now();

    std::cout << "  counter = " << counter.load()
              << " (expected " << N_THREADS * N_OPS << ")\n";
    return std::chrono::duration<double, std::milli>(end - start).count();
}

// ====== 方案 4：atomic 但用 local accumulation（减少竞争） ======
double bench_atomic_local_acc() {
    std::atomic<int64_t> counter{0};

    auto worker = [&] {
        int64_t local = 0;  // 先在本地累加
        for (int64_t i = 0; i < N_OPS; ++i) {
            ++local;
        }
        counter.fetch_add(local, std::memory_order_relaxed);  // 最后一次性加
    };

    auto start = std::chrono::high_resolution_clock::now();
    std::vector<std::thread> threads;
    for (int i = 0; i < N_THREADS; ++i)
        threads.emplace_back(worker);
    for (auto& t : threads) t.join();
    auto end = std::chrono::high_resolution_clock::now();

    std::cout << "  counter = " << counter.load()
              << " (expected " << N_THREADS * N_OPS << ")\n";
    return std::chrono::duration<double, std::milli>(end - start).count();
}

int main() {
    std::cout << "Benchmark: " << N_THREADS << " threads, "
              << N_OPS << " increments each\n\n";

    auto print = [](const std::string& name, double ms) {
        std::cout << "[" << name << "] " << ms << " ms\n";
    };

    print("mutex        ", bench_mutex());
    print("atomic seqcst", bench_atomic_seq_cst());
    print("atomic relaxed", bench_atomic_relaxed());
    print("local acc    ", bench_atomic_local_acc());

    std::cout << "\n分析：\n";
    std::cout << "- mutex: 每次加都加锁，开销最高\n";
    std::cout << "- seq_cst: 无需内核态切换，比 mutex 快，但每次有内存屏障\n";
    std::cout << "- relaxed: 无内存屏障，最快——适合纯计数器\n";
    std::cout << "- local accumulation: 极少竞争，吞吐量取决于本地计算速度\n";

    return 0;
}
```

**预期结果**（相对速度）：
```
local acc > atomic relaxed > atomic seq_cst > mutex
```

---

#### 实验 11（选做）：ThreadSanitizer 实战

ThreadSanitizer 是检测数据竞争的利器。用实验 3 的代码跑一下：

```bash
# 用 ThreadSanitizer 编译和运行
g++ -std=c++17 -pthread -O2 -fsanitize=thread -g exp_atomic_basic.cpp -o tsan_test
./tsan_test 2>&1 | head -50
```

你会看到类似这样的输出（对于非原子版本）：
```
WARNING: ThreadSanitizer: data race (pid=12345)
  Write of size 4 at 0x... by thread T2:
    #0 worker() exp_atomic_basic.cpp:17
  Previous write of size 4 at 0x... by thread T1:
    #0 worker() exp_atomic_basic.cpp:17
```

而对于 `fetch_add` 版本——静默通过，没有 data race。

---

> **实验总结**：这些实验覆盖了第八章中的所有核心概念。建议按顺序跑一遍，每次改动一个小参数（如 memory_order、线程数）观察变化。如果你在 x86 上跑某些内存序实验发现 " 怎么都是对的 "，不要疑惑——这正是 x86 强内存模型的特点。想办法在 ARM 设备（树莓派 / Apple Silicon）上再跑一遍，你会看到明显差异。

---

## 九、C++ 容易翻车的点

1. 把 `std::move` 说成 " 强制移动 "
2. 说不清拷贝构造和拷贝赋值区别
3. 分不清右值引用、万能引用、完美转发
4. 把 `map`、`unordered_map`、`vector` 的复杂度和失效规则说乱
5. 把虚函数表当成标准明文规定
6. 说自己懂并发，但讲不清 `atomic` 和 `mutex` 的边界
7. 知道 `memory_order` 名字，但一问同步直觉就崩
8. 不知道 happens-before 和 synchronizes-with 的区别
9. `std::move` 一个 `const` 对象后发现还是拷贝了
10. 不知道 `compare_exchange_weak` 的 " 虚假失败 "

---

## 十、C++ 项目里怎么说

适合你的口径可以更强一点：

- " 我对 C++ 更熟的是对象生命周期、RAII、STL 容器、值类别和并发基础设施。"
- " 如果结合项目，我可以展开容器选择、资源管理、接口设计、异常安全和并发同步这些工程问题。"
- " 模板元编程和极底层 ABI 细节我理解概念和常见行为，但不会把自己定义成专门写模板库或编译器的人。"

---

## 十一、CUDA 主线

如果简历上写熟悉 `CUDA`，面试官通常会默认你至少要能讲清：

1. 为什么 GPU 适合并行计算
2. thread / block / grid / warp
3. 内存层次
4. 性能瓶颈从哪里来
5. 工程里怎么定位慢点

---

## 十二、CUDA 执行模型

### 1. CPU vs GPU：根本差异

| 维度 | CPU | GPU |
|---|---|---|
| 设计目标 | 低延迟、高单线程性能 | 高吞吐、大规模并行 |
| 核心数 | 少数强核心（~几到几十） | 大量轻量核心（~数千） |
| 控制逻辑 | 分支预测、乱序执行、大缓存 | 简单控制、顺序执行、小缓存 |
| 线程切换 | 昂贵（保存恢复上下文） | 近乎免费（硬件 warp scheduler） |
| 内存带宽 | ~100 GB/s | ~1 TB/s（HBM） |
| SIMD 宽度 | AVX-512 ~512 bit | Warp 32 线程 × 每线程宽度 |

核心直觉：GPU 用大量线程来隐藏内存延迟。当一个 warp 等待数据时，SM 立即切换到另一个就绪的 warp——这就是为什么 occupancy（活跃 warp 数）重要。GPU 本质上是**吞吐量架构**而非延迟架构。

> **思考题：为什么 GPU 有这么多核心但每个核心性能远不如 CPU？**
>
> 这是设计哲学的根本差异——" 晶圆面积预算 " 的取舍。CPU 把大量晶体管用于：
> - 分支预测器、乱序执行引擎
> - 大容量 L1/L2/L3 缓存（减少延迟）
> - 复杂指令调度逻辑
>
> GPU 则把同样的晶体管预算用于塞进更多计算单元（ALU/FPU）。每个 GPU 核心简单得多，但数量是 CPU 的数十到数百倍。GPU 还通过 warp 调度将控制逻辑分摊到 32 个线程上——一条指令对所有线程同时生效，大幅节省了取指和译码开销。这使得 GPU 在规则、数据密集的工作负载上获得极大的吞吐量优势。

### 2. thread / block / grid：完整层次结构

```
Grid (1 次 kernel launch)
├── Block (0,0)                  ← 可以 shared memory 同步
│   ├── Thread 0 → Thread 31    ← 同一 warp，SIMT 执行
│   ├── Thread 32 → Thread 63   ← 另一个 warp
│   └── ...
├── Block (1,0)
│   └── ...
└── Block (N,0)
```

**为什么要有 Block？**
- Block 内线程可以通过 `shared memory` 共享数据
- Block 内线程可以用 `__syncthreads()` 做屏障同步
- 一个 Block 被调度到一个 SM 上执行，所有它的线程共享同一个 SM 的资源（寄存器文件、shared memory）
- Block 之间**不能**同步（除了通过 global memory + atomic + kernel 结束）

> **思考题：grid/block/thread 分别应该如何选择维度？**
>
> - **Thread**：最小执行单元。具体每个线程处理多少数据由你决定（可以每个线程处理一个元素，也可以处理多个）
> - **Block**：最关键的调参目标。需要考虑：
>   - Block size 必须是 warp size（32）的整数倍，常见选择 128/256/512
>   - Block size 太小 → SM 上可容纳的 block 数可能不够，occupancy 不足
>   - Block size 太大 → 寄存器或 shared memory 成为瓶颈，每个 SM 能容纳的 block 数受限
>   - 硬件限制：一个 block 最多 1024 个线程（取决于架构）
> - **Grid**：`gridDim = ceil(total_threads / blockDim)`。Grid 大小受硬件限制很小（通常可以达到 2^31 - 1），让你的 kernel 覆盖所有数据即可
>
> 实践建议：从 block size = 256 开始调（大多数架构上的 sweet spot），然后根据寄存器和 shared memory 使用量调整。

> **思考题：`<<<gridDim, blockDim, sharedMemBytes, stream>>>` 中 shared memory 大小为什么要显式指定？**
>
> 静态 shared memory 在编译期声明：`__shared__ float cache[256];`。但有时你的 shared memory 大小取决于 kernel 参数或模板参数——此时你需要动态 shared memory：
> ```cpp
> extern __shared__ float dynamic_cache[];
> kernel<<<grid, block, dynamic_bytes, stream>>>(...);
> ```
> 这种方式让你在运行时根据数据尺寸来分配 shared memory，但要注意：一个 kernel 只能声明一个 extern `__shared__` 数组（需要手动计算偏移量容纳多个数组）。

### 3. warp：SIMT 的核心

- warp 是硬件执行的**真正调度粒度**——SM 的 warp scheduler 以 warp 为单位发射指令
- 1 warp = 32 个线程（NVIDIA 所有架构至今如此）
- 同一 warp 的线程同时执行同一条指令（SIMT: Single Instruction, Multiple Threads）
- 线程有自己的寄存器和程序计数器，但执行的是同一条指令

> **思考题：如果 warp 内所有线程始终执行相同指令，那 " 多线程 " 的意义何在？**
>
> 答案在同一句话中：Multiple **Threads**——每个线程操作的数据不同。指令相同，但寄存器 / shared memory / global memory 地址不同：
> ```cpp
> int idx = threadIdx.x;       // 每个线程有不同的 idx
> output[idx] = input[idx] * 2;  // 同一条指令，但访问不同地址
> ```
> 这就是数据并行的本质：相同的操作用在不同的数据上。GPU 的寄存器文件为每个线程独立分配，所以 " 同一条指令 " 实际是 " 同一运算，不同操作数 "。

#### 3.1 Warp Divergence（分支发散）—— 高频考点

```cpp
// 不好的写法：分支导致发散
if (threadIdx.x % 2 == 0) {
    // 偶数线程走这条路
    result = heavy_computation_a(data);   // 此时奇数线程闲置等待
} else {
    // 奇数线程走这条路
    result = heavy_computation_b(data);   // 此时偶数线程闲置等待
}
// 实际利用率只有 50%
```

**为什么发散慢？** 同一 warp 的线程共享一个程序计数器（PC）。如果分支让部分线程走 if、部分走 else，硬件只能串行执行两条路径——先执行一边（另一边线程被 mask 掉），再执行另一边。线程利用率降低。

> **思考题：如何避免或减轻 warp divergence？**
>
> 1. **重构数据结构**：如果分支取决于数据，试试对数据排序让相邻线程走相同分支
> ```cpp
> // 先按类型排序，让同一 warp 的线程处理同类型数据
> ```
>
> 2. **用算术替代分支**：
> ```cpp
> // 不好：if (cond) result = a; else result = b;
> // 更好：result = cond ? a : b;  // 大多数编译器会用 select/predicated 指令
> ```
>
> 3. **分支粒度对齐 warp 边界**：
> ```cpp
> if (threadIdx.x / 32 == 0) { ... }  // warp 0 走一路
> else { ... }                         // warp 1 走另一路（整个 warp 统一，无发散）
> ```
>
> 4. **接受小发散**：如果发散发生在 warp 内部的少数线程，开销是可控的。过度优化可能会让代码更难理解。
>
> 关键面试回答：我能解释发散为什么慢（共享 PC，串行化分支路径），知道常见减轻手段（数据重排、谓词指令、warp 级分支），但不会声称可以消除所有发散。

---

## 十三、CUDA 内存层次（详细版）

CUDA 的内存层次是决定 kernel 性能的最关键因素。GPU 没有像 CPU 那样的大缓存层级，程序员需要**显式管理数据的存放位置**。

```
最快/最小                              最慢/最大
  Register → Shared Memory → L1/L2 Cache → Global Memory → Host Memory
  (~0 cycle)  (~20-30 cycles) (~200 cycles)  (~400-800 cycles)  (PCIe)

  + Constant Memory: 有专用 cache，适合广播
  + Texture Memory: 有专用 cache，2D 空间局部性优化
  + Local Memory: 寄存器溢出时使用（本质上是 global memory 的一部分）
```

### 1. Register（寄存器）

- 最快，吞吐量与计算单元相同
- 每线程私有，编译期分配
- 数量有限：每 SM 有寄存器文件（如 Volta ~64K 寄存器/SM）
- 如果 kernel 使用太多局部变量 → 寄存器溢出（spill）到 local memory → 性能急剧下降

> **思考题：怎么看一个 kernel 用了多少寄存器？怎么减少寄存器压力？**
>
> 查看：编译时加 `--ptxas-options=-v` 或 `-Xptxas=-v`，输出会显示 `regcount`。或在 `nsight` profiler 中查看。
>
> 减少寄存器的方法：
> - 用 `__launch_bounds__` 提示编译器限制寄存器使用（如 `__launch_bounds__(256, 2)` 表示 block 最多 256 线程，每个 SM 至少放 2 个 block）
> - 减少局部变量数量、缩小变量作用域
> - 把大中间结果存到 shared memory
> - 减少循环展开（`#pragma unroll` 限制）

### 2. Shared Memory

- 片上 SRAM，延迟约 20-30 cycles（接近 L1 cache）
- Block 内所有线程共享
- 典型大小：每 SM 约 64KB-164KB（可配置 L1/shared 划分）
- 适合：tile 运算、线程间数据交换、减少 global memory 重复读取

```cpp
// 矩阵乘法的 tile 示例（核心思路）
__global__ void matmul_tiled(const float* A, const float* B, float* C, int N) {
    __shared__ float As[TILE_SIZE][TILE_SIZE];
    __shared__ float Bs[TILE_SIZE][TILE_SIZE];

    int row = blockIdx.y * TILE_SIZE + threadIdx.y;
    int col = blockIdx.x * TILE_SIZE + threadIdx.x;
    float sum = 0.0f;

    for (int t = 0; t < N / TILE_SIZE; ++t) {
        As[threadIdx.y][threadIdx.x] = A[row * N + t * TILE_SIZE + threadIdx.x];
        Bs[threadIdx.y][threadIdx.x] = B[(t * TILE_SIZE + threadIdx.y) * N + col];
        __syncthreads();

        for (int k = 0; k < TILE_SIZE; ++k)
            sum += As[threadIdx.y][k] * Bs[k][threadIdx.x];
        __syncthreads();
    }
    C[row * N + col] = sum;
}
```

#### 2.1 Bank Conflict（Bank 冲突）—— 高频考点

- Shared memory 被划分为 32 个 bank（与 warp size 一致）
- 每 bank 每周期可以服务一个地址
- 同一 warp 的多个线程访问同一 bank 的不同地址 → **bank conflict** → 请求串行化

```cpp
// 无冲突：相邻线程访问相邻 bank（stride = 1）
// Thread i 访问 shared[i] → 32 个线程访问 32 个不同 bank → 并行

// 有冲突：步长为 2
// Thread i 访问 shared[i * 2] → 线程 0,16 打到 bank 0，串行化 → 2-way conflict

// 最严重：步长为 32
// 所有线程打到同一个 bank → 32-way conflict → 串行化！
```

> **思考题：如何检测和解决 bank conflict？**
>
> 检测方法：
> - nsight profiler 中 shared memory 的 bank conflict 指标
> - 手动分析：查看 shared memory 的访问模式，计算 `(地址/4) % 32` 看多少线程命中同一 bank
>
> 解决方法：
> 1. **Padding**：在 shared memory 数组末尾加一列
> ```cpp
> __shared__ float tile[TILE][TILE + 1];  // +1 改变 bank 映射，消除 stride 冲突
> ```
> 2. **改变访问模式**：调整线程和数据的映射关系
> 3. **用向量类型**：`float4` 一次读取 4 个 float，减少访存次数
> 4. **重新排列数据**：按列存储改为按行存储

### 3. Global Memory（全局内存 / 显存）

- 容量最大（数 GB 到数十 GB），但延迟最高（400-800 cycles）
- Grid 中所有线程都可以访问
- 通过 L2 cache 缓存（所有 SM 共享，约 6-40 MB）
- 性能关键：**合并访问（Coalescing）**

#### 3.1 Coalesced Access（合并访问）

这是 GPU 性能优化的第一课。

```cpp
// 合并访问（coalesced）：一次内存事务服务 warp 中 32 个线程
// 前提：32 个线程访问的是连续的、对齐的 128 字节段
float x = data[blockIdx.x * blockDim.x + threadIdx.x];  // 连续访问 → 合并

// 非合并访问（strided）：
float x = data[threadIdx.x * N];  // stride = N → 访存效率 = 1/N

// 非合并访问（随机）：
float x = data[permutation[threadIdx.x]];  // 随机访问 → 效率极低
```

> **思考题：为什么合并访问对性能影响如此巨大？**
>
> GPU 的 global memory 以 memory transaction 为单位传输数据——一次事务传输 32/64/128 字节。当 warp 中的 32 个线程访问一段连续的 128 字节时，只需 1 次 transaction。但如果这 32 个线程散布在完全不连续的地址上，可能需要最多 32 次 transaction——有效带宽下降到 1/32。
>
> 这就是为什么：
> - SoA（Structure of Arrays）通常比 AoS（Array of Structures）更适合 GPU：`float[3] * N` 的 x 分量连续访问是合并的，而 `struct{float x,y,z;}` 的 x 分量访问 stride=3，效率低
> - 图像处理中，通道在连续维度上的排列会影响访存效率

### 4. Constant Memory / Texture Memory / Local Memory

- **Constant Memory**（常量内存）：64KB，有专用 cache。同一 warp 的所有线程读取同一地址时效率最高（广播）。适合卷积核、系数表等只读常量数据
- **Texture Memory**（纹理内存）：为 2D 空间局部性优化，有专用 cache。适合图像处理、插值
- **Local Memory**：名字有误导性。它是寄存器的 " 溢出区 "，物理上在 global memory 中，只是每个线程私有的。寄存器不够用时自动 spill 到这里，性能很差

---

## 十四、CUDA 性能优化高频点（详细版）

### 1. Occupancy（占用率）

```
occupancy = active_warps_per_SM / max_warps_per_SM
```

- 高 occupancy 允许 SM 在等待一个 warp 时切换执行另一个，隐藏延迟
- **但 occupancy 不是越高越好**——寄存器或 shared memory 用得越多，每个 SM 能容纳的 block 越少

> **思考题：什么时候低 occupancy 反而比高 occupancy 更好？**
>
> 如果每个线程做了大量的计算和少量的内存访问（计算密集型 kernel），寄存器用得多导致 occupancy 降低，但只要每个 warp 的计算足够密集、能让 SM 持续有指令可发射，低 occupancy 也可以接受。相反，如果一个 kernel 完全被内存延迟限制（访存密集型），高 occupancy 让对方多切换 warp 来隐藏内存延迟就非常重要。
>
> 简单判断：
> - 计算密集（Compute-bound）：occupancy 不那么重要，优先给更多寄存器
> - 访存密集（Memory-bound）：occupancy 很重要，可能需要减少寄存器/ shared memory 使用来容纳更多 block
>
> 实用方法：用 nsight 看 `sm__throughput` 和 `memory_throughput`，判断你是 bound 在哪里。

### 2. Block Size 调优

```cpp
// 典型调参空间
for (int bs : {64, 128, 256, 512, 1024}) {
    kernel<<<num_blocks, bs>>>(...);
    // profile, 记录时间
}
```

Block size 影响：
- 太小：SM 上可能 block 数量受硬件上限限制，warp 数不够 → occupancy 低
- 太大：寄存器 / shared memory 总量限制每个 SM 只能容纳很少的 block → occupancy 也低
- 实践建议：从 warp size 的整数倍（128/256）开始测

### 3. Kernel Launch Overhead

- 每次 `<<<>>>` 调用有固定开销（驱动层、参数拷贝等）
- 如果 launch 大量微型 kernel（如每个 kernel 只处理 100 个元素），launch overhead 占总时间的比例可能超过实际计算
- 解决方案：kernel fusion（把多个小 kernel 合并成一个大 kernel）、使用 CUDA Graph（C++ 图捕获重放）

### 4. 隐式同步点

以下操作可能引入隐式的全局同步，影响异步流水线的设计：
- `cudaMalloc` / `cudaFree`：跨所有 stream 的同步
- `cudaDeviceSynchronize`：显式全局同步
- 默认 stream（stream 0）上的操作

---

## 十五、CUDA 进阶工程点（详细版）

### 1. Stream 和异步并发

**为什么需要 Stream？** GPU 可以在执行一个 kernel 的同时做另一件事（如传输数据），暴露并发性可以提升整体吞吐。

```cpp
cudaStream_t s1, s2;
cudaStreamCreate(&s1);
cudaStreamCreate(&s2);

// 流 1：传输 + 计算
cudaMemcpyAsync(d_a, h_a, size, cudaMemcpyHostToDevice, s1);
kernel<<<grid, block, 0, s1>>>(d_a, d_out);
cudaMemcpyAsync(h_out, d_out, size, cudaMemcpyDeviceToHost, s1);

// 流 2：同时做另一批
cudaMemcpyAsync(d_b, h_b, size, cudaMemcpyHostToDevice, s2);
kernel<<<grid, block, 0, s2>>>(d_b, d_out2);
cudaMemcpyAsync(h_out2, d_out2, size, cudaMemcpyDeviceToHost, s2);
```

关键理解：
- 同一 stream 内的操作是顺序执行的
- 不同 stream 之间的操作可以并发（前提：硬件资源足够、没有隐式同步）
- 默认 stream（不指定或 stream 0）会和其他 stream 隐式同步，注意用 `cudaStreamCreateWithFlags(&s, cudaStreamNonBlocking)` 创建非阻塞流

> **思考题：如何实现 kernel 执行和数据传输的 overlap？**
>
> 核心技巧是用**两个以上的 stream**做流水线（pipelining）：
> ```cpp
> for (int i = 0; i < n_chunks; i++) {
>     // 流 0：传第 i 块 → 计算第 i 块 → 回传第 i 块
>     // 流 1：同时传第 i+1 块...
> }
> ```
> 前提条件：
> - 使用 pinned memory（页锁定内存）进行传输
> - GPU 有独立的拷贝引擎（copy engine，多在高端 GPU 上有 1-2 个）
> - stream 是 non-blocking 的
>
> 用 `cudaEvent_t` 来精确控制各阶段的依赖关系（`cudaStreamWaitEvent`）。

### 2. Pinned Memory（页锁定内存）

- 正常的 `malloc` 分配的是可分页（pageable）内存
- GPU 在 DMA 时需要内存不被换出——所以实际传输时会先拷贝到临时 pinned buffer
- `cudaMallocHost` 分配的 pinned memory 直接可以被 DMA 访问，省去中间拷贝
- 代价：pinned memory 多了会减少 OS 的换页能力，不能无限制使用

> **思考题：什么时候应该用 pinned memory？**
>
> - 数据需要被反复传输到 GPU → pinned memory 性价比高
> - 数据量大且只传一次 → 额外的中间拷贝开销相对于传输时间很小，收益有限
> - 内存紧张的系统 → 谨慎使用，pinned memory 会减少 OS 可换页的内存池

### 3. Unified Memory（统一内存）

```cpp
float *data;
cudaMallocManaged(&data, N * sizeof(float));
// CPU 和 GPU 都可以直接访问 data，无需显式 cudaMemcpy
```

- 编程方便，但性能取决于访问模式
- 早期（Kepler）：数据在 PCIe 上按需迁移，延迟高
- Pascal 及以后：支持按页迁移和 GPU page fault，减少了多余的迁移
- 适合：代码先行调通算法，然后再针对性优化
- 不适合：对性能极度敏感且已明确数据布局的场景

> **思考题：Unified Memory 的 page fault 机制是怎么工作的？为什么预热（warmup）很重要？**
>
> 在 Pascal 及以后架构上，当 GPU 或 CPU 访问 unified memory 中的某页数据，如果该页不在本地，会触发 page fault，然后 CUDA 驱动将该页迁移到请求方。第一次访问时会有迁移开销——这就是为什么第一次 kernel launch 通常更慢（正在建立映射）。在生产代码中通常需要在真正计时的 kernel 之前做一个 warmup 迭代。

### 4. CUDA Graph

- 问题：每次 kernel launch (`<<<>>>`) 都有 CPU 侧开销（驱动调度、验证参数等）
- 解决方案：CUDA Graph 把一系列操作捕获为图，重放时只需要一次提交
- C++ 更高层：`cudaGraphLaunch` 或直接通过 stream capture（C++ 图捕获）

```cpp
// Stream capture 方式
cudaStreamBeginCapture(stream, cudaStreamCaptureModeGlobal);
kernel1<<<...>>>(...);
kernel2<<<...>>>(...);
cudaStreamEndCapture(stream, &graph);
// 后续可以通过 cudaGraphLaunch 重复执行
```

> **思考题：什么场景 CUDA Graph 收益最大？**
>
> - 有许多小 kernel 的重复循环（如训练中的 optimizer step）
> - 需要降低 CPU 调度延迟的实时推理
> - 可以将整个训练 step 捕获为图，减少 launch overhead
>
> 限制：图是静态的（不能有条件分支改变图结构），kernel 参数必须固定或通过更新节点来修改。

### 5. Profiling 思路（完整版）

> **思考题：给你一段 CUDA 代码让它跑得更快，你的排查流程是什么？**

比较稳的工程口径：

**第一步：整体判断**
1. 用 `nvprof` / `nsight systems` / `nsight compute` 跑一次 timeline
2. 看整体时间分布：GPU 时间占比多少？数据传输占比多少？
3. 如果数据传输是大头 → 考虑 pinned memory、异步传输、减少不必要的拷贝

**第二步：定位瓶颈类型**
4. 用 nsight compute 看 kernel 的 roofline：
   - 计算密度 = FLOPs / bytes accessed
   - 对比 GPU 的理论峰值 → 判断是 compute-bound 还是 memory-bound

**第三步：根据瓶颈深入**
5. 如果是 **memory-bound**：
   - 检查 global memory 访存模式（是否 coalesced）
   - 检查 L2 cache hit rate
   - 是否可以利用 shared memory / tiling 减少全局访存
   - 是否有 bank conflict
6. 如果是 **compute-bound**：
   - 是否有 warp divergence
   - 指令混合（FP32/FP64/INT/TensorCore）
   - 是否可以用 TensorCore（fp16/int8）

**第四步：细粒度调优**
7. 看 occupancy（`sm__warps_active.avg.pct_of_peak_sustained_active`）
8. 看 block size、寄存器用量、shared memory 配置
9. 考虑 kernel fusion 减少 launch overhead
10. 检查是否有隐式同步点破坏异步流水线

关键面试表达：" 我一般先用 nsight systems 看整体 timeline 定位主要耗时 kernel，然后用 nsight compute 对该 kernel 做深入分析——看它受限于计算还是访存，再看具体的访存模式、occupancy、warp 发散等指标。不是凭感觉优化，而是跟着 profiling 数据走。"

---

## 十六、CUDA 和 PyTorch 结合时怎么答

面试官经常问：

- 你是会写自定义 CUDA kernel，还是只是会把 tensor 放到 GPU 上

如果你不是重度 CUDA 开发，更稳的说法是：

- " 我主要熟悉训练推理链路中的 GPU 使用和性能直觉，也理解 CUDA 的线程模型、内存层次和常见优化点。"
- " 如果涉及自定义算子，我知道本质上是把自定义 kernel 通过扩展机制接进框架，但不会把自己包装成深度做算子库的人。"

> **思考题：如果 PyTorch 训练慢，从 GPU 角度你怎么排查？**
>
> 1. **看 GPU 利用率**：`nvidia-smi` 看是否接近 100%。如果低，说明 CPU 喂数据不够快（DataLoader 瓶颈）
> 2. **看 kernel 时间分布**：用 `torch.profiler` 或 nsight systems 看 timeline，定位哪些 op 最耗时
> 3. **看是否有过多的小 kernel**：每个 PyTorch op 都是至少一个 kernel launch——过多的 element-wise 操作用掉了大量 launch overhead。可以尝试用 `torch.compile` 或手写 fused kernel 合并
> 4. **检查数据传输**：是否有不必要的 CPU↔GPU 传输（如 `.item()`, `.cpu()` 调用）
> 5. **检查 tensor 是否连续**：不连续的 tensor 可能导致额外的数据移动
> 6. **看是否有过多的 cuda synchronization**：`.item()`, `.cpu()` 等调用都会强制同步

---

## 十七、CUDA 容易翻车的点

1. 只会背 `thread/block/grid`，说不清为什么 block 是协作单位
2. 把 shared memory 说成 " 显存 "
3. 以为 occupancy 越高越好
4. 不知道 warp divergence 为什么慢
5. 不知道为什么 global memory 访问模式会影响性能
6. 说自己会 CUDA 优化，但讲不出实际排查路径
7. 不知道 bank conflict 的检测和解决方法
8. 不知道合并访问的具体条件（连续 + 对齐）
9. 说不清 stream 异步和默认 stream 的隐式同步问题
10. 分不清 L1/shared memory 的硬件关系

---

## 十八、最后的最小口径

### C++

- 我可以按比较深入的层次讲对象生命周期、值类别、移动语义、STL、模板基础、异常安全和并发内存模型。
- 更偏编译器实现、复杂模板库元编程、ABI 细节，我理解概念但不会装成专门做那一块的人。

### CUDA

- 我能讲清 CUDA 的执行模型、内存层次和主要性能瓶颈。
- 我更偏工程使用、训练推理链路和性能直觉，不会把自己包装成专门写高性能 kernel 库的人。
