---
title: "C++ 26 Trivial Relocatable"
categories: [System Programming, C++]
tags: [C++, Relocatable]
---

<!-- prettier-ignore-start -->
> **Trivial relocatable (可平凡重定位)** 概念本身是直观的，但要将其融入具有悠久历史包袱和内在复杂性的 C++ 语言中，则需要非常细致的考量。[**P2786**] [1] 对此进行了大量论述，即便如此也仍需引用额外的提案/论文来尽可能阐明细微之处。
{: .prompt-info }
<!-- prettier-ignore-end -->

<!-- prettier-ignore-start -->
> 本文的定位绝非严谨且全面的论述，而是旨在从语言使用者的角度出发，用尽可能简单的叙述循序渐进地揭示 C++ 引入此概念的必要性。
{: .prompt-warning }
<!-- prettier-ignore-end -->

## C++ 11 移动语义

众所周知，C++ 11 引入了移动语义的概念，这允许对象通过移动构造/赋值转移其内部资源的所有权，这在很多情况下避免了昂贵的 **深拷贝 (Deep Copy)** 开销。

这一特性对于实现高效的 [**RAII**] [2] 资源管理类型很有帮助。例如，当我们定义一个智能指针类来管理动态内存时，RAII 保证了资源能被自动释放的同时兼具异常安全，而移动语义则使其能够低成本地转移其所管理内存的所有权。

一种简单的独占所有权的智能指针实现如下：

```cpp
template <typename T>
class smart_ptr {
  T* p_ = nullptr;

private:
  void move_from(smart_ptr&& other) noexcept {
    p_ = std::exchange(other.p_, nullptr);
  }

  void destroy_self() noexcept {
    if (p_) delete p_;
  }

public:
  explicit smart_ptr(T* p) noexcept : p_{p} {}

  // Move constructor
  smart_ptr(smart_ptr&& other) noexcept {
    move_from(std::move(other));
  }

  // Move assignment operator
  smart_ptr& operator=(smart_ptr&& other) noexcept {
    if (std::addressof(other) != this) {
      destroy_self();
      move_from(std::move(other));
    }
    return *this;
  }

  // Destructor
  ~smart_ptr() {
    destroy_self();
  }

  smart_ptr() noexcept = default;
  smart_ptr(smart_ptr const&) = delete;
  smart_ptr& operator=(smart_ptr const&) = delete;

public:
  // Elide observer & modifier functions ...
};
```

## 但代价是什么呢？

以下代码展示了 `smart_ptr` 对象转移其资源所有权的方式，`std::move` 将 [**值类别 (Value categories)**][3] 为左值的 `ptr1` 和 `ptr2` 转换为亡值，从而匹配到移动构造函数/赋值运算符的调用：

<!-- prettier-ignore-start -->
> `std::move` 是零成本的，它只是纯粹的类型转换。
{: .prompt-info }
<!-- prettier-ignore-end -->

```cpp
smart_ptr ptr1{new int{42}};       // Value initialization
smart_ptr ptr2{std::move(ptr1)};   // Move construction from 'ptr1' to 'ptr2'
smart_ptr ptr3 = std::move(ptr2);  // Move assignment from 'ptr2' to 'ptr3'
```

这里的核心问题是：当 `ptr1` 的资源被“移走”给 `ptr2` 后，`ptr1` 本身变得怎么样了？

答案是，作为一个生命周期与作用域绑定的对象，它仍旧存活，C++ 语言保证它的析构函数最终一定被调用。

这就带来了一个严峻的挑战：如果 `ptr1` 的内部状态 (即 `p_` 指针) 不被重置，那么它的析构操作将会对一个已经被转移了所有权的指针执行 `delete`，这将导致**“二次释放”**的灾难性后果。

因此，移动构造函数“被迫”承担了一项额外的“善后”职责：在取走资源后，必须重置源对象的状态，以确保其未来的析构行为是“无害”的。

<!-- prettier-ignore-start -->
> 这正是 **类不变性 (Class Invariants)** 的一部分。
{: .prompt-info }
<!-- prettier-ignore-end -->

回头看我们的 `move_from` 实现：

```cpp
void move_from(smart_ptr&& other) noexcept {
  p_ = std::exchange(other.p_, nullptr);
}
```

它通过 `std::exchange` 同时完成了“资源转移”和“善后处理”，将 `other.p_` 设置为 `nullptr` 这一步操作的根本目的，就是为了满足 C++ 标准对于 **被移动 (Moved-from)** 对象的规定：它必须处于 [**有效但未指定 (Valid but Unspecified)**] [4] 状态，这里的“有效”，至少隐含了可以被安全地析构这一点。

但也正因为这个“善后行为”的存在，使得移动操作永远无法像我们期望的那样，仅仅是**“把一块内存区域按位拷贝到另一块区域并弃用源区域”**[^1]来的那么纯粹和高效。

<!-- prettier-ignore-start -->
> 你也许会好奇：为什么标准不直接从语言层面禁止访问被移动的对象 (同时省略对析构函数的调用) 呢？答案是，要在编译期跟踪对象是否“被移动”，需要一套严格的生命周期管理系统 (类似 Rust 的所有权模型)。然而，将这样的系统引入 C++ 会彻底破坏其向后兼容性，导致大量现存代码无法编译或行为不正确，这显然是无法接受的。因此，要求对象处于“有效但未指定”状态可以说是一种无奈但务实的选择。
{: .prompt-info }
<!-- prettier-ignore-end -->

<!-- prettier-ignore-start -->
> 然而，这笔“微不足道”的开销，在某些特定场景下，可能并非看上去那么微不足道。
{: .prompt-warning }
<!-- prettier-ignore-end -->

<!-- 它通过 `std::exchange` 做了两件事： -->
<!---->
<!-- 1. 拷贝指针值 -->
<!-- 2. 将源对象的指针设为 nullptr -->
<!---->
<!-- 第二步是关键，因为 C++ 标准规定 **被移动 (Moved from)** 对象必须处于**[有效但未指定状态][2]**，这至少隐含了“重置其内部状态以保证析构函数的正确性”。 -->
<!---->
<!-- 这第二步——回头修改源对象——就是移动的代价。它源于 C++ 标准对“被移动后对象”的规定：必须处于有效但未指定的状态。正是这一规定，让移动操作永远无法像 memcpy 那样，只是一次纯粹的、单向的内存位块搬迁。这个为了保证语言安全而引入的“善后”开销，也成为了更高性能追求之路上的一个障碍。 -->
<!---->

[^1]: 没错就是单纯的 `memcpy`/`memmove`

[1]: https://wg21.link/p2786r13
[2]: https://en.cppreference.com/w/cpp/language/raii.html
[3]: https://en.cppreference.com/w/cpp/language/value_category.html
[4]: https://en.cppreference.com/w/cpp/utility/move.html
