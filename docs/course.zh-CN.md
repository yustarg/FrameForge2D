# FrameForge · 七天 C++ 引擎与并发专项

## 课程目标

用七天完成一个能运行、能观察、能排查问题的小型 2D 引擎。重点是理解一帧怎样执行，内存和资源怎样活着，线程之间怎样交换工作和结果。

按每天 3–4 小时设计，共约 24 小时。第 3–6 天以并发为主，第 7 天综合验证。这个时间预算适用于已经写过 C++、愿意在课内补齐标准库用法的学习者。

场景里有一批移动的图形和少量父子节点，可以增减数量、暂停、切换单线程/多 worker，并在运行时加载另一份场景配置。窗口显示最新已完成的结果，后台工作可以被观察和人为延迟。

最终应能独立解释以下问题。

- 一帧里哪些工作在主线程，哪些在 simulation 线程和 worker 上。
- 什么数据可以共享，什么数据只属于一个任务或一个角色。
- 为什么输入要有快照，为什么一个计算结果需要明确的完成时刻。
- 资源卸载、任务取消、引擎退出时，怎样保证正在运行的工作仍然安全。
- 怎样从时间线、分配记录和帧号定位等待、内存增长和错误的数据交接。

## 技术选择与本机环境

- 使用 C++20 和 CMake。Windows 使用 Visual Studio/MSVC。
- SDL 3.2.28 只承担窗口、输入和简单 2D 绘制。
- 引擎循环、场景数据、线程池、任务依赖、快照交换、资源加载和统计由课程代码实现。
- 核心库可以无窗口运行，用来复现故障、检查结果和测量性能。
- 本机已发现 Visual Studio 18 Community、MSVC 14.51.36231 和 Visual Studio 自带的 CMake 4.3.1。命令行 PATH 当前没有找到 cmake 或编译器，课程开始时使用 VS 的开发环境。
- SDL 选择已发布的固定版本，开始第 1 天时建立依赖和可复现构建。

## 最终架构

主线程负责 SDL 事件与绘制，simulation 线程拥有可变世界，worker 执行独立计算，loader 承担文件读取和解析。

```text
主线程
  读取输入 → 发送命令 → 获取最新完整快照 → SDL 绘制
                ↓              ↑
simulation 线程                │
  tick 边界接收命令            │
  固定步长推进世界              │
  派发更新任务 → 等待本阶段完成 │
  计算父子变换 → 生成只读快照 ──┘
        ↓
worker pool
  在独占的数据范围内计算
  报告完成和异常

loader 线程
  读取与解析场景 → 发送完成结果 → simulation 在 tick 边界提交
```

这是一套小引擎的线程安排。大型引擎的渲染线程和 GPU 队列另有职责。课程中的 SDL 绘制始终留在主线程。

## 每天的学习方式

每天以一个画面或故障开始。

1. 用 30–45 分钟讲清当天的数据流和一两个原理，画出谁拥有、谁读、谁写。
2. 用 90–120 分钟分步实现。每步先预测运行结果，再编译观察。
3. 用 30–45 分钟做一个故障实验和对应验证，保存关键结果。
4. 用 10–15 分钟口述当天的设计，用自己的话解释原因。

学到新 C++ 语法时，用当天的数据寿命和线程问题解释它。每个改动都需要能说明用途。

## Day 1 · 一帧究竟做了什么

当天的问题是“窗口里的物体，怎样每秒稳定地移动，同时接收输入和绘制？”

### 原理

- main loop、tick、固定时间步长和渲染频率。
- 用累计时间决定本次执行几次固定更新，处理暂停和异常长帧。
- 场景状态、输入命令、绘制数据的区别。
- local transform 和 parent transform，父先子后的更新顺序。
- RAII 怎样保证窗口和引擎子系统按顺序释放。

### 当天实现

- 建立 CMake 工程，分成 frameforge_core、frameforge_demo 和无窗口验证入口。
- 实现窗口、输入、固定步长、暂停、单步和有序退出。
- 使用连续容器保存物体的位置、速度和颜色。
- 加入一组简单父子节点，让父节点转动时子节点跟随。
- 记录 tick 编号、更新次数、帧时间和物体数量。

### 故障实验

人为制造一个长帧，观察按渲染帧直接移动和按固定 tick 推进的行为差异。长帧处理策略明确设置追帧上限，并记录实际执行的模拟步数。

### 完成标准

- 相同初始数据和相同 tick 数得到可重复的场景状态。
- 暂停和单步行为一致，父子变换可以手算核对。
- 窗口与引擎资源能够正常释放。
- 能解释为什么“画一次”和“模拟一步”可以是不同频率。

## Day 2 · 谁申请内存，谁把它释放

当天的问题是“每帧临时数据应该存在哪，为什么清空容器后进程内存可能不降？”

### 原理

- 栈、堆、对象寿命、作用域、所有权和借用。
- vector 的 size/capacity，扩容和指针失效。
- unique_ptr、shared_ptr 和引用的适用关系。
- 分配、释放、分配器保留内存、缓存和碎片。
- 持久数据与帧临时数据，线性/arena 分配。
- 对齐、连续内存和缓存局部性。

### 当天实现

- 给引擎控制的容器接上带标签的统计分配器，统计当前字节、峰值、累计分配次数和累计分配字节。
- 使用 std::pmr::memory_resource 做统计入口，使用标准 monotonic_buffer_resource 观察 arena 行为。
- 按 World、FrameScratch、Assets 等用途分类。
- 为更新阶段的临时列表预留并复用空间。
- 给帧临时数据规定失效时刻，后续任务开始后必须等所有使用者完成才能重置。

### 故障实验

让一个容器每帧扩容或反复申请临时数据，观察分配次数。再复用空间，记录变化。反复创建和销毁场景，区别 live bytes、容器 capacity 和操作系统看到的进程内存。

### 完成标准

- 可以指出每份主要数据的所有者和寿命。
- 正确释放指定范围的对象后，统计范围内的 live bytes 回到预期基线。
- 能说明高水位和累计分配量分别表示什么。
- 明确自建统计只覆盖接入的分配，SDL、GPU 和未接入的系统分配需要其他工具观测。

## Day 3 · 自己实现一个线程池

当天的问题是“把一批工作交给后台执行，怎样睡眠等待、接收结果并可靠退出？”

### 原理

- thread、task、thread pool、并发与并行。
- mutex、lock_guard/unique_lock、condition_variable 和谓词等待。
- 队列里的任务与正在执行的任务，完成计数和异常传播。
- 为什么唤醒后还要检查条件，为什么锁内工作要短。
- shutdown 时停止接收新任务、排空已接收任务、唤醒并 join。

### 当天实现

- 固定数量的 worker、一个受 mutex 保护的任务队列和 condition_variable。
- 提交任务、取得结果、统计 queued/running/completed。
- 任务异常可以传回提交方。
- 明确退出契约，shutdown 开始后拒绝新提交，已接收任务执行完成后 worker 退出。
- 本周任务保持有界计算，退出时间受在途任务时间影响。

### 故障实验

在队列非空时退出，在 worker 睡眠时退出，在某个任务抛异常时退出。故障代码单独保留成可控实验，修复后的默认入口保持正确。

### 完成标准

- 已接收的每个任务执行一次，结果和异常有明确去向。
- 关闭时没有丢失已接收任务，没有等待永远不会发生的条件。
- 所有线程被 join，worker 退出之前其访问的数据仍然有效。
- 能画出 mutex 保护的状态，以及 wait 的谓词。

## Day 4 · 把引擎的一帧拆成任务

当天的问题是“1000 个物体怎样并行更新，哪些计算必须等别人？”

### 原理

- 任务依赖图、数据依赖、执行顺序和完成同步。
- 拆分连续数据范围，保证每个输出元素只有一个任务写。
- vector 扩容、结构修改和任务中的引用为何会冲突。
- 并行更新父子层级时的父先子后约束。
- 任务粒度、调度开销、负载不均、false sharing 和等待。

### 当天实现

- 将移动更新分成连续批次，线程数量可配置。
- 加入 TaskGroup，simulation 协调方等待阶段完成。
- 使用明确的阶段依赖“独立更新 → 层级变换 → 快照生成”。
- 层级部分先保持父先子后的简单实现；多个独立根的子树可作为并行单元。
- 结构变化只在阶段边界进行，任务执行期间容器尺寸与资产引用保持稳定。
- worker 执行叶子计算，完成后归还。等待依赖的工作由协调方在正确阶段派发。
- 将内存统计升级为安全的并发计数或线程局部统计后汇总。共享的上游分配器遵守自己的线程安全契约；monotonic_buffer_resource 由单个任务或 worker 独占，所有借用结束后再重置。

### 故障实验

在一个任务里故意加长计算，观察其他任务结束后整帧为什么还在等。改变 batch 大小，对比调度成本和尾部等待。

### 完成标准

- 固定输入与 tick 数下，串行和并行结果按规定精度匹配。
- 可用 1、2、4 个 worker 跑同一工作量并解释测量结果。
- 物体增删和容器扩容发生在约定边界。
- 能说明增加线程何时有收益，以及最快结束的任务为何不能决定帧完成时间。

## Day 5 · 两个线程怎样交接一帧结果

当天的问题是“模拟正在修改世界，窗口怎样稳定地显示完整一帧？”

### 原理

- data race、happens-before、可见性和对象寿命。
- 引用计数、指针本身的同步、对象内部数据的同步分别解决什么。
- 发布后只读的快照，release/acquire 的发布与获取关系。
- frame ID、重复显示上一份完整结果，以及输入延迟。
- 双缓冲/三缓冲的复用条件与消费者持有时间。

### 当天实现

- 将可变世界交给 simulation 线程独占。
- 主线程通过 mutex 保护的命令队列发送输入，simulation 在 tick 边界应用。
- 为命令队列规定容量和拥塞策略，可合并的连续输入保留最新值，切关等离散命令保留顺序并显式处理提交失败。暂停时协调线程仍然消费恢复和退出命令。
- simulation 等计算完成后，创建完整 FrameSnapshot，发布后保持只读。
- 先用受 mutex 保护的 shared_ptr 交换验证所有权和寿命。
- 再把同一个交接点改为 std::atomic<std::shared_ptr<const FrameSnapshot>>，明确 store-release/load-acquire。
- 主线程取得快照后持有自己的引用，SDL 事件和绘制仍在主线程。
- 测量快照分配和复制成本，比较两个发布版本。atomic shared_ptr 的实现可能内部加锁。

### 故障实验

让模拟和显示的速度明显不同，给两边加入可重复的延迟。观察 main 只显示完整 frame ID，以及旧快照何时真正释放。给出固定槽位双缓冲过早复用的故障图，分析如何防止生产者追上读者。

### 完成标准

- 渲染路径只读取已完成快照，simulation 对发布后的对象保持只读。
- 任何旧快照都有明确读者，最后一个读者结束后才回收。
- 能解释发布指针的原子性、数据可见性和寿命是三件事。
- 能解释仅把 currentBuffer 指针变成 atomic 为何无法单独保证槽位复用安全。

## Day 6 · 异步加载、取消和安全卸载

当天的问题是“后台读取新场景时，旧场景还在跑；玩家又换了一次场景，旧请求怎么办？”

### 原理

- 异步完成顺序与请求发出顺序可能不同。
- 文件读取、解析、验证、提交到世界是不同阶段。
- 请求版本/代际、合作式取消和安全回收。
- 旧资产被在途任务或旧快照持有时，回收如何延迟。
- I/O 工作与计算任务池的边界。

### 当天实现

- 增加一个 loader 线程，读取简单文本场景并产生经过验证的只读 SceneDefinition。
- 每次请求分配递增 request ID，完成结果通过线程安全队列送回。
- simulation 只在 tick 边界提交当前仍有效的请求结果。
- 为过期请求提供原子取消标记，解析阶段可以检查标记。阻塞文件 I/O 的即时中断受所用 API 限制，过期结果始终可以被丢弃。
- 用共享只读资产引用保证在途工作安全，提交替换资源时保留现有读者的引用。
- 退出时先停止新请求，处理 loader 和 simulation 的在途工作，最终 join 全部线程。

### 故障实验

快速发出 A、B 两个请求，故意让 A 更晚完成，确认最终仍显示 B。反复切换场景，并在加载途中关闭窗口。观察 live bytes、资源代际和工作线程状态。

### 完成标准

- 过期加载结果不能覆盖新场景。
- 加载失败保留一份有效世界，错误能在主线程显示或打印。
- 在途任务持有的资源不会提前释放，旧资源在最后一个读者结束后可回收。
- 可以解释为什么“请求取消”和“允许释放内存”需要分别判断。

## Day 7 · 用证据解释引擎怎样运行

当天的问题是“引擎能跑，但我怎样证明它正确，怎样找到慢和内存增长的原因？”

### 原理

- CPU 总工作量、帧关键路径和等待。
- 统计平均值、p95/p99、峰值和分配频率。
- 数据布局和批大小改变的收益与成本。
- 同一输入在单线程/多线程下的结果对照。
- AddressSanitizer 与 ThreadSanitizer 的检测范围和平台差异。

### 当天实现

- 给 tick、batch、等待、快照发布、绘制和加载写入带线程 ID 的时间线事件。
- 导出能被 Perfetto 打开的 JSON trace，worker 各自记录局部事件，安全汇总，退出时生成完整文件。
- 显示或打印 frame ID、物体数、任务数、等待时间和内存分类。
- 跑固定输入、固定 tick 数的 headless 场景，完成串行/并行对照。
- 对 1/2/4 worker、不同 batch 大小和不同物体数量记录同样的测量。
- 在 Windows 下使用适用的 AddressSanitizer 配置检查内存问题。若已有合适的 Linux/WSL Clang 环境，再单独运行 ThreadSanitizer 核心用例。ASan 不能代替 race 检测，TSan 一次通过也不能证明所有调度都安全。

### 综合完成标准

- 连续切换场景后，受统计内存的存活量回到对应基线，缓存保留项有明确解释。
- 在模拟和渲染速度不同、加载失败、任务异常、退出等场景下，线程和资源生命周期保持一致。
- 固定输入下串行和并行状态符合约定的一致性要求。
- 保存一份 trace 和一份测量表，能指出一个实际瓶颈并说明改动依据。
- 能在五分钟内画出架构，解释谁拥有世界、谁写快照、谁还持有旧资产。

## 最终代码组织

目录按功能逐天增加，未实现的功能不预建空类。

```text
FrameForge/
  CMakeLists.txt
  README.md
  docs/course.zh-CN.md
  src/
    core/           固定时间步长、统计内存、时间线
    scene/          世界数据、变换层级、场景定义
    jobs/           worker pool、TaskGroup
    runtime/        simulation、命令、FrameSnapshot
    assets/         loader、请求代际、完成队列
    platform/       SDL 窗口、事件和绘制
  app/              可交互 demo
  checks/           headless 验证和故障复现入口
  data/             小型文本场景
  captures/         本机运行产生的测量和 trace
```

CMake 的 build 目录保持独立。生成文件与源码的边界从第 1 天开始明确。

## 本周完成后的面试叙事

可以用以下内容组织英文回答，实际成绩和负责范围以完成后的代码及测量为准。

> I built a small C++ runtime to understand how an engine schedules a frame and manages data across threads. The simulation owns the mutable world, workers update independent batches, and the main thread renders immutable snapshots. I also added asynchronous scene loading, tracked allocations, and task tracing. The most useful part was debugging shutdown and resource lifetime while work was still in flight.

需要能顺着回答的追问包括以下内容。

- 为什么选择固定步长，渲染跟不上时会发生什么。
- 哪些数据共享，哪些数据由每个任务独占。
- 为什么有空闲 worker，simulation 仍然可能在等待。
- 为什么 atomic pointer 无法独自保证对象寿命。
- 带引用计数的数据仍有哪些线程安全问题。
- 任务取消以后，什么时候可以销毁输入和输出。
- 内存增长属于 live allocations、分配器保留还是缓存。
- 单线程可能更快的原因，以及测量怎样支持判断。

## 参考资料

- [SDL 主线程事件约定](https://wiki.libsdl.org/SDL3/SDL_PollEvent)
- [SDL 主线程绘制约定](https://wiki.libsdl.org/SDL3/SDL_RenderPresent)
- [SDL 3.2.28 固定版本](https://github.com/libsdl-org/SDL/releases/tag/release-3.2.28)
- [C++ 标准草案中的 atomic shared_ptr](https://eel.is/c++draft/util.smartptr.atomic.shared)
- [Unreal Tasks System](https://dev.epicgames.com/documentation/unreal-engine/tasks-systems-in-unreal-engine)
- [Unreal 线程间渲染数据所有权](https://dev.epicgames.com/documentation/unreal-engine/threaded-rendering-in-unreal-engine)
- [Unreal Memory Insights](https://dev.epicgames.com/documentation/unreal-engine/memory-insights-in-unreal-engine)
- [Clang ThreadSanitizer 支持范围](https://clang.llvm.org/docs/ThreadSanitizer.html)
