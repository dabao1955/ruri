# 为类 chroot 容器添加 systemd 支持：深层技术思考

> 这篇文章不讲代码，只讲思考。
> 为什么要这样做？背后的技术原理是什么？有哪些权衡？
> 希望这些思考能帮助你理解容器与 init 系统的本质。

---

## 目录
- [核心问题：systemd 为什么不能直接跑在 chroot 里](#核心问题systemd-为什么不能直接跑在-chroot-里)
- [PID 1 的深层含义](#pid-1-的深层含义)
- [为什么不让 systemd 直接当 PID 1](#为什么不让-systemd-直接当-pid-1)
- [信号处理的本质](#信号处理的本质)
- [进程收割的困境](#进程收割的困境)
- [cgroup delegation 的技术原理](#cgroup-delegation-的技术原理)
- [运行时环境的构建逻辑](#运行时环境的构建逻辑)
- [架构设计的决策链](#架构设计的决策链)
- [实现中的关键技术选择](#实现中的关键技术选择)
- [验证与反思](#验证与反思)

---

## 核心问题：systemd 为什么不能直接跑在 chroot 里

### 表层现象与深层原因

最开始的问题是：用户想在 ruri 容器里运行 systemd，但直接执行会失败。

**表层理解**：systemd 需要一些特殊配置。

**深层理解**：systemd 不是一个普通进程，它是一个"系统初始化管理器"。它的设计假设自己是整个系统的起点，拥有对系统资源的完全控制。

### 关键矛盾点

当在宿主机的 PID 命名空间里运行 systemd 时：

| 假设 | 现实 | 矛盾 |
|------|------|------|
| 我是 PID 1 | 宿主机的 init 才是 PID 1，我是 PID 1234 | systemd 的权限模型崩溃 |
| 我控制所有进程 | 我只能控制自己 fork 的进程 | 孤儿进程处理失效 |
| 我管理所有 cgroup | 宿主机的 cgroup 已经被占用 | 资源管理失败 |
| /run 是我的 | /run 被宿主机和其他容器共享 | 状态冲突 |

**思考**：如果容器和宿主机都运行 systemd，它们会争夺对 /run、/sys/fs/cgroup 的控制权。这就像两个 init 系统在同一个机器上运行——必然冲突。

### 解决方案的宏观思考

需要创建一个"虚拟的 init 环境"：
1. 让容器内的第一个进程成为 PID 1（通过 PID 命名空间）
2. 让容器有独立的 /run（通过 tmpfs 挂载）
3. 让容器有独立的 cgroup 层级（通过重新挂载 cgroup2）
4. 让这个环境对 systemd 来说"看起来像一个完整的系统"

**核心洞察**：systemd 不关心它是不是真的在管理整个物理机，它只需要一个"看起来像系统"的环境。我们可以用 namespace + 挂载的组合来创建这个环境。

---

## PID 1 的深层含义

### 为什么是 PID 1 而不是 PID 2？

PID 1 在 Linux 中有特殊地位：
- 它是所有孤儿进程的默认父进程
- 它拥有某些特权（如可以发送信号给任何进程）
- 它不会被 SIGKILL 杀死（只有 SIGKILL 能杀死普通进程，但 PID 1 会忽略它）

**关键问题**：如果 systemd 是 PID 2，某个服务变成孤儿，父进程退出，这个孤儿会被 parent 到 PID 1。但 PID 1 是宿主机 init，它不知道这个进程是 systemd 的服务，也不会按照 systemd 的逻辑处理它。

### PID 命名空间的隔离原理

PID 命名空间创建了一个独立的 PID 空间，关键特性：
- 空间内的进程看不到空间外的进程
- 空间外的进程可以看到空间内的进程（通过 /proc/[pid]/ns/pid）
- 空间内的第一个进程自动成为 PID 1
- 空间内的 PID 1 退出时，整个命名空间会被销毁

**设计决策**：
- 我们不能让 systemd 直接当 PID 1，因为需要在 systemd 启动前做一些准备工作
- 我们需要一个"init shim"来当 PID 1，然后 fork + exec systemd
- shim 负责：准备工作 → 启动 systemd → 收割僵尸 → 传递信号 → 传递退出码

---

## 为什么不让 systemd 直接当 PID 1

这是一个关键的技术决策。很多人会想：既然 systemd 需要成为 PID 1，那直接让它在 PID 命名空间里作为第一个进程启动不就行了？为什么还要一个 init shim？

### 方案 1 的问题：让 systemd 直接当 PID 1

**表面上的做法**：
1. 创建 PID 命名空间
2. 在新命名空间里直接 exec systemd
3. systemd 成为 PID 1

**看起来很简单，但问题在哪里？**

#### 问题 1：环境准备必须在 systemd 启动前完成

systemd 启动前需要准备环境：
- 挂载 /run 和 /tmp 的 tmpfs
- 创建 /run/systemd 目录结构
- 生成 /etc/machine-id
- 设置 cgroup delegation
- 确保 /dev/console 存在

**关键点**：这些准备工作必须在容器内（chroot 后）完成。如果在 chroot 之前完成，文件会创建在宿主机的根目录，而不是容器的根目录。

**矛盾**：
- 要准备环境，必须先 chroot 到容器
- 但 chroot 后，如果直接 exec systemd，它就成为了 PID 1
- 如果 exec 之前还有其他操作（如挂载），这些操作会在 chroot 后的容器内进行

**结论**：需要有一个进程在 chroot 后、systemd 前执行，这个进程必须成为 PID 1。

#### 问题 2：systemd 退出后的清理

systemd 退出后会发生什么？
- PID 命名空间内的所有进程会被杀死（因为 PID 1 退出了）
- 但某些清理工作需要在容器退出前完成

例如：
- 如果容器使用了 rootfs 镜像，可能需要卸载
- 可能需要记录日志
- 可能需要通知父进程（ruri 的主进程）

**结论**：需要一个进程在 systemd 退出后继续运行，做清理工作，然后正确退出。

#### 问题 3（最关键）：宿主机的 init 会崩溃

这是最容易被忽视但最致命的问题。

**场景**：如果 systemd 直接作为 PID 1 运行在某些特殊情况下：

```
容器启动：
1. 父进程创建 PID 命名空间
2. 在 PID 命名空间内 chroot
3. exec systemd（成为 PID 1）

systemd 启动后：
- systemd 发现自己在 PID 1
- systemd 开始执行初始化逻辑
- systemd 可能发送某些系统级信号或执行某些特权操作
```

**问题**：PID 命名空间虽然是独立的，但某些系统资源是共享的。例如：

1. **/proc 挂载**：如果 /proc 没有正确挂载为新的 proc 文件系统，systemd 读取的 /proc/1/... 实际上是宿主机的 init！

2. **信号广播**：某些信号（如 SIGINT、SIGTERM）可能会被广播到整个系统或共享 tty。

3. **特权操作**：systemd 作为 PID 1 会尝试执行一些只有真正的系统 init 才能执行的操作（如设置系统的运行级别、重启系统等）。

**具体崩溃场景**：

假设 systemd 尝试重启系统：
```
systemd 执行 reboot() 系统调用
→ 调用 kernel 的 sys_reboot()
→ kernel 检查调用者是否是 init
→ 在 PID 命名空间内，systemd 确实是 PID 1
→ kernel 可能认为这是合法的系统重启请求
→ 宿主机的 init 收到 SIGTERM 或其他信号
→ 宿主机的 init 开始关机流程
→ 整个系统（包括宿主机）开始关闭
```

**实际案例**：在早期的 Docker 中，如果容器内的进程直接作为 PID 1 并执行某些特权操作，确实会导致宿主机重启或崩溃。

**根本原因**：
- Linux 内核的某些系统调用检查 PID 而不是 namespace
- PID 1 的某些特权是全局的，不是 namespace 内的
- systemd 设计时假设自己是真正的系统 init，会尝试执行系统级操作

#### 问题 4：信号处理

systemd 作为 PID 1 会：
- 安装自己的信号处理函数
- 屏蔽某些信号
- 期望自己控制所有信号

如果直接运行 systemd，父进程（创建 PID 命名空间的进程）和 systemd 之间会有信号处理的冲突。

### 方案 2 的优势：init shim 作为中介

使用 init shim 作为 PID 1，然后 fork + exec systemd：

**架构**：
```
父进程（host）
    ├── 创建 PID 命名空间
    ├── fork
    └── 子进程（PID 1 in new namespace）
            ├── chroot
            ├── 准备环境
            ├── fork
            └── 孙进程（systemd）
                    └── 真正运行服务
```

**优势分析**：

1. **隔离 systemd 的特权**：
   - init shim 控制 systemd 的执行
   - systemd 不再是 PID 1，而是 PID 2
   - systemd 不会获得 PID 1 的某些特权
   - systemd 尝试执行系统级操作时，会被 init shim 拦截或忽略

2. **环境准备**：
   - init shim 可以在 fork systemd 之前准备所有环境
   - chroot、挂载 tmpfs、生成 machine-id 等都在 init shim 中完成
   - systemd 启动时，环境已经就绪

3. **信号管理**：
   - init shim 接收所有信号
   - init shim 决定哪些信号转发给 systemd，哪些自己处理
   - 防止 systemd 收到不应该收到的信号

4. **进程收割**：
   - init shim 作为 PID 1 负责收割僵尸进程
   - systemd 作为 PID 2 可以专注于服务管理
   - 孤儿进程被 init shim 收养，init shim 可以决定是否转给 systemd

5. **退出控制**：
   - systemd 退出时，init shim 捕获退出状态
   - init shim 可以做清理工作
   - init shim 控制退出码的传播

6. **防止宿主机 init 崩溃**：
   - systemd 不是 PID 1，无法执行真正的系统级操作
   - systemd 的重启、关机等请求会被限制在容器内
   - init shim 可以过滤或忽略危险的系统调用

### 技术验证

**验证 systemd 作为 PID 1 的危险性**：

如果在 PID 命名空间内直接运行 systemd，可以尝试：

```bash
# 在容器内
systemctl reboot
```

在某些配置下，这可能导致宿主机重启，因为：
1. systemd 可能通过 dbus 或其他机制与宿主机通信
2. systemd 可能执行特权系统调用
3. 如果 /proc 没有隔离，systemd 可能访问宿主机的 /proc/sys

**验证 init shim 的保护作用**：

使用 init shim 后：
- systemd 不再是 PID 1
- systemd 的重启请求会被 init shim 拦截
- init shim 可以选择忽略或仅退出容器

### 设计决策总结

| 对比项 | 方案 1：systemd 直接 PID 1 | 方案 2：init shim + systemd |
|--------|---------------------------|----------------------------|
| 环境准备 | 困难，必须在 exec 前完成 | 简单，fork 前准备 |
| 宿主机安全 | 有风险，可能导致崩溃 | 安全，init shim 隔离 |
| 信号控制 | 不可控，systemd 独占 | 可控，init shim 转发 |
| 进程收割 | 混乱，systemd 可能无法处理 | 清晰，init shim 统一处理 |
| 退出控制 | 不可控，直接退出 | 可控，init shim 做清理 |
| 代码复杂度 | 简单，但风险高 | 稍复杂，但安全可控 |

**最终决策**：选择方案 2，init shim 是必须的，不能因为看似"简单"而冒险。

---

## 信号处理的本质

### 为什么信号处理如此困难？

信号是异步的——它们可以在任何时刻到达，打断当前的执行流。传统的信号处理函数（signal handler）有几个固有问题：

1. **竞态条件**：信号可能在检查条件和处理之间到达
2. **可重入性**：信号处理函数必须是 async-signal-safe 的，限制很大
3. **丢失信号**：如果信号来得太快，可能会合并或丢失
4. **SIGCHLD 的特殊性**：多个子进程同时退出时，只收到一次信号，但需要处理多个子进程

### signalfd 的革新意义

signalfd 把信号从"中断机制"转变为"事件机制"：
- 信号被阻塞，直到显式读取
- 可以通过 select/poll/epoll 统一处理
- 处理逻辑是同步的，可以调用非信号安全的函数
- 可以一次性读取多个信号事件

**为什么不是默认方案？**

signalfd 需要 Linux 2.6.22+，某些嵌入式系统（旧版本 Android）可能不支持。因此需要回退机制：尝试 signalfd，失败则使用传统信号处理。

### 信号转发的设计

init shim 需要转发信号给 systemd：
- SIGHUP、SIGTERM 等：systemd 需要处理这些信号来做优雅关闭或重新加载配置
- SIGCHLD：这是给 init shim 的，不是给 systemd 的
- 信号值需要保持，不能转换

**技术细节**：siginfo_t 包含了发送进程的信息、发送原因等，简单的 kill() 转发会丢失这些信息。但对于 systemd 来说，信号值本身足够。

---

## 进程收割的困境

### 孤儿进程的产生

在容器内运行服务时：
1. systemd fork 出服务进程（PID 2 fork PID 3）
2. 服务进程 fork 出子进程（PID 3 fork PID 4）
3. 服务进程退出，PID 4 变成孤儿
4. 孤儿应该被 systemd 收养，但 systemd 的父进程是 init shim

**问题**：如果 init shim 不设置特殊标志，PID 4 会被 host 的 init（PID 1）收养，而不是 systemd。

### Child Subreaper 的解决方案

Linux 3.4+ 引入了 PR_SET_CHILD_SUBREAPER：
- 当一个进程设置这个标志后，它的孙进程变成孤儿时，会被 re-parent 到它
- 这样 systemd fork 的服务的孤儿，会被 systemd 收养
- 完美匹配 systemd 的期望

**为什么传统 init 不需要这个？**

传统 init（如 sysvinit）自己就是 PID 1，孤儿自动被它收养。systemd 在容器内不是真正的 PID 1（init shim 才是），所以需要这个机制。

### waitpid 的正确用法

init shim 需要不断调用 waitpid 来收割僵尸进程：
- 使用 WNOHANG 避免阻塞
- 循环调用直到返回 0（没有更多子进程）
- 区分 systemd 退出和其他子进程退出
- 记录退出码，特别是信号终止的情况

**退出码传播**：
- 正常退出：直接使用 exit code
- 信号终止：使用 128 + 信号编号（shell 的约定）

---

## cgroup delegation 的技术原理

### cgroup v2 的层级模型

cgroup v2 使用统一的层级结构：
```
/sys/fs/cgroup/
├── cgroup.procs           # 当前 cgroup 的进程
├── cgroup.subtree_control # 启用的控制器
├── cgroup.controllers     # 可用的控制器
├── cpu.stat              # cpu 统计
├── memory.stat           # memory 统计
└── systemd/              # 子 cgroup
    ├── cgroup.procs
    ├── cgroup.subtree_control
    └── ...
```

关键点：
- 控制器必须在父 cgroup 的 subtree_control 中启用，子 cgroup 才能使用
- 进程只能属于一个 cgroup（叶子节点）
- cgroup 是文件系统，可以重新挂载

### Delegation 的实现

在容器内实现 cgroup delegation：
1. 在容器内重新挂载 cgroup2（不影响 host）
2. 在 root cgroup 启用 cpu、memory、pids 控制器
3. 创建 /sys/fs/cgroup/systemd 子组
4. 在子组中再次启用控制器
5. 将容器进程移动到这个子组

**为什么能工作？**

容器有独立的 mount namespace，重新挂载 cgroup2 只影响容器内的视图。host 的 cgroup 仍然存在，但容器内的进程看到的是新的层级。

**权限问题**：
- 写入 subtree_control 需要 CAP_SYS_ADMIN
- 如果容器没有权限，会失败
- 但这是可接受的降级：systemd 仍然可以运行，只是不能管理资源

---

## 运行时环境的构建逻辑

### /run 和 /tmp 的特殊性

为什么必须是 tmpfs？

1. **非持久化**：systemd 在 /run 存储运行时数据（pid 文件、socket、状态），不应该持久化
2. **独立性**：每个容器需要独立的 /run，不能共享
3. **性能**：tmpfs 在内存中，访问更快
4. **自动清理**：容器停止时，/run 自动消失

**挂载选项**：
- nosuid：不允许 setuid 程序
- noexec：不允许执行
- nodev：不解释设备文件
- mode=755：权限
- size=65536k：大小限制

### /etc/machine-id 的生成逻辑

systemd 使用 machine-id 来：
- 标识机器（用于日志、网络管理等）
- 生成其他 ID（如 journal 的 ID）
- 确保某些服务的唯一性

**生成策略的权衡**：

| 方案 | 优点 | 缺点 |
|------|------|------|
| 从宿主机复制 | 简单 | 所有容器 ID 相同，会导致冲突 |
| /dev/urandom | 真随机 | 需要熵池，容器启动可能阻塞 |
| 时间戳+ID | 唯一且确定 | 不是真随机，但足够 |

选择：时间戳 + container_id → 伪随机 32 字符十六进制字符串

### /dev/console 的必要性

systemd 需要 /dev/console 用于：
- 输出日志（如果没有 syslog）
- getty 登录
- 紧急模式下的交互

**创建逻辑**：
- 检查是否存在
- 如果不存在，创建字符设备（major 5, minor 1）
- 设置正确的权限

---

## 架构设计的决策链

### 为什么选择 init shim 而不是直接让 systemd 当 PID 1？

**方案 1：直接让 systemd 当 PID 1**
- 优点：简单，systemd 直接控制一切
- 缺点：无法在 systemd 启动前做准备，无法在 systemd 退出后做清理，可能导致宿主机崩溃

**方案 2：init shim 作为 PID 1**
- 优点：可以在 systemd 启动前准备环境，可以在 systemd 退出后做清理，可以处理 systemd 不处理的信号，防止宿主机崩溃
- 缺点：需要实现 init 逻辑

**决策**：选择方案 2，因为安全性和灵活性更重要。

### 为什么选择 signalfd + 传统信号回退？

**方案 1：只使用传统信号处理**
- 优点：兼容性好
- 缺点：有竞态条件，代码复杂

**方案 2：只使用 signalfd**
- 优点：无竞态条件，代码清晰
- 缺点：不支持旧系统

**方案 3：signalfd 为主，传统信号为回退**
- 优点：既有 signalfd 的优点，又有兼容性
- 缺点：需要维护两套代码

**决策**：选择方案 3，先尝试 signalfd，失败则使用传统信号。

### 为什么 cgroup 操作失败可以继续？

**方案 1：失败就报错**
- 优点：严格，问题暴露早
- 缺点：容器可能无法启动

**方案 2：忽略错误，继续运行**
- 优点：容器可以启动，只是功能受限
- 缺点：某些功能不可用

**决策**：选择方案 2，因为 cgroup 是优化不是必需，且某些环境（如没有 CAP_SYS_ADMIN）确实无法配置 cgroup。

---

## 实现中的关键技术选择

### 信号处理的代码组织

将信号列表抽象为数组：
```
static const int forward_signals[] = {
    SIGHUP, SIGINT, SIGQUIT, SIGTERM,
    SIGUSR1, SIGUSR2, SIGPWR, SIGWINCH, SIGCHLD, 0
};
```

优点：
- 统一处理所有信号，避免重复代码
- 便于维护，添加新信号只需修改数组
- 数组以 0 结尾，便于遍历

### 进程收割的循环设计

```
while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
    // 处理子进程
}
```

关键点：
- WNOHANG：非阻塞，立即返回
- 循环：一次信号可能对应多个子进程退出
- 记录：如果是 systemd，记录退出状态

### cgroup 的层次构建

先启用 root cgroup 的控制器，再创建子组：

1. echo +cpu > /sys/fs/cgroup/cgroup.subtree_control
2. mkdir /sys/fs/cgroup/systemd
3. echo +cpu > /sys/fs/cgroup/systemd/cgroup.subtree_control

原因：控制器必须在父级启用，子级才能使用。

### 退出码的计算

```
if (WIFEXITED(status)) {
    exit_code = WEXITSTATUS(status);
} else if (WIFSIGNALED(status)) {
    exit_code = 128 + WTERMSIG(status);
}
```

遵循 POSIX shell 的约定，让调用者能够区分正常退出和信号终止。

---

## 验证与反思

### 测试策略的思考

**应该验证什么？**

1. **功能验证**：systemd 能启动吗？
2. **环境验证**：/run、/tmp 是 tmpfs 吗？
3. **PID 验证**：容器内的第一个进程是 PID 1 吗？
4. **cgroup 验证**：控制器启用了吗？
5. **信号验证**：systemd 能正确接收信号吗？
6. **退出验证**：退出码正确传播了吗？

**测试方法**：
- 创建最小容器环境（busybox 或类似）
- 运行 ruri，检查输出
- 使用自动化脚本验证

### 技术债务与改进空间

**当前实现的局限**：

1. cgroup delegation 不够完善：只是基本启用，没有处理更复杂的场景
2. 错误处理可以更详细：某些错误只是警告，没有详细的诊断信息
3. 性能：信号处理循环可以优化
4. 文档：用户文档不够详细

**可能的改进方向**：

1. 支持更多的 cgroup 控制器（blkio、net_cls 等）
2. 添加诊断模式，输出详细的配置信息
3. 优化信号处理，减少系统调用
4. 更完善的文档和示例

### 核心洞察的回顾

**最重要的三个洞察**：

1. **PID 命名空间是核心**：没有 PID 命名空间，systemd 不可能成为 PID 1
2. **init shim 是必须的**：需要一个中介来处理 systemd 不能处理的事情，并保护宿主机安全
3. **环境准备是关键**：/run、/tmp、machine-id 等必须正确设置

**设计原则的坚持**：

- **轻量级**：不引入重量级依赖，保持代码简洁
- **向后兼容**：DISABLE_SYSTEMD 条件编译，不影响现有功能
- **跨平台**：支持 Android/Bionic，提供回退机制

---

## 结语

添加 systemd 支持不是关于代码的堆砌，而是关于理解容器和 init 系统的本质。

**核心思想**：
- 容器是一个"看起来像系统"的环境
- 我们可以用 namespace + 挂载来创建这个环境
- systemd 不关心它是不是真的在管理物理机，它只需要正确的接口
- init shim 是必须的，它保护了宿主机，也提供了灵活性

**最终实现的价值**：
- 让容器能够运行需要 systemd 的应用
- 保持轻量级和向后兼容
- 为其他容器运行时提供参考
- 确保宿主机的安全（不会因为容器内的 systemd 而崩溃）

**最后的思考**：
技术实现只是表象，理解背后的原理才是根本。希望这篇文章能帮助你理解为什么要这样做，而不仅仅是复制代码。

---

*技术不是目的，解决问题才是。*
*代码不是重点，思考过程才是。*
