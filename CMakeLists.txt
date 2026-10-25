# threadpool.neuralnet.cpp — 独立线程池 + 并行算法包装（header-only）
#
# 内容：
#   core_threadpool.hpp  ThreadPool + global_thread_pool()（任务队列/工作线程）
#   core_parallel.hpp    顶层并行算法（for_each/transform/transform_reduce/
#                        parallel_for_blocks/parallel_for_samples），按阈值串行回落
#
# 依赖：base.neuralnet.cpp（Result<T>/断言/常量），经 extern/base submodule 引入。
#
# 消费方式：
#   git submodule add .../base.neuralnet.cpp.git extern/base   # 连同本仓一起
#   add_subdirectory(threadpool)                               # 其会自动 add extern/base
#   target_link_libraries(app PRIVATE neuralnet::threadpool)
#   #include <neuralnet.cpp/core_parallel.hpp>
cmake_minimum_required(VERSION 3.20)
project(neuralnet_threadpool LANGUAGES CXX)

# base 依赖：优先用外部已定义的目标（父工程已 add_subdirectory base 时），
# 否则回落到 submodule。避免重复 add_subdirectory 同一目录报错。
if(NOT TARGET neuralnet_base)
    add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/extern/base
                     ${CMAKE_CURRENT_BINARY_DIR}/extern/base)
endif()

add_library(neuralnet_threadpool INTERFACE)
add_library(neuralnet::threadpool ALIAS neuralnet_threadpool)
target_include_directories(neuralnet_threadpool INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>)
target_link_libraries(neuralnet_threadpool INTERFACE neuralnet::base)
target_compile_features(neuralnet_threadpool INTERFACE cxx_std_23)
