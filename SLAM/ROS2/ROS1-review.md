# 第 1 章：ROS1 核心概念回顾

## 开篇段落

ROS1（Robot Operating System）作为机器人领域最成功的中间件框架，从 2007 年诞生至今已经成为学术界和工业界的事实标准。尽管 ROS2 带来了诸多架构改进，但理解 ROS1 的核心设计理念对于掌握 ROS2 至关重要。本章将系统回顾 ROS1 的核心架构，重点分析其设计决策背后的权衡，为后续章节理解 ROS2 的改进动机奠定基础。

**学习目标**：

- 深入理解 ROS1 的 Master-Slave 架构及其设计哲学
- 掌握三种通信机制（话题、服务、动作）的实现原理与适用场景
- 理解 Catkin 构建系统的工作原理与包管理机制
- 分析参数服务器的设计模式与动态重配置能力
- 通过 PR2 机器人案例理解大规模机器人系统的架构设计

---

## Master 节点与分布式架构

### ROS Master 的角色定位

ROS1 采用了中心化的 Master 节点设计，这是整个系统的神经中枢。Master 节点本质上是一个轻量级的名称服务器（Name Server），提供以下核心功能：

1. **名称注册与解析**：维护节点名称到网络地址的映射表
2. **服务发现**：帮助节点之间建立点对点连接
3. **参数服务器**：存储和分发全局配置参数

```
     +----------------+
     |   ROS Master   |
     |   (roscore)    |
     +-------+--------+
             |
     +-------+--------+
     |  Name Service  |
     |   Registry     |
     +----------------+
            / | \
           /  |  \
    +-----+   |   +-----+
    |Node1|   |   |Node2|
    +-----+   |   +-----+
              |
          +-------+
          |Node3  |
          +-------+
```

### XMLRPC 协议与通信流程

ROS1 使用 XMLRPC 作为节点与 Master 之间的通信协议。这个选择反映了 2007 年的技术栈现状：XMLRPC 简单、跨语言支持好，但也带来了性能开销。

**节点启动与注册流程**：

1. 节点启动时，通过 `ROS_MASTER_URI` 环境变量找到 Master
2. 使用 XMLRPC 调用 `registerNode()` 方法注册自己
3. Master 返回注册确认，节点获得唯一 ID
4. 节点注册自己提供的话题/服务到 Master

**话题订阅建立流程**：

```
发布者节点                Master                订阅者节点
    |                      |                      |
    |--registerPublisher-->|                      |
    |                      |<--registerSubscriber-|
    |                      |                      |
    |                      |--publisherUpdate---->|
    |<-----------------requestTopic---------------|
    |------------------TCPROS连接---------------->|
```

这个过程的关键点：

- Master 只负责"牵线搭桥"，不参与数据传输
- 节点之间建立直接的 TCPROS 连接传输数据
- 这种设计降低了 Master 负载，但也引入了单点故障

### 分布式系统设计考量

ROS1 的分布式架构设计有几个重要特征：

**1. 松耦合通信**
节点之间通过话题进行松耦合通信，发布者和订阅者互不知晓对方存在。这种设计带来了极大的灵活性，但也引入了一些挑战：

- **优势**：节点可以独立开发、测试和部署
- **挑战**：难以保证消息的可靠传输和时序一致性

**2. 点对点数据传输**
数据不经过 Master 直接在节点间传输，这个设计决策影响深远：

```
带宽利用率 = 数据量 / (数据量 + 协议开销)

对于 ROS1：
- 小消息（<1KB）：带宽利用率约 60-70%
- 大消息（>100KB）：带宽利用率可达 95%+
```

**3. 网络透明性**
ROS1 的网络透明性设计让分布式部署变得简单，但也带来了安全隐患：

- 任何知道 Master URI 的节点都可以加入网络
- 没有内置的认证和加密机制
- 适合可信网络环境，不适合公网部署

### 多机通信配置

在多机环境下部署 ROS1 需要谨慎配置：

```bash
# 机器 A (Master 所在)
export ROS_MASTER_URI=http://192.168.1.100:11311
export ROS_IP=192.168.1.100

# 机器 B (Worker 节点)
export ROS_MASTER_URI=http://192.168.1.100:11311
export ROS_IP=192.168.1.101
export ROS_HOSTNAME=worker-robot  # 可选，用于 DNS 解析
```

**网络配置检查清单**：

1. 所有机器时钟同步（NTP）
2. 防火墙开放必要端口（11311 for Master, 随机端口 for nodes）
3. 主机名解析正确（/etc/hosts 或 DNS）
4. 网络延迟 < 10ms（局域网环境）

---

## 话题、服务、动作通信机制

### 话题（Topics）：发布-订阅模式

话题是 ROS1 中最基础的通信机制，实现了经典的发布-订阅模式。

**消息传输特征**：

- **异步通信**：发布者不等待订阅者接收
- **多对多通信**：多个发布者和订阅者可以共享同一话题
- **无应答机制**：发布者不知道消息是否被接收

**TCPROS 协议细节**：

```
[4字节长度][消息序列化数据]
           |
           +-- 使用 ROS 消息序列化格式
               (类似 Protocol Buffers 但更简单)
```

**性能特征分析**：

- 延迟：局域网 < 1ms，取决于消息大小和网络状况
- 吞吐量：可达网络带宽的 80-90%（大消息）
- CPU 开销：序列化/反序列化约占 5-15%（取决于消息复杂度）

**队列管理策略**：

```python
# 发布者队列大小设置
pub = rospy.Publisher('topic', MessageType, queue_size=10)
# queue_size 影响：
# - 太小：高频发布时可能丢失消息
# - 太大：占用内存，增加延迟
```

### 服务（Services）：请求-响应模式

服务提供同步的请求-响应通信模式，适合需要确定性结果的场景。

**服务调用流程**：

```
客户端                    服务器
  |                         |
  |---请求（Request）------>|
  |                         |处理请求
  |<---响应（Response）------|
  |                         |
```

**关键设计决策**：

1. **同步阻塞**：客户端等待服务器响应
2. **单次连接**：每次调用建立新的 TCP 连接
3. **无状态**：服务器不维护客户端状态

**性能考量**：

```
服务调用开销 = 连接建立时间 + 请求传输 + 处理时间 + 响应传输

典型场景：
- 小请求（<1KB）：总开销 5-10ms
- 大请求（>10KB）：主要受网络带宽限制
```

**持久连接优化**：

```python
# 使用持久连接减少开销
from rospy import ServiceProxy
service = ServiceProxy('service_name', ServiceType, persistent=True)
# 重用 TCP 连接，减少握手开销
```

### 动作（Actions）：带反馈的异步任务

动作是 ROS1 中最复杂的通信机制，适合长时间运行的任务。

**动作协议的五个组成部分**：

1. **Goal**：任务目标
2. **Result**：最终结果
3. **Feedback**：执行过程中的反馈
4. **Status**：任务状态（pending/active/succeeded/aborted）
5. **Cancel**：取消机制

```
动作内部实现 = 5个话题 + 状态机管理
           /action_name/goal        (目标发送)
           /action_name/cancel      (取消请求)
           /action_name/status      (状态更新)
           /action_name/feedback    (进度反馈)
           /action_name/result      (最终结果)
```

**状态机转换图**：

```
        [PENDING]
            |
            v
        [ACTIVE] <---> [PREEMPTING]
         /    \              |
        v      v             v
   [SUCCEEDED] [ABORTED] [PREEMPTED]
```

**设计模式应用场景**：

- **导航任务**：发送目标点，接收路径执行反馈
- **机械臂控制**：执行轨迹，监控执行进度
- **感知处理**：长时间的图像处理或 SLAM 建图

---

## Catkin 构建系统

### Catkin 的设计理念

Catkin 是 ROS1 的构建系统，基于 CMake 扩展而来，解决了大规模机器人软件的构建挑战。

**核心设计目标**：

1. **包管理**：支持细粒度的功能包组织
2. **依赖管理**：自动处理包之间的依赖关系
3. **并行构建**：充分利用多核 CPU
4. **跨平台**：支持 Linux、macOS（部分）

### 工作空间结构

```
catkin_ws/
├── src/               # 源代码目录
│   ├── package1/
│   │   ├── CMakeLists.txt
│   │   ├── package.xml
│   │   ├── src/
│   │   └── include/
│   └── package2/
├── build/             # 构建中间文件
│   └── [CMake 生成的构建文件]
├── devel/             # 开发空间
│   ├── setup.bash     # 环境配置脚本
│   ├── lib/           # 编译的库文件
│   └── share/         # 资源文件
└── install/           # 安装空间（可选）
    └── [发布版本文件]
```

### CMakeLists.txt 深度解析

```cmake
cmake_minimum_required(VERSION 3.0.2)
project(my_robot_package)

# 查找 catkin 和依赖包
find_package(catkin REQUIRED COMPONENTS
  roscpp
  std_msgs
  sensor_msgs
  geometry_msgs
)

# 声明 catkin 包
catkin_package(
  INCLUDE_DIRS include
  LIBRARIES ${PROJECT_NAME}
  CATKIN_DEPENDS roscpp std_msgs
  DEPENDS eigen3  # 系统依赖
)

# 包含目录
include_directories(
  include
  ${catkin_INCLUDE_DIRS}
)

# 编译库
add_library(${PROJECT_NAME}
  src/algorithm.cpp
)

# 编译可执行文件
add_executable(robot_node src/main.cpp)
target_link_libraries(robot_node
  ${PROJECT_NAME}
  ${catkin_LIBRARIES}
)

# 安装规则
install(TARGETS robot_node
  RUNTIME DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)
```

### 包依赖管理

**package.xml 结构**：

```xml
<?xml version="1.0"?>
<package format="2">
  <name>my_robot_package</name>
  <version>1.0.0</version>
  <description>机器人控制包</description>

  <maintainer email="dev@robot.com">Developer</maintainer>
  <license>MIT</license>

  <!-- 构建依赖 -->
  <buildtool_depend>catkin</buildtool_depend>
  <build_depend>roscpp</build_depend>

  <!-- 运行依赖 -->
  <exec_depend>roscpp</exec_depend>
  <exec_depend>rospy</exec_depend>

  <!-- 测试依赖 -->
  <test_depend>rostest</test_depend>
</package>
```

**依赖解析算法**：

1. 拓扑排序确定构建顺序
2. 检测循环依赖
3. 并行构建无依赖关系的包

### 构建优化技巧

**1. 并行构建加速**：

```bash
# 使用所有 CPU 核心
catkin_make -j$(nproc)

# 或使用 catkin_tools（推荐）
catkin build --jobs $(nproc)
```

**2. 增量构建优化**：

```bash
# 只构建修改的包
catkin build --this

# 构建指定包及其依赖
catkin build package_name --deps
```

**3. ccache 加速重复编译**：

```bash
# 安装 ccache
sudo apt-get install ccache

# 配置 catkin 使用 ccache
export CC="ccache gcc"
export CXX="ccache g++"
```

---

## 参数服务器与动态配置

### 参数服务器架构

ROS1 的参数服务器是一个中心化的配置存储系统，运行在 Master 节点上。它使用层次化的命名空间存储键值对。

**参数类型支持**：

- 基本类型：bool, int, double, string
- 复合类型：list, dict（嵌套结构）
- 二进制数据：base64 编码的二进制 blob

**命名空间层次结构**：

```
/
├── robot_name              # 全局参数
├── /navigation/
│   ├── max_velocity        # 导航模块参数
│   ├── planner/
│   │   ├── algorithm       # 规划器配置
│   │   └── resolution
│   └── controller/
│       └── gains           # 控制器参数
└── /perception/
    ├── camera/
    │   └── fps
    └── lidar/
        └── range
```

### 参数操作 API

**参数读写操作**：

```python
# Python API
import rospy

# 读取参数
max_vel = rospy.get_param('/navigation/max_velocity', 1.0)  # 带默认值
params = rospy.get_param('/navigation/')  # 获取整个命名空间

# 写入参数
rospy.set_param('/navigation/max_velocity', 2.0)

# 删除参数
rospy.delete_param('/navigation/obsolete_param')

# 检查参数存在
if rospy.has_param('/navigation/max_velocity'):
    # 参数存在
    pass
```

**私有参数与相对命名**：

```python
# 私有参数（节点命名空间）
rospy.init_node('my_node')
# 参数实际路径：/my_node/param_name
private_param = rospy.get_param('~param_name')

# 相对参数（当前命名空间）
# 如果当前命名空间是 /robot1/
relative_param = rospy.get_param('sensor/range')
# 实际路径：/robot1/sensor/range
```

### 动态重配置（Dynamic Reconfigure）

动态重配置是 ROS1 的一个强大特性，允许运行时修改参数而无需重启节点。

**配置文件定义（.cfg）**：

```python
#!/usr/bin/env python
from dynamic_reconfigure.parameter_generator_catkin import *

gen = ParameterGenerator()

# 添加参数：名称、类型、级别、描述、默认值、最小值、最大值
gen.add("max_velocity", double_t, 0,
        "Maximum velocity", 1.0, 0.0, 5.0)
gen.add("enable_obstacle_avoidance", bool_t, 0,
        "Enable obstacle avoidance", True)

# 枚举类型
algorithm_enum = gen.enum([
    gen.const("DWA", int_t, 0, "Dynamic Window Approach"),
    gen.const("TEB", int_t, 1, "Timed Elastic Band"),
    gen.const("MPC", int_t, 2, "Model Predictive Control")
], "Planning algorithm selection")

gen.add("algorithm", int_t, 0,
        "Path planning algorithm", 0, 0, 2,
        edit_method=algorithm_enum)

exit(gen.generate("my_package", "my_node", "MyConfig"))
```

### 参数服务器性能分析

**性能特征**：

```
参数读取延迟 = 网络往返时间 + XMLRPC 解析
             ≈ 1-5ms（局域网）

批量操作优化：
- 单个参数读取：N 次网络往返
- 命名空间读取：1 次网络往返
- 推荐：启动时批量读取，缓存在本地
```

**缓存策略**：

```python
class ParameterCache:
    def __init__(self, namespace):
        # 启动时批量读取
        self.params = rospy.get_param(namespace)
        self.namespace = namespace

    def get(self, key, default=None):
        # 从本地缓存读取
        return self.params.get(key, default)

    def refresh(self):
        # 定期刷新缓存
        self.params = rospy.get_param(self.namespace)
```

---

## 产业案例研究：Willow Garage PR2 机器人系统架构

### PR2 系统概述

PR2（Personal Robot 2）是 Willow Garage 开发的双臂移动服务机器人，是 ROS1 发展史上的里程碑项目。其系统架构充分展示了 ROS1 在复杂机器人系统中的应用。

**硬件规格**：

- 2 个 7 自由度机械臂 + 2 自由度夹爪
- 全向移动底盘（4 个驱动轮）
- 传感器阵列：激光雷达、立体相机、Kinect、力/力矩传感器
- 计算资源：2 个 Xeon 服务器（16 核心），32GB RAM

### 软件架构设计

**节点拓扑结构**：

```
                    PR2 ROS 节点架构
    +------------------------------------------------+
    |                  高层任务规划                    |
    |    task_executive    move_base    manipulation  |
    +------------------------------------------------+
                            |
    +------------------------------------------------+
    |                   中间件服务                     |
    |  tf  robot_state  diagnostics  power_management |
    +------------------------------------------------+
                            |
    +------------------------------------------------+
    |                  硬件抽象层                       |
    |   pr2_controller_manager    pr2_ethercat        |
    +------------------------------------------------+
                            |
    +------------------------------------------------+
    |                  驱动程序层                       |
    | motor_drivers  sensor_drivers  camera_drivers   |
    +------------------------------------------------+
```

**关键设计决策**：

1. **实时控制回路分离**：
   - 1kHz 电机控制回路运行在实时内核
   - 100Hz 运动规划运行在普通用户空间
   - 通过共享内存传递控制命令

2. **传感器数据流水线**：

```
激光雷达 (40Hz) ─┐
              ├─> 传感器融合 ─> 八叉树地图 ─> 导航规划
立体相机 (30Hz) ─┘                    |
                                  v
Kinect (30Hz) ────> 点云处理 ─> 物体识别 ─> 抓取规划
```

3. **分布式计算架构**：
   - 主计算机：高层规划、传感器融合
   - 从计算机：图像处理、点云处理
   - 实时控制器：电机控制、安全监控

### 通信模式选择策略

PR2 在不同场景下选择不同的 ROS1 通信机制：

**话题使用场景**：

- 传感器数据流（激光、相机、IMU）
- 机器人状态发布（关节状态、电池状态）
- 可视化数据（RViz 显示）

**服务使用场景**：

- 运动学求解（IK 服务）
- 抓取规划请求
- 系统配置更改

**动作使用场景**：

- 机械臂轨迹执行
- 导航目标执行
- 复杂任务执行（开门、抓取）

### 性能优化实践

**1. 消息传输优化**：

```cpp
// 使用 nodelet 减少数据拷贝
class ImageProcessingNodelet : public nodelet::Nodelet {
    // 同进程内使用指针传递，避免序列化
    void imageCallback(const sensor_msgs::ImageConstPtr& msg) {
        // 零拷贝处理
    }
};
```

**2. 话题分流策略**：

```python
# 高频数据使用独立话题
/base_scan          # 40Hz 激光数据
/base_scan_filtered # 10Hz 滤波数据（导航使用）
/base_scan_marking  # 5Hz 障碍标记（建图使用）
```

**3. 参数服务器优化**：

```yaml
# 启动时批量加载参数
rosparam:
  - file: config/navigation.yaml
    ns: /move_base
  - file: config/manipulation.yaml
    ns: /arm_controller
```

### 故障处理与恢复

PR2 实现了多层次的故障检测与恢复机制：

**1. 硬件层安全机制**：

- 急停按钮（硬件中断）
- 电机过流保护
- 关节限位检测

**2. 软件层监控**：

```python
# 诊断聚合器配置
analyzers:
  motors:
    type: diagnostic_aggregator/GenericAnalyzer
    path: Motors
    contains: ['motor_']
    timeout: 5.0

  sensors:
    type: diagnostic_aggregator/GenericAnalyzer
    path: Sensors
    contains: ['laser', 'camera', 'imu']
    timeout: 2.0
```

**3. 系统级恢复策略**：

- 节点看门狗（自动重启）
- 降级运行模式（单臂操作）
- 安全停机程序

### 经验教训总结

PR2 项目为 ROS2 的设计提供了宝贵经验：

1. **Master 单点故障**：PR2 在实际部署中多次遇到 Master 崩溃导致全系统失效
2. **实时性不足**：TCPROS 的不确定延迟影响控制性能
3. **安全性缺失**：缺乏认证机制，任何节点都可以控制机器人
4. **资源开销大**：每个节点都是独立进程，内存和 CPU 开销显著

---

## 高级话题：ROS1 分布式系统优化与多 Master 方案

### 分布式系统性能优化

#### 网络拓扑优化

在大规模机器人系统中，网络拓扑设计直接影响系统性能：

```
星型拓扑（中心化）：           网状拓扑（分布式）：
     Master                   Node1 ←→ Node2
    /   |   \                   ↑  ×  ↓
Node1 Node2 Node3             Node3 ←→ Node4

延迟：O(1)跳                 延迟：O(log n)跳
带宽：受中心限制              带宽：多路径均衡
容错：单点故障                容错：多路径冗余
```

**优化策略**：

1. **话题路由优化**：根据数据流量模式调整网络拓扑
2. **局部性原理**：相关节点部署在同一子网
3. **带宽预留**：为关键数据流预留网络带宽

#### 消息传输优化技术

**1. 消息批处理（Message Batching）**：

将多条小消息打包成一条大消息，减少协议头开销。对于小消息（<1KB），协议头可能占 30-40% 带宽。

**2. 压缩传输**：

```python
import cv2
from sensor_msgs.msg import CompressedImage

# 发布压缩图像
def publish_compressed(image):
    # JPEG 压缩
    _, compressed = cv2.imencode('.jpg', image,
                                 [cv2.IMWRITE_JPEG_QUALITY, 80])
    msg = CompressedImage()
    msg.data = compressed.tostring()
    msg.format = "jpeg"
    compressed_pub.publish(msg)
```

**3. 共享内存传输（同机优化）**：

使用 nodelet 实现零拷贝，同进程内使用指针传递而非序列化。

### 多 Master 架构方案

#### 方案一：Foreign Relay

Foreign Relay 是最简单的多 Master 连接方案：

- ✅ 实现简单，不需要修改 ROS 核心
- ✅ 可以选择性中继特定话题
- ❌ 增加延迟（额外的序列化/反序列化）
- ❌ 需要手动配置每个中继话题

#### 方案二：Multimaster FKIE

Multimaster FKIE 是功能最完整的多 Master 解决方案：

```xml
<!-- multimaster.launch -->
<launch>
  <!-- Master 发现节点 -->
  <node name="master_discovery" pkg="master_discovery_fkie"
        type="master_discovery">
    <param name="mcast_group" value="224.0.0.1"/>
    <param name="mcast_port" value="11511"/>
  </node>

  <!-- Master 同步节点 -->
  <node name="master_sync" pkg="master_sync_fkie"
        type="master_sync">
    <rosparam>
      sync_topics: ['/sensor_data', '/robot_status']
      sync_services: ['/get_plan', '/compute_ik']
    </rosparam>
  </node>
</launch>
```

**关键特性**：

1. **自动发现**：使用组播 UDP 自动发现其他 Master
2. **选择性同步**：可配置同步规则
3. **冲突解决**：处理命名冲突和时钟同步

```
  机器人1                    机器人2
+----------+              +----------+
| Master 1 |←---发现---→| Master 2 |
+----------+              +----------+
     ↑                         ↑
     |同步                     |同步
     ↓                         ↓
+----------+              +----------+
| 节点组 1  |←---数据---→| 节点组 2  |
+----------+              +----------+
```

#### 方案三：ROS1 Gateway（面向云机器人）

针对云机器人场景的网关架构，通过 WebSocket 连接云端，实现话题过滤、数据压缩和带宽优化。

### 实时性增强技术

#### RT-PREEMPT 内核集成

```bash
# 安装 RT-PREEMPT 内核
sudo apt-get install linux-image-rt-amd64

# 配置实时优先级
cat <<EOF > /etc/security/limits.d/ros-rt.conf
@ros-rt - rtprio 98
@ros-rt - memlock unlimited
EOF

# 将用户添加到实时组
sudo usermod -a -G ros-rt $USER
```

#### 确定性通信保证

**时间触发通信**：固定周期发布（如 1kHz），监控抖动。

**优先级队列管理**：高优先级消息优先传输，限制队列大小防止内存溢出。

---

## 论文导读

**关键论文推荐**：

1. **"ROS: an open-source Robot Operating System"** (Quigley et al., 2009)
   - ROS 原始设计理念
   - 分布式架构决策依据
   - 早期应用案例

2. **"Performance Evaluation of ROS-Based Systems"** (Maruyama et al., 2016)
   - ROS1 性能基准测试
   - 瓶颈分析方法
   - 优化建议

3. **"Real-Time ROS Extensions"** (Wei et al., 2016)
   - RT-PREEMPT 集成经验
   - 实时性保证机制
   - 工业应用案例

**开源项目推荐**：

- [multimaster_fkie](https://github.com/fkie/multimaster_fkie)：多 Master 解决方案
- [ros_comm](https://github.com/ros/ros_comm)：ROS1 核心通信实现
- [nodelet_core](https://github.com/ros/nodelet_core)：零拷贝通信框架

---

## 本章小结

### 核心概念总结

1. **Master-Slave 架构**：
   - Master 作为名称服务器，负责节点发现和连接建立
   - 节点间直接通信，Master 不参与数据传输
   - 简化了系统设计，但引入了单点故障风险

2. **三种通信模式**：
   - **话题**：异步发布-订阅，适合数据流传输
   - **服务**：同步请求-响应，适合命令执行
   - **动作**：带反馈的异步任务，适合长时间操作

3. **关键性能公式**：

```
系统延迟 = 网络延迟 + 序列化时间 + 处理时间

其中：
- 网络延迟 ≈ RTT/2 (局域网 < 1ms)
- 序列化时间 ≈ 消息大小 / CPU频率 × 复杂度因子
- 处理时间 = 应用相关
```

4. **Catkin 构建系统**：
   - 基于 CMake 的包管理系统
   - 支持并行构建和依赖管理
   - 工作空间隔离开发环境

5. **参数服务器**：
   - 中心化配置存储
   - 支持动态重配置
   - 层次化命名空间组织

### 设计权衡分析

| 设计决策 | 优势 | 劣势 | ROS2 改进方向 |
|---------|------|------|-------------|
| 中心化 Master | 简单、易理解 | 单点故障 | DDS 分布式发现 |
| XMLRPC 协议 | 跨语言支持好 | 性能开销大 | DDS-RTPS 二进制协议 |
| 进程隔离 | 故障隔离好 | 资源开销大 | 组件化架构 |
| TCPROS 传输 | 可靠传输 | 缺乏 QoS 控制 | DDS QoS 策略 |
| 无安全机制 | 部署简单 | 安全风险 | SROS2 安全框架 |

### 从 ROS1 到 ROS2 的演进动力

通过本章的分析，我们可以看到推动 ROS2 诞生的关键因素：

1. **可靠性需求**：消除 Master 单点故障
2. **实时性需求**：确定性通信和调度
3. **安全性需求**：认证、加密、访问控制
4. **嵌入式支持**：降低资源占用
5. **产业化需求**：生产环境的稳定性和可维护性

---

## 练习题

### 基础题

**练习 1.1：Master 故障分析**
假设一个 ROS1 系统有 10 个节点正在运行，突然 Master 节点崩溃。请分析：
a) 已建立的话题通信是否会中断？
b) 新节点能否加入系统？
c) 参数服务器的数据会发生什么？

> [!faq]- 📝 参考答案
>
> a) **已建立的话题通信不会立即中断**。
>
> **为什么？** 这是 ROS1 "牵线搭桥"设计的核心体现——Master 只在连接建立阶段起作用（名称注册 → 地址交换 → 帮助双方建立 TCPROS），一旦 TCPROS 直连建立，数据流完全绕开 Master。这类似于电话交换机：拨号时需要交换机转接，通话建立后交换机就不再参与。
>
> **但要注意**：这只是"不会立即中断"。如果任何一个节点因网络抖动导致 TCPROS 断开，它将无法重新向 Master 注册来重建连接，此时通信彻底失效。这就是单点故障的本质——系统进入一个脆弱的"玻璃态"，看似正常，但经不起任何扰动。
>
> b) **新节点无法加入系统**。
>
> **为什么？** 新节点启动的第一步就是通过 `ROS_MASTER_URI` 找到 Master 并调用 XMLRPC `registerNode()`。Master 不可用意味着这套"电话本"机制完全失效。新节点连自己的存在都无法声明，更谈不上发现其他节点、建立连接。这是中心化注册模型的固有缺陷。
>
> c) **参数服务器数据完全丢失**。
>
> **为什么？** 参数服务器不是独立进程，它作为 Master 进程的一个功能模块运行。Master 崩溃 = 整个参数服务器内存被操作系统回收。虽然节点本地可能缓存了部分参数，但全局配置状态完全消失。对比分布式配置中心（如 etcd/ZooKeeper），这暴露了单进程耦合的脆弱性。
>
> **ROS2 的解决思路**：DDS 的自动发现机制通过组播在局域网内广播节点信息，不再需要中心化 Master。参数服务被拆分为独立节点，支持持久化存储。

---

**练习 1.2：通信模式选择**
为以下场景选择最合适的 ROS1 通信机制（话题/服务/动作），并说明理由：
a) 激光雷达数据流（40Hz）
b) 获取机器人当前位置
c) 机械臂移动到指定位置
d) 紧急停止命令

> [!faq]- 📝 参考答案
>
> a) **话题**——高频数据流的标准选择
>
> **为什么不是服务？** 服务是同步阻塞的。如果客户端以 40Hz 频率调用服务，这意味着服务器必须在 25ms 内完成处理并返回，任何延迟都会导致调用方堆积。更重要的是：数据流的本质是"持续产生、可能多个消费者"，这与话题的多对多模型完美匹配，而服务的一对一模型会迫使数据源为每个消费者单独发送一份。
>
> **为什么不是动作？** 动作是面向"单个重要任务"设计的（如导航到某个目标点），其内部包含了 5 个话题和管理状态机的开销，用来传递持续的数据流是杀鸡用牛刀。
>
> b) **服务**——一次性查询的最优解
>
> **为什么不是话题？** 如果你订阅话题获取位置，你需要等待下一次发布才能拿到数据——这可能是几十毫秒之后。服务提供即时响应，符合"问-答"的语义。话题的异步模型适合"我不需要立刻知道，但我想持续知道"，而位置查询显然是"我现在就要知道一次"。
>
> **为什么不是动作？** 获取位置是一个瞬间操作（通常在微秒级完成），不需要反馈进度、不需要中途取消、不需要状态机。动作为此引入的复杂性完全多余。
>
> c) **动作**——长时间异步任务的最佳匹配
>
> **为什么比服务好？** 机械臂移动到目标位置可能需要几秒甚至几十秒。如果用服务（同步阻塞），调用方会一直阻塞到移动完成，期间无法做任何其他事情，也无法取消请求。动作提供了三个关键能力：
> - **反馈**：调用方可以实时获知执行进度（如当前关节角度）
> - **取消**：遇到障碍物时可以优雅地中断执行
> - **异步**：调用方在等待期间可以继续处理其他事情
>
> **为什么不用话题？** 话题无法保证"这个命令发给谁"——机械臂控制需要明确的 target 和一对一的语义保证。
>
> d) **话题**——紧急停止的特殊考量
>
> **为什么不用服务？** 紧急停止需要的是"以最快速度广播给所有相关节点"。服务的一对一同步调用在紧急场景下有两个致命问题：一是延迟——客户端需要等待每个节点的响应；二是范围——需要显式地逐个调用每个相关节点。当你的机器人正在撞向墙壁时，你不应该在等待 `stop_service` 返回。
>
> **关键设计决策**：使用话题时设置 `queue_size=1` 并开启可靠传输（TCPROS），保证只保留最新的停止命令，且确保命令一定送达。这个设计体现了"速度优先于丰富性"的紧急处理原则。

---

**练习 1.3：Catkin 工作空间问题**
你有两个 Catkin 工作空间：ws1 和 ws2。ws1 中有包 A（版本 1.0），ws2 中也有包 A（版本 2.0）。如果按照 ws1、ws2 的顺序 source 两个工作空间的 setup.bash，运行时会使用哪个版本的包 A？

> [!faq]- 📝 参考答案
>
> 会使用 **ws2 中的包 A（版本 2.0）**。
>
> **为什么是"后来者覆盖"而不是"先来者优先"？**
>
> `setup.bash` 的核心操作是将工作空间路径 **前置**（prepend）到 `ROS_PACKAGE_PATH` 环境变量。当 `rospack find A` 查找包时，它按 `ROS_PACKAGE_PATH` 中的顺序遍历，找到第一个匹配就返回。
>
> 所以 `source ws1/setup.bash → source  ws2/setup.bash` 的结果是：
> ```
> ROS_PACKAGE_PATH = ws2/src:ws1/src:/opt/ros/<distro>/share
> ```
> ws2 排在 ws1 前面，因此 ws2 的包 A 先被找到。这是一种 **遮蔽（shadowing）** 机制。
>
> **为什么设计成遮蔽而不是报错？**
>
> 这是刻意为之的设计：允许开发者创建含同名包的工作空间来 **覆盖系统或上游的包**，在开发和调试中极为实用。比如你可以在不修改系统 ROS 安装的前提下，用一个本地 workspace 中的修改版 `move_base` 来测试你的改进——只需 source 你的 workspace。
>
> 验证命令：
> ```bash
> echo $ROS_PACKAGE_PATH
> rospack find A  # 会显示 ws2 中的路径
> ```

---

### 挑战题

**练习 1.4：性能优化方案设计**
某机器人系统有一个相机节点发布 1920×1080 的 RGB 图像（30 FPS），三个处理节点订阅这些图像。当前架构导致 CPU 使用率过高，网络带宽接近饱和。请设计一个优化方案，要求：
- 减少 CPU 使用率 50%
- 减少网络带宽 70%
- 保持处理精度

> [!faq]- 📝 参考答案：分层优化，建立"先诊断-再对症"的思维框架
>
> **首先诊断瓶颈来源**：
>
> 当前架构下，每次图像发布完整的数据流是：
> ```
> 相机节点获取原始图像（6.2 MB/帧）
>   → 序列化为 ROS 消息
>     → 写入 TCP socket
>       → 内核态 TCP 栈（3 次拷贝到各订阅者）
>         → 订阅者 TCP 接收
>           → 反序列化为图像矩阵
>             → 传递给处理算法
> ```
>
> 这个流水线中的三大浪费：
> - **序列化浪费**：将内存中的连续图像数组转换为 ROS 消息格式，需要遍历并拷贝每个像素
> - **网络拷贝浪费**：同一份 6.2 MB 数据在 TCP 栈中被复制 3 次（每个订阅者一份）
> - **反序列化浪费**：3 个订阅者各自做一次反向转换
>
> **总资源消耗**（单帧）：序列化 1 次 + 网络传输 3×6.2 MB + 反序列化 3 次 = 24.8 MB 的数据移动，每秒 30 帧 = **744 MB/s 的数据搬运**——这就是 CPU 高、带宽满的根源。
>
> ---
>
> **优化方案（分层递进）**：
>
> **第 1 层：Nodelet 零拷贝（CPU ↓40%）**
>
> 将相机节点和三个处理节点改写为 nodelet，运行在同一进程内。
>
> **为什么这样有效？** Nodelet 的核心理念是"进程内直接传指针"而不是"跨进程传数据"：
> ```cpp
> // 传统节点：跨进程通信，必须序列化
> image_pub.publish(image);  // 序列化 → TCP → 反序列化
>
> // Nodelet：同进程，直接传共享指针
> void imageCallback(const sensor_msgs::ImageConstPtr& msg) {
>     processImage(msg);  // const Ptr 是引用计数指针，零拷贝
> }
> ```
> 三个处理节点不再各自拉一份 TCP 数据，而是共享同一个内存中的图像对象。原先的 1 次序列化 + 3 次反序列化全部消除。
>
> **为什么满足"保持精度"要求？** Nodelet 只是改变了数据传递方式（从跨进程 TCP 到进程内指针），不改变图像数据本身——处理算法看到的像素值完全一致。
>
> **第 2 层：话题分流 + 选择性处理（CPU ↓15%，带宽 ↓10%）**
>
> 不是所有处理节点都需要全分辨率。建立处理金字塔：
> ```
> /camera/image_raw        → 30Hz × 1080p（原始全分辨率）
> /camera/image_half       → 15Hz × 540p（降采样，用于初步检测）
> /camera/image_roi        → 30Hz × ROI区域（感兴趣区域，用于精细分析）
> ```
> **为什么这样设计？** 三个处理节点可能做不同的事情：节点 A 做人脸检测（只需 ROI）、节点 B 做场景理解（降采样足够）、节点 C 做视觉 SLAM（需要全分辨率的特征点但不是每帧都需要）。统一推送 1080p@30Hz 意味着每个节点都在处理不需要精度的数据。
>
> **第 3 层：远程节点的压缩传输（带宽 ↓65%）**
>
> 如果某个处理节点必须运行在远程机器上（如云端 AI 推理），对这部分使用 JPEG 压缩：
> ```python
> # 智能压缩：根据订阅者位置自适应
> if is_remote_subscriber:
>     compressed_msg = CompressedImage()
>     compressed_msg.data = cv2.imencode('.jpg', image, [cv2.IMWRITE_JPEG_QUALITY, 85])[1]
>     compressed_pub.publish(compressed_msg)
> ```
> **为什么 quality=85？** 这是 JPEG 压缩的"甜点"区域——85 以上的质量改善人眼不可感知，但带宽消耗会急剧上升。quality=85 时压缩比约 15:1，186 MB/s → 12.4 MB/s，对视觉 SLAM 和物体检测的特征提取精度影响小于 1%。
>
> **为什么不是 H.264？** H.264 是帧间编码，需要 P 帧/B 帧参考前后帧。如果网络丢包，错误会传播到后续多帧。对于机器人感知（每帧独立处理的场景），帧内编码（JPEG/MJPEG）的容错性更好——一帧丢失不影响后续帧。
>
> ---
>
> **综合效果验证**：
> | 优化项 | CPU 节省 | 带宽节省 | 精度影响 |
> |--------|---------|---------|---------|
> | Nodelet 零拷贝 | 40% | 0% | 无 |
> | 话题分流 | 15% | 10% | 无（对应节点仍拿到所需精度） |
> | JPEG 压缩 | -5%（编码开销） | 65% | <1% 特征损失 |
> | **总计** | **~50%** | **~75%** | **可忽略** |

---

**练习 1.5：分布式系统设计**
设计一个多机器人 SLAM 系统，要求：
- 3 个机器人协同建图
- 每个机器人有自己的 Master
- 实时共享地图数据
- 处理网络分区故障

> [!faq]- 📝 参考答案：从"单机思维"到"分布式思维"的转换
>
> **核心挑战分析**：
>
> 多机器人 SLAM 的难点不在于"如何建图"（单个 SLAM 算法已经很成熟），而在于"如何在不可靠的网络上让多个局部视角融合成全局一致的地图"。这本质上是分布式系统的一致性（Consistency）、可用性（Availability）、分区容错（Partition Tolerance）的 CAP 权衡问题。
>
> 在这个场景中，网络分区（机器人进入不同 WiFi 覆盖区）是不可避免的物理现实，所以我们必须在 C 和 A 之间做取舍：
> - **强一致性（C）**：所有机器人始终看到完全相同的地图 → 网络分区时不可用
> - **高可用性（A）**：每个机器人始终能建图 → 分区期间数据可能冲突
>
> 我们选择 **最终一致性**（Eventual Consistency）：牺牲实时强一致性，换取分区容忍。
>
> ---
>
> **架构设计逐个决策**：
>
> **1. 为什么选择 multimaster_fkie 而不是 Foreign Relay？**
>
> Foreign Relay 的本质是单向桥接——话题数据从 A 的 Master 拷贝到 B 的 Master。在 3 个机器人场景中，这意味着：每个机器人都需要配置到其他 2 个的 relay 链路（共 6 条），且需要手动管理同步规则。更重要的是，Relay 无法处理双向的发现——机器人 C 不知道机器人 A 和 B 已经建立了连接。
>
> Multimaster FKIE 通过组播 UDP（224.0.0.x）实现 **自动拓扑发现**：任何一个 Master 上线后，通过组播广播自己的存在，其他 Master 自动建立同步通道。这符合分布式系统的"去中心化发现"原则。
>
> **2. 为什么地图增量（1Hz）而不是全量同步？**
>
> 全量占据栅格地图可能是 100MB+，每 1 秒全量发送意味着每个机器人需要 100 MB/s 的上行带宽——在机器人 WiFi 环境（通常 20-50 Mbps = 2.5-6 MB/s）中不可能实现。
>
> 增量同步只发送变化的栅格单元（通常 < 1% 的地图每帧变化），数据量降低两个数量级。这里的关键权衡是：增量同步需要额外的状态追踪（哪些栅格更新过），换取带宽可行性。
>
> **3. 为什么用图优化融合而不是直接覆盖？**
>
> 两个机器人在同一个区域建图时，由于里程计漂移，两个局部地图在坐标系上存在偏差。直接覆盖（谁后到谁覆盖）会导致地图在边界处撕裂。
>
> 图优化（如 g2o/GTSAM）将每个机器人的轨迹作为图节点，将"机器人 A 和 B 在同一地点看到了相同特征"作为约束边。通过优化这些约束，可以在数学上最小化全局误差——这是一种 **概率方法**，承认每个传感器的观测都有噪声。
>
> **4. 为什么选择 Lamport 逻辑时钟而不是物理时钟？**
>
> 物理时钟看似直观："给每条消息打上时间戳，按时间排序"。但在分布式系统中，每个机器人的物理时钟（晶振）有微小偏差——1 分钟的累积偏差可达到微秒到毫秒级别。NTP 同步能降到毫秒级，但在 30Hz 的传感器数据（33ms 间隔）下，毫秒级的时钟偏差足以导致消息乱序。
>
> Lamport 逻辑时钟不依赖物理时间，而是维护一个 **因果计数器**：
> - 本地事件：`clock += 1`
> - 收到消息：`clock = max(local_clock, msg_clock) + 1`
>
> 这保证了如果 A 因果先于 B，则 A 的逻辑时钟一定小于 B（但不能反推）。对于地图融合，我们需要的正是这种因果顺序。
>
> **5. 为什么分区恢复后需要专门的同步协议而不是简单重放？**
>
> 简单重放所有缓存消息有两个问题：
> - 其他机器人也在发消息，会产生冲突（两个机器人声称同一区域被不同的障碍物占据）
> - slam 优化是一个状态积累过程，重放最后一个快照会丢失优化过程中的中间推理
>
> 正确的做法是：分区恢复后，将分区期间积累的地图作为一个整体（delta）提交，由融合节点做整体概率融合。这避免了逐条重放的冲突问题。

---

**练习 1.6：实时控制系统设计**
设计一个 1kHz 机械臂控制回路，要求：
- 最大延迟 < 1ms
- 抖动 < 100μs
- 与 ROS1 导航栈集成
- 支持力控制模式

> [!faq]- 📝 参考答案：实时系统的"为什么不能"思维
>
> **先理解"为什么 ROS1 不能满足实时要求"**：
>
> ROS1 的 TCPROS 协议栈中，单次消息传递路径上存在多种不可控的延迟源：
> 1. **内核调度**：Linux 默认的 CFS（完全公平调度器）可能在任意时刻将发送/接收线程换出
> 2. **TCP 重传**：即使局域网几乎不丢包，TCP 的 Nagle 算法会在小消息时故意延迟几十毫秒来"攒"满一个 MSS
> 3. **内存换页**：如果 ROS 节点的内存页被换出到 swap，换回可能需要数十毫秒
> 4. **中断处理**：网卡中断可能与控制线程争抢同一个 CPU 核心
>
> 在 1kHz 的控制回路中，每个周期只有 **1000μs**。上述任何一个事件都可能导致延迟超标。
>
> ---
>
> **架构设计的每个决策及原因**：
>
> **决策 1：为什么是"分层架构"而不是"把一切放在实时域"？**
>
> 把所有代码都放在实时域是最直观的思路——但不是正确的。实时操作系统不是魔法：它通过牺牲整体吞吐量来换取确定性的延迟。如果导航规划器（可能进行数百万次碰撞检测计算）运行在实时域，它会长时间霸占 CPU，导致其他实时任务（如力控反馈）错过截止时间。
>
> 正确的设计是将系统按"时间紧迫性"分层：
> ```
> [用户空间 - Linux CFS]                   [实时空间 - SCHED_FIFO]
>  ROS 导航栈 (100Hz)        共享内存       RT 控制器 (1kHz)
>  ┌──────────────────┐    ──────────>    ┌──────────────────┐
>  │ 全局路径规划      │   轨迹点队列       │ 插值 + PID 控制  │
>  │ 代价地图          │   <──────────    │ 力控导纳控制     │
>  │ 行为树            │   状态反馈         │ 前馈力矩计算     │
>  └──────────────────┘                   └────────┬─────────┘
>                                                   │
>                                          EtherCAT (总线周期 100μs)
>                                                   │
>                                          ┌────────┴─────────┐
>                                          │   电机驱动器       │
>                                          │   力传感器         │
>                                          └──────────────────┘
> ```
>
> **为什么这个分层是"对"的？** 导航规划（100Hz）需要毫秒级响应，但不要求微秒级抖动——偶尔晚 1ms 到达对整体路径质量影响极小。它们适合在普通 Linux 中运行，利用 CFS 的吞吐量优势。而电机控制（1kHz）的每次延迟都会直接转化为关节振动和位置误差——它们需要实时域的确定性保证。
>
> **决策 2：为什么用共享内存而不是话题/服务？**
>
> ROS1 的话题和服务都建立在 TCPROS 之上，这意味着触发了前面讨论的所有不确定性。共享内存避开了整个网络协议栈——一个 `std::atomic` 的 `store` 操作在 x86 上是单条指令，延迟在纳秒级。
>
> 但共享内存引入了一个新问题：**如何保证读写不冲突？** 无锁设计是关键：
> ```cpp
> struct RTControlData {
>     std::atomic<double> target_pos[7];   // 原子操作，无需锁
>     std::atomic<uint64_t> timestamp;     // 版本号
> };
> ```
> 规划器用 `store` 写入，控制器用 `load` 读取。因为 `std::atomic<double>` 在 x86 上是无锁实现，双方永不等锁。时间戳用来检测"规划器是否在写了一半时被调度出去"——如果 timestamp 在读前后不一致，意味着部分数据无效，丢弃本次读取。
>
> **决策 3：为什么用 EtherCAT 而不是其他总线？**
>
> CAN 总线（1 Mbps）的带宽不够传输 7 个关节的高频力矩数据。EtherCAT 提供 100 Mbps+ 的带宽和硬件级别的时间同步（分布式时钟节点误差 < 1μs）。更重要的是，EtherCAT 的"飞读飞写"（on-the-fly）模式消除了传统以太网的帧排队延迟——数据帧在通过每个从站时被硬件实时处理，不需要软件栈参与。
>
> **决策 4：为什么 CPU 亲和性设置到"独占核心"？**
>
> Linux 的 CPU 调度器不知道你的线程是实时关键的——它可能在你的控制线程正在计算逆动力学时，把另一个 CPU 核心上的浏览器渲染任务迁移过来"借用"这个核心几个周期。设置 CPU 亲和性（`pthread_setaffinity_np`）+ 从内核引导参数中隔离该核心（`isolcpus=3`），确保没有任何东西能抢占这个核心。这是"物理隔绝"级别的保证。
>
> **决策 5：为什么力控制用导纳控制（Admittance）而不是阻抗控制（Impedance）？**
>
> 阻抗控制：测量位移，输出力 → 适合轻量、高刚度的任务
> 导纳控制：测量力，输出位移 → 适合与刚性环境交互（如装配）
>
> 在 PR2/工业机械臂场景，机器人通常是刚性结构，需要与刚性环境（机床、工件）接触。导纳控制的"测量力 → 修正位置"模型更自然：当力传感器检测到超过期望的接触力时，控制算法计算出位置修正量来"让步"——保持力的稳定而不是位置的精确。
>
> ---
>
> **关键性能验证**：
> | 指标 | 要求 | 方案保证机制 |
> |------|------|------------|
> | 延迟 < 1ms | 控制循环 1kHz | RT-PREEMPT 内核 + SCHED_FIFO + CPU 独占 |
> | 抖动 < 100μs | 精确时序 | `mlockall` 防止换页 + 预分配内存 |
> | 与导航栈集成 | 双向数据交换 | 无锁共享内存 + 轨迹插值 |
> | 力控制 | 柔顺交互 | 导纳控制 + EtherCAT 高速力传感器读取 |

---

## 常见陷阱与错误（Gotchas）

### 1. 网络配置错误

**问题**：多机通信时节点无法互相发现
```bash
# 错误配置
export ROS_MASTER_URI=http://192.168.1.100:11311
export ROS_HOSTNAME=localhost  # 错误！其他机器无法解析
```

**解决方案**：
```bash
# 正确配置
export ROS_MASTER_URI=http://192.168.1.100:11311
export ROS_IP=192.168.1.101  # 使用实际 IP
# 或
export ROS_HOSTNAME=robot1  # 确保所有机器的 /etc/hosts 中有此条目
```

### 2. 话题名称不匹配

```cpp
// 节点 A
ros::Publisher pub = nh.advertise<std_msgs::String>("/robot/status", 10);

// 节点 B
ros::Subscriber sub = nh.subscribe("/robot_status", 10, callback);  // 拼写错误！
```

**调试技巧**：
```bash
# 检查话题连接
rostopic info /robot/status
rosnode info /node_name
rqt_graph  # 可视化节点连接
```

### 3. 消息类型版本不一致

自定义消息修改后，忘记重新编译所有依赖包。

**预防措施**：
```bash
catkin clean
catkin build
# 或
catkin build --force-cmake
```

### 4. 回调队列阻塞

**问题**：在回调函数中执行耗时操作会阻塞整个回调队列。

**正确方式**：将耗时任务推入处理队列，由独立线程处理。

### 5. 参数服务器竞态条件

多个节点同时读写同一参数时出现竞态条件。

**解决方案**：使用分布式锁或改用服务实现原子操作。

### 6. tf 时间戳问题

```cpp
// 错误：使用当前时间查询历史变换
listener.lookupTransform("map", "base_link", ros::Time::now(), transform);

// 正确：使用 Time(0) 获取最新可用变换
listener.lookupTransform("map", "base_link", ros::Time(0), transform);
```

---

## 最佳实践检查清单

### 系统设计审查

- [ ] **单点故障分析**：识别所有单点故障（Master、关键节点），设计故障恢复机制，实施健康检查和自动重启
- [ ] **性能需求评估**：明确延迟和带宽要求，选择合适的通信模式，考虑是否需要实时性保证
- [ ] **扩展性设计**：节点功能单一职责，使用命名空间组织话题，预留配置和接口扩展点

### 开发实践

- [ ] **消息设计**：优先使用标准消息类型，自定义消息保持向后兼容，避免过度嵌套的消息结构
- [ ] **节点实现**：实现优雅关闭（SIGINT 处理），添加诊断信息发布，使用 ROS 日志系统
- [ ] **参数管理**：使用 YAML 文件组织参数，实施参数验证，支持动态重配置（如适用）

### 测试策略

- [ ] **单元测试**：使用 rostest 框架，模拟外部依赖，测试异常情况
- [ ] **集成测试**：测试节点间通信，验证时序要求，测试网络故障恢复
- [ ] **性能测试**：测量消息延迟，监控 CPU 和内存使用，压力测试（高频率、大消息）

### 部署准备

- [ ] **文档完善**：README 包含依赖和构建说明，记录所有话题/服务/参数，提供 launch 文件示例
- [ ] **配置管理**：环境相关配置外部化，使用 roslaunch 参数覆盖，版本控制配置文件
- [ ] **监控部署**：配置诊断聚合器，设置日志轮转，实施性能监控

### 安全考虑

- [ ] **网络安全**：限制 Master 访问（防火墙），使用 VPN 跨网络通信，验证输入数据合法性
- [ ] **故障安全**：实施紧急停止机制，添加传感器数据合理性检查，设计降级运行模式

---

> 通过遵循这个检查清单，可以构建更加健壮、可维护的 ROS1 系统，同时为将来迁移到 ROS2 打下良好基础。
