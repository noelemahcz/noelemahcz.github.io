---
title: "llvm::formatv 源码解析"
categories: [System Programming, C++]
tags: [C++, LLVM]
mermaid: true
---

## 一、动机

最近我在开发 cxx-decorator 项目，该项目基于 ClangAST 和 libtooling 实现类似 Python 语言中的装饰器语法特性。

在开发期间，由于调试输出和日志记录等需求，需要频繁对 `SmallVector`、`StringRef`、`Decl*` 等 LLVM / ClangAST 核心类型进行格式化，原本我计划为这些类型实现 C++20 标准格式化支持（即 `std::format`），不过在分析设计如何将其与 LLVM 的 `raw_ostream` 体系合理集成时，我发现原来 LLVM 已经提供了一个功能完备的 `formatv` 格式化函数。

出于对 LLVM 如何实现类型安全格式化的好奇，也为了探究其与 `std::format` 在底层设计上的异同，我决定深入剖析其源码实现。

---

## 二、使用方式

在剖析实现之前，有必要简单介绍一下 `llvm::formatv` 的使用方式，总体而言与 `std::format` 非常相似，差异主要集中在在类型特定的格式符上。

### 基础示例

```cpp
// 位置索引
formatv("{0} {1}", "a", "bb")      // 输出："a bb"
// 参数复用
formatv("{0} {1} {0}", "a", "bb")  // 输出："a bb a"
// 布局控制（右对齐，宽度为 5，默认填充空格）
formatv("{0, +5}", "a")            // 输出："    a"
// 布局控制（居中，宽度为 5，填充字符'-'）
formatv("{0,-=5}", "a")            // 输出："--a--"
```

### 格式化字符串语法

`llvm::formatv` 使用包含替换序列（Replacement Sequence）的格式字符串，其核心语法结构如下：

```text
{ index [, layout] [: format] }
```
{: .nolineno }

1. 核心参数：
  - `index`**（必填）**：非负整数，对应参数包中参数的索引位置，同一索引可以在格式字符串中多次引用（如 `"{0} {1} {0}"`）。
  - `layout`（选填）：语法格式为 `[[char]loc]width`，用于控制输出内容的对齐、宽度和填充。
  - `format`（选填）：类型特定的格式化选项。

  > 与 `std::format` 和 `std::vformat` 不同，`llvm::formatv` 必须指定 `index`，否则行为未定义。
  {: .prompt-warning }

2. 布局（`layout`）用于控制字段在可用空间内的展示方式，只有指定了 `width`，对齐和填充才会生效：
  - `width`：字段宽度。正整数，如果内容长度小于此值，将根据 `loc` 和 `char` 进行填充，否则直接输出内容，不进行填充。
  - `loc`：对齐位置。`-` 左对齐，`=` 居中对齐，`+` 右对齐。
  - `char`：填充字符。默认为空格。

{% raw %}
3. 转义字符：`{` 和 `}` 是保留字符，如果要在输出中包含大括号，必须进行转义，使用双大括号 `{{` 来输出一个 `{` 字面量。
{% endraw %}

### 内置格式化选项

`llvm::formatv` 的格式说明符（即 `{index:format}` 中的 `format` 部分）根据目标类型遵循不同的文法。

#### 1. 整数类型（Integral）

整数的格式字符串文法为：**`[style][digits]`**

- `style`：决定输出格式。如十六进制、千分位等，见下表。
- `digits`：正整数（`0-99`）。对于十六进制（`x/X`），表示最小输出位数，如果数值长度不足，会在左侧补 `0`，对于其他样式，该参数被忽略。

| 样式（`style`） | 说明                        | 输入     | 格式串 | 输出      |
| :-------------- | :-------------------------- | :------- | :----- | :-------- |
| **`D` / `d`**   | 十进制（默认）              | `100000` | `D`    | `100000`  |
| **`N` / `n`**   | 千分位分隔                  | `123456` | `N`    | `123,456` |
| **`x+` / `x`**  | 十六进制（小写，`0x` 前缀） | `42`     | `x`    | `0x2a`    |
| **`X+` / `X`**  | 十六进制（大写，`0x` 前缀） | `42`     | `X+4`  | `0x002A`  |
| **`x-`**        | 十六进制（小写，无前缀）    | `42`     | `x-`   | `2a`      |
| **`X-`**        | 十六进制（大写，无前缀）    | `42`     | `X-`   | `2A`      |

#### 2. 浮点类型（Floating Point）

浮点数的格式字符串文法为：**`[style][precision]`**

- `style`：决定浮点数的表现形式。小数、百分比、科学计数，见下表。
- `precision`：正整数（`0-99`）。指定小数点后的位数，如果省略，科学计数法（`E/e`）默认为 6 位，其他（`F/P`）默认为 2 位。

| 样式（`style`） | 说明             | 输入    | 格式串 | 输出                   |
| :-------------- | :--------------- | :------ | :----- | :--------------------- |
| **`F` / `f`**   | 小数（默认）     | `1.0`   | `F`    | `1.00`（默认2位）      |
| **`P` / `p`**   | 百分比           | `0.05`  | `P`    | `5.00%`                |
| **`E`**         | 科学计数（大写） | `12345` | `E`    | `1.234500E+04`         |
| **`e`**         | 科学计数（小写） | `12345` | `e3`   | `1.235e+04`（指定3位） |

#### 3. 布尔类型（Boolean）

默认样式为 **`t`**，即：`true / false`

| 样式（`style`） | 说明                 | `true` 输出 | `false` 输出 |
| :-------------- | :------------------- | :---------- | :----------- |
| **`t`**         | true / false（默认） | `true`      | `false`      |
| **`T`**         | TRUE / FALSE         | `TRUE`      | `FALSE`      |
| **`y`**         | yes / no             | `yes`       | `no`         |
| **`Y`**         | YES / NO             | `YES`       | `NO`         |
| **`D` / `d`**   | 0 / 1                | `1`         | `0`          |

#### 4. 指针类型（Pointer）

指针主要按十六进制地址输出，文法结构类似整数。

| 样式（`style`） | 说明                     | 示例输出                    |
| :-------------- | :----------------------- | :-------------------------- |
| **`x` / `X`**   | 十六进制（带前缀，默认） | `0xdeadbeef` / `0xDEADBEEF` |
| **`x-` / `X-`** | 十六进制（无前缀）       | `deadbeef` / `DEADBEEF`     |

#### 5. 范围与容器（Ranges / Containers）

将任意 Ranges 打印为由 `separator` 分隔、以 `element_style` 指定的格式显示的元素序列，其 BNF 文法如下：

```haskell
range_style     ::= [separator] [element_style]
separator       ::= "$" delimeted_expr
element_style   ::= "@" delimeted_expr
delimeted_expr  ::= "[" expr "]" | "(" expr ")" | "<" expr ">"
expr            ::= <any string not containing delimeter>
```

- 分隔符（**`$`**）：插入在容器内相邻两个元素之间的字符串，未指定时默认以空格分隔。
- 元素样式（**`@`**）：指定容器内元素的格式说明符（即上述的整数、浮点等规则），未指定时以默认格式显示。

| 场景              | 格式串               | 说明                            | 示例输出 |
| :---------------- | :------------------- | :------------------------------ | :------- |
| 默认              | **`{0}`**            | 默认空格分隔，默认元素格式      | `255 10` |
| 分隔符            | **`{0:$[,]}`**       | 以逗号分隔                      | `255,10` |
| 元素格式          | **`{0:@[x-2]}`**     | 以小写十六进制并补齐到 2 位显示 | `ff 0a`  |
| 分隔符 + 元素格式 | **`{0:$[, ]@[X-]}`** | 以 `, ` 分隔，大写十六进制显示  | `FF, A`  |

---

## 三、实现解析

LLVM 的 `formatv` 实现可以分为三个主要阶段：编译期参数适配、对象构建与存储、运行时格式化解析与输出。

> 以下所有代码片段已适当简化，在保持逻辑等价的前提下提高可读性，以便于读者理解。
{: .prompt-info }

---

### 1. 编译期参数适配

`formatv` 本身的实现非常简单，只是依次对每个待格式化参数进行包装，再将包装结果以 tuple 形式连同 `Fmt`（格式串）一起打包进 `formatv_object` 中返回：

```cpp
template <typename... Ts>
auto formatv(const char *Fmt, Ts&&... Vals) {
  return formatv_object{
    Fmt,
    make_tuple(build_format_adapter(forward<Ts>(Vals))...)
  };
}
```

---

包装逻辑都集中在 `build_format_adapter` 中，这是一个重载的函数模板，利用 SFINAE 特性对于不同类型应用不同的格式化适配器：

> 我将其翻译成了等价的 `constexpr if` + `concepts` 实现以提高可读性。
{: .prompt-info }

```cpp
template <typename T>
auto build_format_adapter(T&& Item)
{
  if constexpr (is_base_of_v<format_adapter, remove_reference_t<T>>) {
    return Item;

  } else if (HasFormatProvider<T>) {
    return provider_format_adapter<T>{forward<T>(Item)};

  } else if (HasStreamOperator<T>) {
    static_assert(
      !is_same_v<Error, remove_cv_t<T>>,
      "llvm::Error-by-value must be wrapped in fmt_consume() for formatv");
    return stream_operator_format_adapter<T>{forward<T>(Item)};

  } else {
    return missing_format_adapter<T>{};
  }
}

template <typename T>
concept HasFormatProvider =
  requires(decay_t<T> const& val, raw_ostream& stream, StringRef options)
{
  { format_provider<decay_t<T>>::format(val, stream, options) } -> same_as<void>;
};

template <typename T>
concept HasStreamOperator =
  requires(raw_ostream& os, decay_t<T> const& val)
{
  { os << val } -> same_as<raw_ostream&>;
};
```

包装逻辑有四种，优先级从高到低：

  1. 如果 `T` 继承自 `format_adapter`，则不进行任何包装，直接返回 `Item` 本身。

  2. 如果存在 `format_provider<T>` 类模板特化，且该特化定义了以 `(T const&, raw_ostream&, StringRef)` 为参数的名为 `format` 的成员函数，则使用 `provider_format_adapter` 进行包装。

  3. 如果能通过 ADL 查找到适用于 `T` 的 `operator<<` 重载，且该重载接受 `(raw_ostream&, T const&)` 并返回 `raw_ostream&`，则使用 `stream_operator_format_adapter` 进行包装。

  4. 否则使用 `missing_format_adapter` 进行包装。

> **关于 `llvm::Error` 的特殊保护**
>
> `llvm::Error` 具有严格的“强制检查”语义，要求错误对象在析构前必须被显式消费（Checked / Consumed），否则会导致程序崩溃。
>
> 当 `Error` 对象以**右值**传递给 `formatv` 时（例如 `formatv("Result: {0}", fallible_call())`），`stream_operator_format_adapter` 会接管其所有权，由于通用的流适配器仅执行打印操作而不消费错误对象，当适配器销毁时其内部持有的 `Error` 对象析构会触发断言。
>
> 因此 `formatv` 禁止右值 `Error` 对象的直接格式化，强制要求通过 `fmt_consume()` 进行包装（如 `formatv("{0}", fmt_consume(err))`），该函数底层使用 `ErrorAdapter`，在打印错误信息后会正确消费错误对象。按**左值**传递仅涉及引用访问，不发生所有权转移，故不受此限制。
{: .prompt-warning }

> 如果要为自定义类型或第三方类型实现 `vformat` 支持，应首选特化 `format_provider` 的方式，在全局范围内定义 `T` 的默认格式化行为。
>
> 由于 `raw_ostream` 的流操作符接口不支持传递额外状态，所以 `stream_operator_format_adapter` 适配器不支持 `options` 参数，故不作为首选方式。
{: .prompt-info }

`format_adapter` 是一个多态基类，提供了 `format` 纯虚函数，`provider_format_adapter` 和 `stream_operator_format_adapter` 都继承自 `format_adapter`，它们重写的 `format` 将实际操作转发给 `format_provider<T>::format()` 和 `operator<<`，这使得所有满足条件的类型 `T` 经包装后都具备了统一的接口：

```cpp
struct format_adapter {
  virtual ~format_adapter() = default;
  virtual void format(raw_ostream& S, StringRef Options) = 0;
};

template <typename T>
struct provider_format_adapter : format_adapter {
  T Item;
  void format(raw_ostream& S, StringRef Options) override {
    format_provider<decay_t<T>>::format(Item, S, Options);
  }
};

template <typename T>
struct stream_operator_format_adapter : format_adapter {
  T Item;
  void format(raw_ostream& S, StringRef) override { S << Item; }
};

template <typename T> class missing_format_adapter;
```

当类型 `T` 不满足前三种情况时就会实例化 `missing_format_adapter`，这是一个**不完整类型**，对其进行实例化会触发编译期报错（类型安全），并且它的类型名本身就是清晰的诊断信息。

> 特化 `format_provider` 的方式也存在一定局限性，即无法覆盖已受支持类型的默认格式化行为。对于这种情况，应通过继承 `llvm::FormatAdapter<T>`（`format_adapter` 的浅包装类模板）并重写 `format` 虚函数来创建一个自定义格式化适配器，在调用 `formatv` 时不直接传递 `T` 类型对象，而是传递适配后的对象（例如 `formatv("{0}", format_int_custom{42}`)），这种方式也适用于在特定上下文中临时修改类型的默认格式化行为，为这套格式化机制提供了灵活性。
{: .prompt-info }

> **FIXME**: 悬垂引用陷阱
{: .prompt-warning }

---

### 2. 对象构建与内联存储

从上一阶段构建的 `std::tuple` 对象初始化一个轻量级的 `formatv_object` 中间对象，并不真正执行格式化。

```cpp
class formatv_object_base {
protected:
  StringRef Fmt;
  ArrayRef<format_adapter*> Adapters;

  formatv_object_base(StringRef Fmt, ArrayRef<format_adapter*> Adapters)
      : Fmt(Fmt), Adapters(Adapters) {}

  // ...
};

template <typename Tuple>
class formatv_object : public formatv_object_base {
  Tuple Parameters;
  array<format_adapter*, tuple_size_v<Tuple>> ParameterPointers;

public:
  formatv_object(StringRef Fmt, Tuple&& Params)
      : formatv_object_base(Fmt, ParameterPointers),
        Parameters(move(Params)) {
    ParameterPointers = apply([](Ts&... xs) { return {{&xs...}}; }, Parameters);
  }

  // ...
};
```

LLVM 利用 `formatv_object` 及其基类 `formatv_object_base`，在不进行堆分配的前提下，实现了类型安全的变参存储和运行时随机访问：

- 存储层（`formatv_object`）
    - 本身存储在栈上，通过 tuple 成员内联存储所有适配器对象（持有所有权），避免了堆分配带来的额外性能开销。
    - 通过多态基类指针将异构元组转换为同构数组，这是后续解析阶段能够处理 `{1} {0}` 这种乱序占位符的前提。

- 逻辑层（`formatv_object_base`）
    - 采用模板类继承非模板基类的设计，将格式化逻辑分离进基类中，最大限度地减轻了模板实例化导致的代码膨胀问题（加快编译速度的同时也能提高指令缓存的命中率）。

由于 `std::tuple` 仅支持编译期索引访问（通过 `std::get<N>`），而 `formatv` 的格式化字符串在运行期解析，这就需要一种通过运行期索引访问适配器参数的能力。

具体而言，LLVM 利用 `std::apply` 将 tuple 展开，依次获取每个适配器元素的地址，并存储进类型为 `std::array<format_adapter*>` 的基类指针数组，建立了从“编译期元组索引”到“运行期数组索引”的映射，由此实现运行期随机访问适配器对象。

> `formatv` 函数最终返回的是 `formatv_object` 对象，此对象承载了格式化所需的所有上下文信息，但尚未执行任何字符串解析和格式化操作，因此开销极小。
>
> 只有当用户显式调用 `.str()` / `operator std::string()` 或将其通过流操作符传输给 `raw_ostream` 时，才会调用 `format` 成员函数真正开始解析，这种惰性解析和格式化机制可以避免不必要的字符串拼接和内存分配开销。
{: .prompt-info }










<!-- 1. 泛型存储容器 `formatv_object<Tuple>`： -->
<!--   - 使用 `std::tuple<Adapters...>` 作为底层存储，保留了适配器的具体类型 -->
<!--   - 拥有所有适配器对象的所有权 -->
<!--   - 不进行任何堆分配 -->
<!---->
<!-- 2. 运行时索引与类型擦除 -->
<!---->
<!--   `std::tuple` 仅支持编译期索引访问（通过 `std::get<N>`），而 `formatv` 的格式化字符串是在运行期解析，因此需要引入类型擦除机制来实现参数的随机访问。 -->
<!---->
<!--   `formatv_object` 内部（通过继承）维护了一个 `std::array<format_adapter*, N>` 类型的数组，在构造函数中利用 `std::apply` 展开 tuple，将每个适配器的地址存入该数组。 -->
<!---->
<!--   这是一种典型的**非拥有（Non-owning）**转换，同时也发生了类型擦除：tuple 实际拥有适配器对象，array 仅持有指向 tuple 内元素的基类指针，通过此 array 进行随机访问。 -->






---

```mermaid
flowchart TD
  Start(["调用 formatv(Fmt, Vals...)"])

  subgraph SFINAE [" "]
    Start --> BuildAdapter
    BuildAdapter["利用 SFINAE 萃取类型特征，为参数构建多态适配器，对于每个参数项 T item"]

    BuildAdapter -- "T 直接继承自 format_adapter" --> UseMember
    UseMember("不进行任何包装")
    UseMember --> BuildTuple

    BuildAdapter -- "存在特化 provider_format_adapter&lt;T&gt;" --> UseProvider
    UseProvider("包装为 provider_format_adapter(item)")
    UseProvider --> BuildTuple

    BuildAdapter -- "满足表达式<br/>raw_ostream() << arg<br/>（通过 ADL）" --> UseStream
    UseStream("包装为 stream_operator_format_adapter(item)")
    UseStream --> BuildTuple

    BuildTuple["使用 std::tuple&lt;Adapters...&gt; 存储所有适配器（未擦除类型）"]
  end

  subgraph FormatvObject [" "]
    BuildTuple -- "从 tuple 构建 formatv_object" --> Object
    Object["formatv_object&lt;tuple&lt;Adapters...&gt;&gt; 作为 storage（拥有所有适配器的所有权）"]

    Object -- "初始化非多态基类 formatv_object_base" --> BaseObject
    BaseObject["formatv_object_base 使用 ArrayRef&lt;format_adapter*&gt; 存储 tuple 中所有适配器的基类指针，通过虚函数和类型擦除实现高效的参数乱序访问支持"]

    BaseObject --> ReturnObject
    ReturnObject["返回 formatv_object 对象（惰性格式化，尚未进行任何解析或格式化）"]
  end

  subgraph Formatting [" "]
    ReturnObject -- "formatv_object.format() / .str() / operator<< 触发解析和格式化" --> ParseFmt
    ParseFmt["调用 parseFormatString 解析格式字符串"]

    ParseFmt --> Iterate
    Iterate{"遍历 ReplacementItem"}

    Iterate -- "Type == Literal" --> PrintLit
    PrintLit["直接将文本输出到 Stream"]
    PrintLit --> Iterate

    Iterate -- "Type == Format {index, options}" --> GetAdapter
    GetAdapter["通过 index 从指针数组获取适配器基类指针"]
    CallVirtual["调用虚函数 adapter->format(Stream, Options)"]
    AlignPad["处理对齐与填充"]
    GetAdapter --> AlignPad --> CallVirtual --> Iterate
  end

  Iterate -- "遍历完成" --> End
  End([最终结果])
```
