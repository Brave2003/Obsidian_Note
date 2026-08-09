# C++ 与工程能力

## 学习资源推荐

### Modern C++

| 资源 | 类型 | 说明 |
|------|------|------|
| [A Tour of C++ - Bjarne Stroustrup](https://www.stroustrup.com/tour3.html) | 书籍 | C++之父写的快速入门，2-3天可读完 |
| [Effective Modern C++ - Scott Meyers](https://www.oreilly.com/library/view/effective-modern-c/9781491908419/) | 书籍 | C++11/14最佳实践，面试必读书 |
| [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines) | 在线文档 | 官方编码规范 |
| [cppreference.com](https://en.cppreference.com/) | 在线文档 | 最权威的C++参考 |
| [CppCon Back to Basics 系列](https://www.youtube.com/playlist?list=PLHTh1InhhwT6bwIpRk-4nTQTeU7OEHtpQ) | 视频 | 每年CppCon的基础回炉系列 |
| [LearnCpp.com](https://www.learncpp.com/) | 在线教程 | 系统学习C++ |

### 工程工具

| 资源 | 类型 | 说明 |
|------|------|------|
| [Pro Git](https://git-scm.com/book/zh/v2) | 在线书 | 中文Git教程 |
| [CMake Tutorial](https://cmake.org/cmake/help/latest/guide/tutorial/) | 官方文档 | CMake官方入门 |
| [Professional CMake](https://crascit.com/professional-cmake/) | 书籍 | CMake进阶 |
| [GDB Tutorial](https://sourceware.org/gdb/current/onlinedocs/gdb.html/) | 官方文档 | GDB官方手册 |
| [AddressSanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer) | Wiki | 内存错误检测 |
| [ROS2 Tutorials](https://docs.ros.org/en/humble/Tutorials.html) | 官方文档 | ROS2官方教程 |
| [perf Examples](https://www.brendangregg.com/perf.html) | 博客 | Brendan Gregg的perf圣经 |

---

## 核心概念详解

### 1. C++14/17 必知必会

**智能指针**：
```cpp
std::unique_ptr<T>   // 独占所有权，不能拷贝
std::shared_ptr<T>   // 共享所有权，引用计数
std::weak_ptr<T>     // 不增加引用计数，避免循环引用
```

**SLAM中常见用法**：
```cpp
// 地图点用shared_ptr，多个关键帧共享
std::vector<std::shared_ptr<MapPoint>> mvpMapPoints;

// 关键帧用unique_ptr，有明确的所有者
std::unique_ptr<KeyFrame> mpCurrentFrame;
```

**移动语义**：
```cpp
// 避免不必要的拷贝：传出去、插入容器
frame = std::move(new_frame);

// std::move 只是类型转换，本身不移动！
```

**RAII (Resource Acquisition Is Initialization)**：
```cpp
// 锁自动释放
std::lock_guard<std::mutex> lock(mMutex);

// 文件自动关闭
std::ifstream file("data.txt");
```

**Lambda 表达式**：
```cpp
// 常见用途：给优化器添加残差
auto residual = [&params](const T* x, T* r) {
    r[0] = ...;
};
```

**线程安全**：
```cpp
std::mutex mMutexNewKF;  // 保护关键帧列表
std::condition_variable cv;  // 通知线程
```

### 2. CMake 核心

**项目结构模板**：
```cmake
cmake_minimum_required(VERSION 3.16)
project(MySLAM)

# 找依赖
find_package(Eigen3 REQUIRED)
find_package(OpenCV REQUIRED)
find_package(Ceres REQUIRED)

# 库
add_library(my_slam_lib
    src/System.cpp
    src/Tracking.cpp
)

target_include_directories(my_slam_lib PUBLIC include)
target_link_libraries(my_slam_lib
    Eigen3::Eigen
    ${OpenCV_LIBS}
    Ceres::ceres
)

# 可执行文件
add_executable(run_slam app/main.cpp)
target_link_libraries(run_slam my_slam_lib)
```

### 3. GDB 调试

**SLAM调试常用命令**：
```gdb
# 启动
gdb --args ./run_slam config.yaml

# 断点
b Tracking.cc:245
b VioManager::feed_measurement_imu
b condition.cpp:50 if frame_id > 100

# 运行
r               # 开始
c               # 继续
n               # 下一步（不进入函数）
s               # 下一步（进入函数）
finish          # 执行到函数返回

# 查看
p variable      # 打印变量
p R             # 打印旋转矩阵
bt              # 查看调用栈
frame 3         # 跳转到调用栈第3层

# 点云特殊技巧
p T.matrix()    # 打印完整变换矩阵
```

**常用断点位置**（以VINS为例）：
```
estimator.cpp:processImage()         # 视觉数据处理入口
estimator.cpp:optimization()         # 滑窗优化
imu_factor.h:Evaluate()              # IMU残差计算
initial_alignment.cpp:solveGyroscopeBias()  # 陀螺仪偏置初始化
```

### 4. 内存调试

**AddressSanitizer**：
```bash
# 编译时加
g++ -fsanitize=address -g -O1 my_code.cpp

# 会检测：
# - use-after-free
# - heap-buffer-overflow
# - stack-buffer-overflow
# - memory-leaks
```

**Valgrind** (Linux)：
```bash
valgrind --leak-check=full --track-origins=yes ./my_program
```

### 5. 性能分析

**perf** (Linux)：
```bash
perf record -g ./my_program
perf report  # 查看火焰图
```

**Nsight Systems** (Jetson)：
```bash
nsys profile --stats=true ./my_program
```

**常见优化点**：
- 特征提取：并行处理、SIMD加速
- 最近邻搜索：kd-tree替代暴力搜索
- 优化问题：稀疏性利用、Schur补
- 循环展开：编译器优化或手动展开

---

## 面试常见C++问题

1. **智能指针的区别**：unique_ptr vs shared_ptr vs weak_ptr
2. **虚函数实现原理**：vtable、vptr
3. **移动语义的作用**：什么时候对象可以被move
4. **多线程安全**：mutex、lock_guard、deadlock
5. **STL容器选择**：vector vs list vs map vs unordered_map
6. **const 的各种用法**：const变量、const成员函数
7. **内存泄漏排查**：什么情况下会泄漏，怎么找

---

## 动手实践清单

- [ ] 搭建自己的 CMake + Eigen + OpenCV 项目模板
- [ ] 用 GDB 完整跟踪一次 ORB-SLAM3 的 Tracking 线程
- [ ] 用 AddressSanitizer 检查自己的代码
- [ ] 写一个多线程程序（生产者-消费者模式），正确使用 mutex 和条件变量
- [ ] 阅读 VINS-Fusion 的 CMakeLists.txt，理解每个 target
- [ ] 用 rosbag 录制并回放一段数据
- [ ] 尝试交叉编译到 ARM/Jetson 平台
