# threadpool.neuralnet.cpp

neuralnet.cpp 生态的**独立线程池 + 并行算法**库（header-only）。
领域无关，可被任何 C++ 项目复用。

| 头文件 | 内容 |
|---|---|
| `core_threadpool.hpp` | `ThreadPool` / `global_thread_pool()` —— 任务队列 + 工作线程 |
| `core_parallel.hpp` | 顶层并行算法：`nn::for_each` / `transform` / `transform_reduce` / `parallel_for_blocks` / `parallel_for_samples`，按 `PARALLEL_THRESHOLD` 自动串行/并行回落 |

## 用法

```cpp
#include <neuralnet.cpp/core_parallel.hpp>

nn::parallel_for_samples(n, [](std::size_t i){ /* ... */ });
nn::for_each(v.begin(), v.end(), [](int &x){ x *= 2; });   // 超阈值自动并行
```

## 依赖

依赖 [`base.neuralnet.cpp`](https://github.com/neuralnet-cpp-dev/base.neuralnet.cpp)（`Result<T>`/断言/常量），以 **submodule** 位于 `extern/base`。

```bash
git clone --recurse-submodules .../threadpool.neuralnet.cpp
# 或已 clone 后：git submodule update --init
```

CMake：

```cmake
add_subdirectory(threadpool)   # 自动拉起 extern/base
target_link_libraries(app PRIVATE neuralnet::threadpool)
```
