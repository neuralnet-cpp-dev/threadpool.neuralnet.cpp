# threadpool.neuralnet.cpp

neuralnet.cpp 生态的**独立线程池 + 并行算法**库（header-only）。
领域无关，可被任何 C++ 项目复用。

## 内容

| 头文件 | 内容 |
|---|---|
| `core_threadpool.hpp` | `ThreadPool` / `global_thread_pool()` —— 任务队列 + 工作线程 |
| `core_parallel.hpp` | 顶层并行算法：`nn::for_each` / `transform` / `transform_reduce` / `parallel_for_blocks` / `parallel_for_samples`，按 `PARALLEL_THRESHOLD`（524288 元素，来自 base 仓 `core_config.hpp`）自动串行/并行回落 |

## 设计要点

- **零分配 latch 提交模型**：1 次批量入队 + 1 个原子计数器 + 调用者参与处理。
  每分块零次 `shared_ptr<packaged_task>` 堚分配、零次 future 同步开销。
  因此 `ThreadPool` **不提供单任务 `submit`**——需要 future 语义时应在调用方
  分块后用 `parallel_*` 系列原语。
- **调用者不空等**：等待分块完成时走「短自旋 → work-steal（帮忙处理队列任务）→
  cv 阻塞」三阶段，调用线程参与计算而非空转。
- **worker 索引**：`parallel_for_each_indexed` 的回调接收 `worker_index`
  （worker 线程 = `0..n-1`，调用者线程 = `n`），供 per-worker 状态
  （如 BPE 训练的并行合并 delta）安全分区。

## API 速览

```cpp
#include <neuralnet.cpp/core_threadpool.hpp>
#include <neuralnet.cpp/core_parallel.hpp>

// 直接用池（分块原语，latch 同步）
nn::ThreadPool pool(8);                                   // 0 → 回落 1 线程
pool.parallel_for_each(v.begin(), v.end(), [](int &x) { ++x; });
pool.parallel_for_each_indexed(v.begin(), v.end(),
    [](int &x, std::size_t worker_index) { /* per-worker 状态 */ });
pool.parallel_transform(in.begin(), in.end(), out.begin(), op);
long long s = pool.parallel_transform_reduce(in.begin(), in.end(), 0LL,
    reduce_op, transform_op);
pool.parallel_for_blocks(blocks.begin(), blocks.end(), heavy_op);  // 块级，无阈值
pool.parallel_for_samples(batch_size, [](std::size_t i) { /* 样本 i */ });

// 顶层算法（内部按 PARALLEL_THRESHOLD 自动串行/并行回落）
nn::for_each(v.begin(), v.end(), op);
nn::transform(a.begin(), a.end(), b.begin(), op);                    // 一元
nn::transform(a.begin(), a.end(), b.begin(), out.begin(), bin_op);   // 二元
nn::transform_reduce(v.begin(), v.end(), init, reduce_op, transform_op);
nn::parallel_for_samples(n, op);                          // 0/1 样本串行回落
```

## 用法

```cpp
#include <neuralnet.cpp/core_parallel.hpp>

nn::parallel_for_samples(n, [](std::size_t i){ /* ... */ });
nn::for_each(v.begin(), v.end(), [](int &x){ x *= 2; });   // 超阈值自动并行
```

CMake（库消费）：

```cmake
add_subdirectory(threadpool)   # 自动拉起 extern/base（若父工程未提供）
target_link_libraries(app PRIVATE neuralnet::threadpool)   # 提供 include + C++23
```

## 依赖

依赖 [`base.neuralnet.cpp`](https://github.com/neuralnet-cpp-dev/base.neuralnet.cpp)（`Result<T>`/断言/常量），以 **submodule** 位于 `extern/base`。

```bash
git clone --recurse-submodules .../threadpool.neuralnet.cpp
# 或已 clone 后：git submodule update --init --recursive
```

## 构建与测试

```bash
git submodule update --init --recursive
cmake -B build
cmake --build build
ctest --test-dir build --output-on-failure
```

| 测试 | 覆盖 |
|---|---|
| `threadpool_threadpool_test` | 构造/析构；`parallel_for_each` 全覆盖（无丢失/无重复）；`parallel_for_each_indexed` worker 索引域；`parallel_for_blocks`/`parallel_for_samples`；并行 transform/reduce 与串行参考一致；重复空并行区压测（latch/cv 丢唤醒回归） |
| `threadpool_parallel_test` | 顶层算法在「串行回落（< 阈值）/ 真并行（≥ 阈值）」两档规模下与串行参考精确对拍（整数 Σ、Σi² 闭式）；空/单元素边界；0/1 样本回落 |

被父工程 `add_subdirectory` 消费时测试**默认不构建**（`NN_BUILD_TESTS` 默认值 =
是否顶层工程）；显式 `-DNN_BUILD_TESTS=ON` 可在嵌入时强制打开。

许可证：LGPL-3.0（见 [LICENSE](LICENSE)）。
