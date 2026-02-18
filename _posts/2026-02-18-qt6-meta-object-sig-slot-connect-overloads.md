---
title: "[Work in Progress] Qt 6 中的元对象系统 —— 信号槽番外（connect 的兄弟们）"
categories: [System Programming, C++]
tags: [C++, Qt, MOC]
mermaid: true
---

> **为什么要写这个番外篇**
>
> 这其实是一个老生常谈的话题，毕竟 `QObject::connect` 贯穿 Qt 应用始终，无论初学者还是高手都无时不刻不在使用它，但我觉得对于系列文章而言，它仍是不可缺少的一块拼图，也能从中窥得 MOC 系统的演进，同时也是梳理不同重载的使用场景，规避一些性能和安全性陷阱（没错说的就是你，**三参数 Lambda 重载**）。
{: .prompt-info }

---

## 一、前言

在深入研究 `QObject::connect` 那个复杂的实现源码之前，我们需要先停下来，整理一下它的外在表现。

如果翻看 "qobject.h" 头文件，可能会被那一堆 `connect` 的重载弄得眼花缭乱，归根结底，它们代表了 Qt 在不同 C++ 时代的演进逻辑。我们可以将它们分为四大流派，分别基于：
  - 字符串
  - 函数指针
  - 函数对象（Functor / Lambda）
  - 元对象（字符串本质基于此）

接下来我们将分别认识它们。

---

## 二、基于字符串的重载

这是最古老（Qt 4 时代）、最动态，同时也是开销最大的一种连接方式，它的函数签名如下：

```cpp
// 用法：connect(sender, SIGNAL(valueChanged(int)), receiver, SLOT(setValue(int)))
static QMetaObject::Connection
connect(const QObject *sender, const char *signal,
        const QObject *receiver, const char *method,
        Qt::ConnectionType type = Qt::AutoConnection);
```

---

### 1. SIGNAL 和 SLOT 宏的真面目

在进入函数体之前，我们先看传入的参数 `signal` 和 `method` 分别是什么。

在 "qobjectdefs.h" 中，`SIGNAL` 和 `SLOT` 宏分别被[定义为](https://github.com/qt/qtbase/blob/000d6c62f7880bb8d3054724e8da0b8ae244130e/src/corelib/kernel/qobjectdefs.h#L58)：

```cpp
...
# define QSLOT_CODE                 1
# define QSIGNAL_CODE               2
# define QT_PREFIX_CODE(code, a)    QT_STRINGIFY(code) #a
# define QT_STRINGIFY_SLOT(a)       QT_PREFIX_CODE(QSLOT_CODE, a)
# define QT_STRINGIFY_SIGNAL(a)     QT_PREFIX_CODE(QSIGNAL_CODE, a)
# define SLOT(a)                    QT_STRINGIFY_SLOT(a)
# define SIGNAL(a)                  QT_STRINGIFY_SIGNAL(a)
...
```

这意味着当写下 `SIGNAL(valueChanged(int))` 时，预处理器把它变成了字符串 `"2valueChanged(int)"`，同理，`SLOT(setValue(int))` 变成了 `"1setValue(int)"`。

字符串的第一个字符（0, 1, 2）标识了它的身份（Method, Signal, Slot）。

---

### 2. connect 函数内部的身份核验

在进入 `connect` 函数后，首先做的就是[检查这个前缀](https://github.com/qt/qtbase/blob/000d6c62f7880bb8d3054724e8da0b8ae244130e/src/corelib/kernel/qobject.cpp#L3011)：

```cpp
// 检查 `signal` 参数是否是用 `SIGNAL()` 宏包裹的（即首字符必须是 '2'）
if (!check_signal_macro(sender, signal, "connect", "bind"))
    return ...;

// 提取 `method` 的首字符（'1' 表示槽，'2' 表示信号）
int membcode = extract_code(method);

// 跳过首字符，指向真正的函数签名（如 "valueChanged(int)"）
++signal;
++method;
```

这一步保证了不能将一个 `SLOT` 宏误作 `signal` 参数传递。

---

### 3. 查表确定 Signal index

这是最耗时的一步，拿到 `"valueChanged(int)"` 这个字符串后，需要知道它在 `sender` 的元数据表中是第几个信号：

```cpp
// 解析签名，分离出函数名 "valueChanged" 和参数类型列表
QByteArrayView signalName = QMetaObjectPrivate::decodeMethodSignature(signal, signalTypes);

const QMetaObject *smeta = sender->metaObject();

// 去 MOC 生成的数据表里查找
int signal_index = QMetaObjectPrivate::indexOfSignalRelative(
        &smeta, signalName, signalTypes.size(), signalTypes.constData());
```

这里的 `indexOfSignalRelative` 会遍历我们在[上一篇]({% post_url 2026-02-17-qt6-meta-object-sig-slot-1-moc %})看到的 `qt_stringData` 和 `qt_methods` 数组，逐个比对字符串，这是一个 O(N) 的线性查找过程。

> **关于 Normalization（规范化）**
>
> 代码中有一段 `if (signal_index < 0)` 的判断，如果手滑写了 `SIGNAL(valueChanged( int ))`，或者 `const int&` 写成了 `int`，第一次查找会失败。
>
> 但实现会调用 `normalizedSignature` 将其格式化为标准形式（去除多余空格等），然后再查找一次，这既是容错机制，也是字符串连接方式较慢的潜在因素之一。
{: .prompt-tip }

---

### 4. 查表确定 Method index

有了 Signal index，还需要找到 Receiver 侧的 index。

```cpp
switch (membcode) {
case QSLOT_CODE:
    // 如果目标是槽，则查找槽的索引
    method_index_relative = QMetaObjectPrivate::indexOfSlotRelative(...);
    break;
case QSIGNAL_CODE:
    // 如果目标也是信号（即信号连接信号），则查找信号的索引
    method_index_relative = QMetaObjectPrivate::indexOfSignalRelative(...);
    break;
}
```

这里解释了为什么 `connect` 可以连接两个信号，只要去查信号表而不是槽表就可以了。

---

### 5. 参数兼容性检查

找到两个 index 后，实现必须确保这次连接是安全的：

```cpp
if (!QMetaObjectPrivate::checkConnectArgs(signalTypes.size(), signalTypes.constData(),
                                          methodTypes.size(), methodTypes.constData())) {
    qCWarning(lcConnect, "QObject::connect: Incompatible sender/receiver arguments ...");
    return QMetaObject::Connection(nullptr);
}
```

这个函数会检查：
  - 信号的参数数量 >= 槽函数的参数数量
  - 信号和槽函数的所有参数类型相匹配

---

### 6. 交付底层

当一切检查通过，我们拿到了：
  - sender 指针
  - `signal_index`
  - receiver 指针
  - `method_index_relative`

最后一行代码将这些处理好的参数交给真正的幕后函数：

```cpp
QMetaObject::Connection handle = QMetaObject::Connection(
    QMetaObjectPrivate::connect(sender, signal_index, smeta, receiver, method_index_relative, rmeta, type, types)
);
```

`QMetaObjectPrivate::connect` 负责加锁、操作链表、处理线程安全性，这部分逻辑我们将在后续文章中详细拆解。

---

### 7. 小结

通过对源码的简单分析，我们可以清楚认识到字符串版本 `connect` 的特点：
  1. 开销相对大：每次 connect 都要解析字符串、分配内存（QByteArray）、进行字符串匹配查找。
  2. 编译期不安全：参数不匹配要等到运行时才报错。
  3. 动态灵活性：它不需要知道类的具体定义，只要有 `QObject` 指针和元数据即可，非常适合脚本绑定和插件系统。















