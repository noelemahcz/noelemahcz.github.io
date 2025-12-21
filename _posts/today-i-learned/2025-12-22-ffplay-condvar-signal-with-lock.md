---
title: "[TIL] ffplay 中条件变量 Signal 操作加锁的原因"
categories: [System Programming, C++]
tags: [TIL, C++, FFmpeg, ffplay]
---

<!-- --- -->
<!---->
<!-- ## 起因 -->
<!---->
<!-- 最近在用 FFmpeg 实现一个视频播放器，截至今天已经基本实现了以下目标： -->
<!-- 1. 将 FFmpeg 中的常用类型（如 `AVFrame`）和接口封装成了 Modern C++ 模块，便于简化开发流程和内存管理。 -->
<!-- 2. 基于 C++20 `std::atomic` 从零构建了一个单生产者单消费者无锁队列（`SPSCQueue<T>`）类型，利用 [Ring Buffer]() 和预分配机制实现了对象复用，配合 (1) 中所述的 RAII 包装实现内存和资源的自动化管理，并且提供了阻塞（使用 `std::atomic::wait/notify` 保持无锁架构的同时避免长时间自旋占满 CPU 资源）和非阻塞（`try_peek` / `try_consume` ...）两种接口用于不同的场景，此队列主要用于在线程间高效地传输解码和转码结果，也可用于其它 SPSC 场景。 -->
<!-- 3. 实现了三线程架构的纯 CPU 驱动的视频播放流程： -->
<!--   - 解码线程：调用 `av_read_frame` / `av_send_packet` / `av_receive_frame` 解码出 `AVFrame`，通过 `SPSCQueue<AVFrame>` 投递给转码线程，这里利用了队列的对象池配合 `av_frame_move_ref` 转移 `AVFrame` 的数据所有权，避免频繁且重复的内存分配和释放。 -->
<!--   - 转码线程：接收 `AVFrame` 并通过 `sws_scale` 转换编码格式，如 `YUV420P` -> `RGB888`，并将结果通过 `SPSCQueue<QImage>` 投递给渲染线程，这里同样利用队列对象池配合 `QImage` 内部缓冲区复用避免了重复内存分配。 -->
<!--   - 渲染线程：不使用 `QApplication::exec`，而是通过手写循环调用 `QApplication::processEvents` 精确控制视频帧的时序同步，使用 `QWidget::paintEvent` 和 `QPainter` 进行绘制。 -->
<!---->
<!-- 经测试已完美支持 2560x1600 60FPS 视频的流畅播放，当然这只是为了学习软解和 CPU 渲染流程，商业级播放器是不可能考虑 CPU 转码和渲染的，后续增加 OpenGL / DX11 支持是必然的。 -->
<!---->
<!-- --- -->
<!---->
<!-- ## ffplay -->
<!---->
<!-- 其实在开发这个播放器 Demo 前，我就已经知道并且使用过 ffplay，它可说是无数商业级播放器的雏形和参考对象，但我故意没有提前学习它的源码实现，为的是看看只凭自己的理解和思考能实现到什么程度，然后再学习 ffplay 的实现，对比处理方法和思路的差异。 -->
<!---->
<!-- 首先我观察到的一点就是 `AVPacket` / `AVFrame` 队列的实现异同，ffplay 同样采用了 Ring Buffer 配合对象复用机制，这与我的思路和设计相同，但它的同步机制是更传统的互斥锁和条件变量模式，并且很多共享变量都没有使用 `atomic`，这可能是由于 ffplay 的历史较久，并且它工作良好久经考验，没有充分的必要进行修改。 -->

---

## 起因

今天分析了 ffplay 的退出流程，无论哪种触发方式，最终都派发到 [`do_exit`](#ref1) 函数，其内部调用 [`stream_component_close`](#ref2) 处理音频、视频、字幕流的关闭，该函数又调用 [`decoder_abort`](#ref3) 通知对应的解码线程中止：

```cpp
static void decoder_abort(Decoder *d, FrameQueue *fq)
{
    packet_queue_abort(d->queue);
    frame_queue_signal(fq);
    SDL_WaitThread(d->decoder_tid, NULL);
    d->decoder_tid = NULL;
    packet_queue_flush(d->queue);
}

static void frame_queue_signal(FrameQueue *f)
{
    SDL_LockMutex(f->mutex);
    SDL_CondSignal(f->cond);
    SDL_UnlockMutex(f->mutex);
}
```

ffplay 这里对条件变量的 Signal 操作使用了互斥锁进行单独保护，一般而言，我们只会使用互斥锁保护共享数据和“唤醒条件”本身，Signal 操作通常是在解锁后进行的，这是因为持锁 Signal 可能导致等待线程被唤醒后无法获取锁而立刻再次陷入休眠，产生不必要的额外上下文切换开销。

> 很多实现都采用了 [Wait Morphing](https://github.com/cloudius-systems/osv/issues/24) 优化，避免了这个问题
{: .prompt-tip }

---

## 设计使然，必须加锁

然而，在 ffplay 的设计下，`frame_queue_signal` 加锁是必须的。

```cpp
static Frame *frame_queue_peek_readable(FrameQueue *f)
{
    /* wait until we have a readable a new frame */
    SDL_LockMutex(f->mutex);
    while (f->size - f->rindex_shown <= 0 &&
           !f->pktq->abort_request) {
        SDL_CondWait(f->cond, f->mutex);
    }
    SDL_UnlockMutex(f->mutex);

    if (f->pktq->abort_request)
        return NULL;

    return &f->queue[(f->rindex + f->rindex_shown) % f->max_size];
}
```

解码线程会循环调用 `frame_queue_peek_writable` 持续提供解码后的画面帧，而该函数又通过检查 `PacketQueue` 上的 `abort_request` 标志位获取中断通知。

#### **不加锁会有什么问题**

这个中断标志位的修改操作只受到 `PacketQueue` 的互斥锁保护，而不受 `FrameQueue` 的互斥锁保护：

```cpp
static void packet_queue_abort(PacketQueue *q)
{
    SDL_LockMutex(q->mutex);
    q->abort_request = 1;
    SDL_CondSignal(q->cond);
    SDL_UnlockMutex(q->mutex);
}
```

如果 Signal 不加锁，可能会发生丢失唤醒信号的情况，比如以下时序：
1. 解码线程：持有 `f->mutex`，检查 `f->pktq->abort_request == 0`（不中止），此时，在调用 `SDL_CondWait` **之前**发生了上下文切换，解码线程被系统挂起。
2. 控制线程：设置 `abort_request = 1`，不加锁直接调用 `SDL_CondSignal`，此时解码线程还未进入等待，信号丢失。
3. 解码线程：被系统恢复执行，调用 `SDL_CondWait(f->cond, f->mutex)` 释放 `f->mutex` 陷入睡眠，然后它就永远睡死了，这个后果是致命的。

#### **加锁如何解决问题**

ffplay 在 `frame_queue_signal` 中加上了 `SDL_LockMutex(f->mutex)`，整个流程就会强制**串行化**：
1. 解码线程：持有 `f->mutex`，检查条件，准备 `wait`。
2. 控制线程：设置 `abort_request = 1`，调用 `frame_queue_signal`，尝试 `SDL_LockMutex(f->mutex)` 被阻塞，因为解码线程持有该锁。
3. 解码线程：调用 `SDL_CondWait`，该函数内部会原子地执行“释放锁 + 进入等待队列”的操作，这意味着一旦锁被释放，解码线程就已经处于等待状态了。
4. 控制线程：终于成功获取到 `f->mutex`，调用 `SDL_CondSignal`，然后释放锁。
5. 解码线程：收到唤醒信号，被成功唤醒。

#### **为什么这样设计**

`PacketQueue` 和 `FrameQueue` 在逻辑上共享同一个中断标志 `pktq->abort_request`，这是一种依赖关系的体现，`PacketQueue` 是数据的上游（Source），`FrameQueue` 是数据的下游（Destination）。

如果上游（`PacketQueue`）已经终止，解码器也会停止工作，意味着不会再有新的数据包投送，那么下游（`FrameQueue`）继续等待或工作就没有意义了，因此，将 `FrameQueue` 的生命周期状态直接绑定在 `PacketQueue` 上是很合理的。

---

## 内存可见性保证

除了防止信号丢失，`frame_queue_peek_writable` 加锁还有一个作用，即保证控制线程对 `pktq->abort_request` 的修改对解码线程可见。

ffplay 这里的设计是：
- 控制线程：`abort_request` 的修改在 `PacketQueue` 的锁 `pktq->mutex` 保护下进行。
- 解码线程：`abort_request` 的读取在 `FrameQueue` 的锁 `f->mutex` 保护下进行。

问题在于，控制线程修改了 `abort_request`，但没有触碰 `f->mutex`，然后直接 Signal 唤醒解码线程，后者尝试获取 `f->mutex`，根据 C/C++ 内存模型的定义，解码线程不保证能立即看到控制线程对 `abort_request` 的修改。

而互斥锁的作用除了串行化（排他访问），还负责提供内存屏障，互斥锁的 Unlock 操作带有 `mo_release` 语义，Lock 操作带有 `mo_acquire` 语义，同一个互斥锁上的这两个操作结合构成了 [Happends-before](#ref4) 关系，提供了对内存修改的可见性保证。

如果没有加锁，即使控制线程设置了 `abort_request = 1`，解码线程被唤醒后读取到的值可能仍然是 `0`，使得它误以为不需要终止，再次进入等待，从而导致线程睡死或延迟终止。

---

## 总结

为了防止唤醒信号丢失，互斥锁必须至少覆盖“修改条件状态”或“发送唤醒信号”其中之一，以强制保证信号发送永远发生在等待线程入睡之后。

---

## References

1. <span id="ref1">[[ffplay.c] `do_exit`](https://github.com/FFmpeg/FFmpeg/blob/894da5ca7d742e4429ffb2af534fcda0103ef593/fftools/ffplay.c#L1301)</span>
2. <span id="ref2">[[ffplay.c] `stream_component_close`](https://github.com/FFmpeg/FFmpeg/blob/894da5ca7d742e4429ffb2af534fcda0103ef593/fftools/ffplay.c#L1207)</span>
3. <span id="ref3">[[ffplay.c] `decoder_abort`](https://github.com/FFmpeg/FFmpeg/blob/894da5ca7d742e4429ffb2af534fcda0103ef593/fftools/ffplay.c#L816)</span>
4. <span id="ref4">[[cppreference] Happends-before](https://en.cppreference.com/w/cpp/atomic/memory_order.html)</span>
